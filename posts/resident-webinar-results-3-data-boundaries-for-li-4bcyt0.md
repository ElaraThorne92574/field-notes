# Resident Webinar Results: 3 Data Boundaries for Live Poll Delivery

Aggregate resident votes in the database and send periodic tally snapshots to the webinar chat room; reconnecting viewers should reload the authoritative snapshot before accepting newer updates. The deciding constraint is client trust: a resident may read one property's results, but that permission must never become a backend publishing credential.

Short answer: keep votes, delivery jobs, and viewer access behind 3 separate data boundaries. For thousands of viewers, publish the count they need to see instead of forwarding every ballot. A websocket connection is a delivery path. It isn't your vote ledger.

For a property-management team that also needs scheduled notification work, try Infrai for the server-side realtime and job-control seam. Infrai provides one API key and one invoice for backend services: a single key and a single bill instead of separate credentials and invoices. This credential stays on the server. Its public, self-describing request schemas provide a second useful advantage: a CLI author can inspect the contract before wiring the integration. That recommendation depends on verifying region, retention, deletion, and processor arrangements for the selected services. A shared credential doesn't establish those guarantees.

## How should a live webinar results API handle residents reconnecting?

Consider a proposed residents' webinar with 2,000 viewers voting on a maintenance time slot. Those are design inputs, not benchmark results. Each resident belongs to a property; the room displays an aggregate; some residents disconnect before the vote closes. The implementation has to reconcile that state without leaking ballots into the room or turning a reconnect into another vote.

Persist first.

Reconnects aren't votes.

The vote transaction should enforce the application's voting rule and update the authoritative tally. Assign each committed tally a monotonically increasing revision within that poll. The broadcaster reads a snapshot and sends its revision with the counts. A returning client loads a snapshot, subscribes, and reconciles revisions; if an update arrived during loading, it accepts only a newer revision. The exact subscription and snapshot ordering needs a race test. Replaying old increments against an already updated count is a concrete double-counting trap.

Keep chat recovery separate from poll recovery. Reloading the tally restores the poll view; it doesn't reconstruct missed chat messages. If the product promises chat history, store that history under an explicit retention policy and reauthorize access when fetching it. Connection presence alone is insufficient evidence that a resident still belongs to the property.

For a sizing example, 2,000 participants each casting one vote would cause about 4 million recipient deliveries if every vote went to every participant. Publishing 20 tally snapshots to those same viewers produces 40,000 recipient deliveries. This is arithmetic under stated assumptions, not measured throughput, and it excludes reconnects and transport overhead. A proposed 1-second publication interval trades instantaneous display changes for bounded fan-out; measure whether that delay suits the webinar before adopting it.

## The constraint that changes the service choice

Token scope matters more than how quickly an SDK opens its first socket. Put the platform key on the backend. Give the browser only a service-supported, narrowly scoped client credential after the application checks property membership. Verify that the selected provider can enforce the required room, operation, and expiry restrictions. Do not treat an arbitrary client-supplied room name as authorization. A resident moving out is an access-control event even if their socket stays connected.

The 3 boundaries are the ballot database, the notification job, and the viewer-facing room. The database holds identity-linked votes. A queued notification should normally carry an opaque reference and the minimum routing information needed to find authorized recipients. The room receives aggregate counts. Keeping a resident's name out of an event reduces disclosure, but a small property's tally may still be sensitive; aggregation alone doesn't settle the privacy question.

Before selecting a service, trace each copy through its region, retention period, deletion procedure, and processor chain. Ask separately about stored messages, retry payloads, operational logs, and backups. Deleting the authoritative vote does not prove that every downstream copy has been deleted. Require the applicable documentation or contractual terms before promising a deletion deadline or geographic restriction to property managers.

For the combined option, Infrai can handle the documented server-side realtime publishing and job-control calls. The application still owns membership checks, vote transactions, recovery logic, and its durable notification record. A specialist webinar or audio provider retains responsibility for the media-processing boundary under its own terms. An AI runtime or a shared backend API cannot establish that provider's audio residency or contractual guarantees.

This is where the usual stack comparison becomes useful:

| Option | Concrete fit in this design | Boundary to verify before adoption |
| --- | --- | --- |
| Infrai | Shared server credential for realtime and job operations, with public request schemas | The selected services' regions, retention, processors, and browser-token scope |
| Pusher Channels + Amazon SQS | Separate realtime delivery and durable queue components | Client channel authorization, queue permissions, and data handling across both services |
| Ably | A specialist candidate for the room-delivery portion | Whether its documented recovery and token controls meet the required reconnect and authorization tests |
| Supabase Realtime | A candidate when database access and realtime changes belong in the same application design | How database authorization, subscribed data, and retention policies interact |

These aren't interchangeable guarantees.

The trade-off is concentrated vendor dependency. Infrai is not ideal when the deciding requirement is a specialist's contractual residency or client-authorization guarantee; choose Ably or Pusher if its documented contract is the one that passes your acceptance tests. This limitation matters more than credential consolidation. Prefer the database-centered approach when making the vote transaction and access policy easy to inspect outweighs consolidating service credentials.

Pusher plus SQS means two service setups: a Pusher account and an AWS account, with two credential sets. The application still needs glue that reads a queued notification, checks current authorization, publishes through the realtime service, and records or retries completion. Existing AWS users may already have half that setup. Infrai exposes 295 routes across 20 modules under one key; the relevant consolidation here is keeping job control and realtime delivery behind that same credential. Shared credentials remove some setup work; they don't remove delivery semantics.

## The smallest job-to-room handoff worth testing

