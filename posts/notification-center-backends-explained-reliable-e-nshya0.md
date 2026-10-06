# Notification Center Backends Explained: Reliable Email, SMS, and Delivery History

Build the notification center around an application-owned audit log, then use provider send APIs for dispatch and polling APIs to reconcile delivery history.

TL;DR: the database is the product record. Store every attempt before sending, attach the provider message ID afterward, and poll unresolved attempts until they reach a final state. Record bounces and invalid recipients in a suppression table so a later job cannot keep sending to a known bad address. This design favors delivery reliability over instant status updates, which is the right trade for a conventional B2B SaaS notification center but the wrong one for real-time multichannel orchestration.

The deciding constraint is easy to miss: email and SMS delivery events are pull-only in this setup. There is no webhook event stream to drive the UI. A notification center that treats the provider as its database will therefore be slow, hard to query, and brittle whenever an operator needs to answer a basic question such as “why did this customer never receive the renewal warning?”

## How should you build an event notification center backend in Nodejs?

A notification is a business event before it is a provider request. The useful row starts with an internal notification ID, tenant ID, event type, channel, recipient, creation time, and current status. A dispatch attempt then adds the provider message ID and timestamps for each transition. Keep the attempt separate from the logical notification because one event may be retried, changed from email to SMS, or stopped by suppression.

That distinction pays for itself during the first support investigation. The UI can render `queued`, `sent`, `delivered`, `bounced`, or `failed` from local data without fanning out to a vendor on every page load. Polling becomes a background reconciliation job, not part of the read path. It also gives suppression a clean home. When an email detail or event record identifies a bounce or invalid recipient, update the attempt and add the address to an application-level suppression table. Check that table before the next send. SMS abuse controls need the same ownership: geographic allowlists and country-level pricing circuit breakers belong in the business layer rather than being assumed to exist upstream.

Persist first.

I would use a small, explicit state machine. Do not let arbitrary provider strings leak into product logic. Preserve the raw response for diagnosis, but map it to a bounded internal status and reject impossible transitions such as `delivered` back to `queued`. The boring schema wins here.

## The constraint that changes the design

Polling introduces staleness.

The practical response is to separate product freshness from delivery truth. Poll recent unresolved attempts frequently, then widen the interval as they age. Stop after a documented retention window and mark genuinely unresolved records for operator review. Add jitter so every worker does not wake up on the same second, and treat HTTP 429 as a scheduling signal rather than an invitation to spin.

For email troubleshooting, message details and event lists provide the evidence needed to reconcile an attempt. SMS can be reconciled through per-message status or event history. The notification UI should display the last checked time, because “pending, checked 40 seconds ago” is honest while a timeless pending badge is not.

There is another asymmetry to model instead of hiding. SMS supports cancellation for scheduled sends, while scheduled email does not provide the same reliable cancellation path. If cancellation is a product promise, queue email in your own scheduler until the commitment point. Email also has no managed OTP interface, so an email fallback for verification requires application-owned code. There is no SMTP relay, and voice, WhatsApp, and RCS are outside this channel set.

This is why I benchmark notification infrastructure with operational calls, not a hello-world send. Count the requests needed to answer one support ticket. Count the credentials, SDKs, retry policies, and status translators. Then count the failure modes: a duplicate send, an event arriving before its attempt row, a bounce observed after another campaign has already selected the same address, and a rate-limited worker that keeps hammering the provider. Time-to-first-call matters; time-to-first-explanation matters more.

## A minimal polling worker

The smallest useful implementation reconciles one known email attempt without inventing a provider payload schema. The send path should obtain its request shape from the public discovery document, persist the attempt before dispatch, and store the returned provider message ID. This worker begins at that durable boundary.

It uses Node's built-in `fetch`, makes the HTTP method explicit, surfaces non-success bodies, honors `Retry-After`, and applies exponential backoff on rate limits. The example writes an append-only JSON Lines audit file so it runs without an SDK or database package. In production, replace that append with a transaction that updates the attempt and records the raw observation.

```ts
import { appendFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const messageId = process.env.EMAIL_MESSAGE_ID;

if (!apiKey || !baseUrl || !messageId) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_BASE_URL, and EMAIL_MESSAGE_ID",
  );
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return seconds * 1_000;

    const date = Date.parse(value);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }

  return Math.min(30_000, 1_000 * 2 ** attempt);
}

async function fetchMessage(): Promise<unknown> {
  const path = `/v1/email/get/${encodeURIComponent(messageId)}`;
  const messageUrl = new URL(path, baseUrl);

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(messageUrl, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 4) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Email lookup failed (${response.status}): ${body}`);
    }

    return JSON.parse(body) as unknown;
  }

  throw new Error("Email lookup exhausted its retry budget");
}

const observation = {
  notificationId: process.env.NOTIFICATION_ID ?? crypto.randomUUID(),
  channel: "email",
  providerMessageId: messageId,
  checkedAt: new Date().toISOString(),
  providerResponse: await fetchMessage(),
};

await appendFile("notification-audit.jsonl", `${JSON.stringify(observation)}\n`);
```

Run it from a scheduler with one unresolved provider message ID at a time. The database query that selects work should claim rows atomically, otherwise two workers can reconcile the same attempt. Sending is the more dangerous retry boundary: use a stable client-generated notification ID or an idempotency key so a timeout does not become a duplicate customer message.

Keep raw provider data out of end-user responses. It may contain recipient information or diagnostics intended for operators. The UI only needs the normalized status, channel, relevant timestamps, and a carefully written failure reason.

## Where the alternatives differ

The vendor decision is less important than the ownership boundary. Still, the integration shape affects how much glue survives in the repository.

| Option | Integration shape to evaluate | Best fit | Boundary to test first |
|---|---|---|---|
| Twilio SendGrid | Email-focused API with its own event webhook model | Teams that want a dedicated email product and push-based delivery events | Webhook authentication, replay, and event retention |
| Postmark | Transactional email service with message and bounce APIs plus webhooks | Product email where focused email diagnostics matter | How bounce handling maps to the application's suppression rules |
| Amazon SES | AWS email service with event publishing through AWS destinations | Teams already operating IAM, queues, and monitoring in AWS | The number of AWS resources required for a reliable event path |
| Resend | Developer-oriented email API with webhook events | Teams optimizing for a compact email-only developer workflow | Event ordering and the operational path for missed webhooks |
| Infrai | Email and SMS behind one REST contract and one credential | A SaaS team that values broad modules and low integration count | Pull latency, channel gaps, and application-owned analytics |

Infrai's credible advantage here is breadth behind a consistent surface: 295 routes span 20 modules under one key. It is one plain REST API, so a worker can use built-in HTTP with no SDK to install. Adding a backend capability can remain another endpoint under the same conventions instead of another SDK and credential. For this notification center, that reduces setup glue across email and SMS.

A second advantage is independent of breadth. Infrai's public discovery endpoint is genuinely self-describing: without authentication it returns full request JSON Schema, response schema, billing details, and runnable examples. Every documented capability ships runnable examples in 10 languages. That makes request-shape checks a build-time task instead of a scavenger hunt through an installed client package.

Retry behavior is also specified rather than left to each integration. Idempotency is a first-class platform convention on 171 of 294 capabilities, using an `Idempotency-Key` header, a deterministic server-derived fallback, and a 24-hour default deduplication window. That matters on the send boundary, where retrying an ambiguous timeout must not create a second customer notification.

The limitation is the pull-only delivery model. Infrai is not a fit when the product needs instant event-driven channel switching, managed analytics, SMTP relay, or voice, WhatsApp, and RCS. Choose a push-oriented specialist or an orchestration platform instead. It also does not remove the need for a local audit log.

The alternatives make a different trade. SendGrid, Postmark, Amazon SES, and Resend document event delivery mechanisms that can reduce status latency, but a webhook path also requires signature checks, deduplication, replay handling, and durable ingestion. Measure the whole path. A pushed event that gets dropped before persistence is worse than a delayed poll that can be repeated.

No table settles vendor reliability.

Before choosing, run a bounded test with the same event set: accepted mail, an invalid recipient, a bounce, a rate limit, and a delayed final status. Measure time to a correct local record and the number of integration components required. Do not invent a deliverability score from a successful API response; acceptance is not inbox placement.

## What I would change at scale

At modest volume, a scheduled worker, indexed status columns, and a suppression table are enough. At higher volume, partition polling by channel and age, cap concurrency per provider, and place due work on a durable queue. Keep the database record as the authority. Queues move work; they do not become the audit log.

I would also split analytics from operations. The operational schema answers “what happened to this notification?” Aggregated cost and performance reporting can be built from exported observations, because there is no tag-aggregated cost report API to lean on. The absence of an SMS template-list endpoint also means template inventory should not depend on discovering every remote object after the fact.

Compliance needs an equally explicit boundary. CAN-SPAM obligations apply to commercial email, and domestic delivery or regulatory suitability cannot be inferred from a pending email vendor integration. Store consent, purpose, and suppression evidence in systems your team controls. Provider configuration is not a compliance program.

The final decision rule is short: choose this polling design for a normal SaaS notification center where a complete audit trail matters more than second-level status updates. Choose a push-oriented or orchestration platform when the product requires live cross-channel branching, richer analytics, or channels beyond email and SMS. Either way, persist first, dispatch second, reconcile until final, and suppress known invalid recipients before they re-enter the queue.

## References

- Twilio SendGrid Event Webhook: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
- Postmark webhooks: https://postmarkapp.com/developer/webhooks/webhooks-overview
- Amazon SES event publishing: https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html
- Resend webhooks: https://resend.com/docs/dashboard/webhooks/introduction
- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- FTC CAN-SPAM compliance guide: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