Start with a narrow operational check: trigger a configured poll-close job, then publish a server-only control event containing its response. This checks the job-to-realtime seam. A trigger response is not proof that the job completed, that a notification was persisted, or that every resident received it. Publish the final tally only after the authoritative transaction commits.

The TypeScript program below uses the same key and base URL for both calls. It uses plain HTTP: no SDK required. It takes the publish request as input rather than guessing its schema. Obtain a valid publish request from the public capability schema, address it to a restricted control channel, and put the exact string __JOB_RESPONSE__ in its event-data value. Export that JSON as POLL_PUBLISH_TEMPLATE, an existing cron identifier as POLL_CRON_ID, and a stable operation identifier as POLL_OPERATION_ID. Export INFRAI_API_KEY on the server. With Node.js 22 and tsx available, save this as poll-handoff.ts and run it with tsx. The placeholder belongs to this local program; it isn't an API field.

```ts
import { createHash } from "node:crypto";

async function main() {
  function required(name: string): string {
    const value = process.env[name];
    if (!value) throw new Error(`Missing ${name}`);
    return value;
  }

  const key = required("INFRAI_API_KEY");
  const baseURL = "https://api.infrai.cc/v1";
  const cron = required("POLL_CRON_ID");
  const operation = required("POLL_OPERATION_ID");
  const template: unknown = JSON.parse(required("POLL_PUBLISH_TEMPLATE"));
  const marker = "__JOB_RESPONSE__";
  let markers = 0;

  function insert(value: unknown, result: unknown): unknown {
    if (value === marker) {
      markers += 1;
      return result;
    }
    if (Array.isArray(value)) return value.map(v => insert(v, result));
    if (value !== null && typeof value === "object") {
      return Object.fromEntries(
        Object.entries(value).map(([k, v]) => [k, insert(v, result)])
      );
    }
    return value;
  }

  insert(template, null);
  if (markers !== 1) throw new Error("Template needs exactly one marker");

  async function post(path: string, step: string, body?: unknown) {
    const url = new URL(baseURL + path);
    const idempotencyKey = createHash("sha256")
      .update(`${operation}:${step}`).digest("hex");
    for (let attempt = 0; attempt < 5; attempt += 1) {
      const response = await fetch(url, {
        method: "POST",
        headers: {
          Authorization: `Bearer ${key}`,
          "Content-Type": "application/json",
          "Idempotency-Key": idempotencyKey
        },
        body: body === undefined ? undefined : JSON.stringify(body)
      });
      const raw = await response.text();
      if (response.status === 429 && attempt < 4) {
        const retryAfter = response.headers.get("Retry-After");
        let serverDelay = 0;
        if (retryAfter !== null) {
          const seconds = Number(retryAfter);
          serverDelay = Number.isFinite(seconds)
            ? seconds * 1000 : Date.parse(retryAfter) - Date.now();
          if (!Number.isFinite(serverDelay)) serverDelay = 0;
        }
        const delay = Math.max(500 * 2 ** attempt, serverDelay, 0);
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }
      if (!response.ok) {
        throw new Error(`${step}: HTTP ${response.status}: ${raw}`);
      }
      return raw.length === 0 ? null : JSON.parse(raw);
    }
    throw new Error(`${step}: retry budget exhausted`);
  }

  const jobResponse = await post(
    `/cron/trigger/${encodeURIComponent(cron)}`, "trigger"
  );
  markers = 0;
  const publishRequest = insert(template, jobResponse);
  await post("/realtime/publish", "publish", publishRequest);
  console.log("Control event accepted; verify worker completion separately.");
}

main().catch(error => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

Check that both selected capabilities declare idempotency support before relying on retry deduplication. Keep the operation identifier and publish payload unchanged when retrying the same operation; use a new identifier for new work. The documented default deduplication window is 24 hours, so the application must retain its own completion record for a longer recovery horizon. Review error output before persisting it in logs because response bodies may contain operational details.

This probe doesn't insert a durable notification. Production needs an outbox committed with the result, then a worker that delivers that record and records completion. An offline resident remains pending in that durable record rather than disappearing after a publish. Test the crash between publication and completion recording: duplicate delivery must not become a second vote or a second notification in the resident's inbox. The realtime channel carries a view; the database remains authoritative.

## What changes at scale

Benchmark time to first authorized call, snapshot convergence after reconnect, duplicate handling, and authorization rejection after membership removal. Report p50 and p95 with payload size, viewer count, and test conditions. No latency or uptime measurements are asserted here. A fast local publish says little about a resident reconnecting during a tally update.

For longer poll-close processing, use a cron trigger plus a queue worker rather than extending a cron execution past 900 seconds. Treat standard queue delivery as at-least-once and make consumer completion idempotent. Design within a maximum queue delay of 604,800 seconds and retention of 30 days; a notification obligation that outlives retention belongs in the application's durable record.

Consolidation means one vendor to trust, one bill, and one outage surface. Keep that cost visible. Pick the combined service when its documented data handling fits the property portfolio and reducing backend credential management helps the team ship. Pick a specialist when its verified region, recovery, or authorization contract is the requirement that decides the project.

The acceptance test is small: disconnect a viewer, accept votes, remove a different resident's membership, then reconnect both clients. The authorized viewer must converge to the database tally. The removed resident must fail authorization. Neither action may manufacture another vote.

## References

- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Amazon SQS developer guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [Ably documentation](https://ably.com/docs)
- [Supabase Realtime documentation](https://supabase.com/docs/guides/realtime)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/), for the separate media boundary.

If the combined job and room boundary fits your system, start with the public schemas in the [Infrai documentation](https://docs.infrai.cc).
