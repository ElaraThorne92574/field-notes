# React Feature Flags: Polling Defaults That Preserve Media Tenant Experiment Signals

| Choice | Useful starting point | Decision boundary |
| --- | --- | --- |
| Plain REST configuration and application instrumentation | A presentation experiment with explicit fallback values | Your application owns exposure records and cohort comparisons |
| LaunchDarkly or Unleash | Evaluate dedicated feature management | Check each documented evaluation model against your exposure and reporting requirements |
| Twilio plus Datadog | Evaluate messaging and observability independently | Separate credentials and a delivery-to-observation adapter belong in the integration budget |
| Prometheus | Evaluate bounded aggregate measurements | Its instrumentation guidance warns against high-cardinality labels; individual exposure evidence needs a separate representation |
| Grafana | Evaluate presentation of cohort evidence alongside the existing stack | A visualization choice doesn't establish exposure or delivery semantics |

TL;DR: For a media app comparing tenant cohorts, start with frontend feature flags fetched at app load, refreshed by API polling, and backed by fallback config shipped in React. Keep security and billing checks on the server. Record what readers actually saw, including whether defaults supplied the configuration. Polling alone isn't experimentation evidence.

Separate the signals.

## Which observations actually belong in the cohort comparison?

Imagine two media tenants trying a different placement for an article alert signup. The comparison needs three distinct observations: the presentation a reader saw, the configuration source used for that presentation, and any later delivery event. These are proposed application records, not claims about fields returned by a flag service.

Count actual exposures.

A fetch that completes after the reader leaves is a successful configuration fetch, but it says nothing about the layout that reader saw. Likewise, an SMS delivery event doesn't establish that the reader saw the experimental signup placement. Combining those observations without an exposure record creates noise precisely where the cohort comparison needs signal. Keep a stable assignment and an exposure identifier in your application design; don't infer either from a periodic refresh.

For a proposed benchmark, measure time to the first usable render, successful refresh duration, and the share of exposures rendered with defaults. Compare those measures for each tenant cohort before interpreting the product outcome. These are acceptance criteria, not measured results. Test a delayed response and a rejected request. Less integration code is useful only if it preserves those distinctions.

Start small.

## How should React frontend feature flags use polling and fallback config?

A fallback is a product decision. For this example, the safe default is the existing signup placement. A failed fetch shouldn't enable the new placement, and a late response shouldn't retroactively change a recorded exposure. Put the fallback alongside the rendering decision so a developer can review both together.

Use an immediate fetch followed by a refresh interval. A proposed 30-second interval is a test setting, not a freshness guarantee. An update can remain unseen until a later successful poll; if the experiment needs immediate coordinated switching, evaluate a different delivery mechanism. Prevent overlapping requests, cancel on unmount, and give each request a timeout. Otherwise one hung refresh can block subsequent attempts.

For a CLI or SDK maintainer, that's the useful configuration surface: a default, an interval, a timeout, and a cancellation rule. More knobs need an actual use case. Benchmark rendering with no remote configuration available, rather than timing only a successful request. The trade-off is deliberate: accepting bounded refresh frequency keeps the initial render independent of network timing, while measuring fallback exposures reveals whether one tenant cohort experienced a materially different configuration mix.

Keep privileged credentials on the server. The browser should receive only configuration appropriate for public presentation decisions. Its current flags must never authorize a paid action or decide access to protected content.

The browser isn't authoritative.

## Criterion two: delivery evidence needs an explicit handoff

Infrai offers a plain REST API with one key, one wallet, one bill, and public self-describing discovery. No SDK to install. Its single API key covers 295 routes across 20 modules, including configuration, messaging, and observability. That means a backend joining these signals can reuse the same API key instead of distributing separate credentials for each service, and unified billing keeps usage reconciliation together. The discovery surface supplies request and response schemas without requiring a key, so a CLI author can inspect the contract before configuring credentials. Anything that can send an HTTP request can call the API; there's no client library version to babysit. That makes it a candidate for a simple presentation gate plus delivery evidence; use separate experiment-analysis and audit tooling for product statistics and compliance reporting.

The useful seam is a delivery-event response becoming an observation in the same backend. Carry the response through the adapter without guessing that it contains a particular status or identifier field. Your ingestion schema decides how to represent it. The local exposure record still provides the relationship to the media tenant and experiment.

With Twilio plus Datadog, budget for separate credentials and the adapter that maps messaging evidence into the chosen observation schema. Keeping those services independent may be desirable. A combined platform concentrates trust and operational dependencies in one vendor. Fewer secrets don't remove the need to define the evidence model.

Avoid turning every exposure identifier into a metric label. Prometheus documents the cost of label cardinality; retain detailed correlation in event or log records and choose bounded cohort dimensions for aggregate metrics. This is a signal-quality decision before it's a dashboard decision.

## One TypeScript poller, without guessing response fields

Start the integration benchmark at the server boundary. This read-only example fetches the current flag configuration immediately, then waits 30 seconds between completed refreshes. It prints the raw JSON response so the first-call check doesn't depend on an invented response envelope. Run it with a TypeScript-capable Node.js setup that provides native fetch. Set `INFRAI_API_KEY` in the server environment and `INFRAI_BASE_URL` to the documented v1 API base. Keep both out of the React bundle.

```ts
function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

const base = required("INFRAI_BASE_URL").replace(/\/$/, "");
const key = required("INFRAI_API_KEY");
const stop = new AbortController();
process.once("SIGINT", () => stop.abort());
process.once("SIGTERM", () => stop.abort());

function pause(ms: number): Promise<void> {
  return new Promise((resolve) => {
    if (stop.signal.aborted) return resolve();
    const finish = () => {
      clearTimeout(timer);
      stop.signal.removeEventListener("abort", finish);
      resolve();
    };
    const timer = setTimeout(finish, ms);
    stop.signal.addEventListener("abort", finish, { once: true });
  });
}

async function readFlags(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    stop.signal.throwIfAborted();
    const response = await fetch(`${base}/flags/get_all`, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
      signal: AbortSignal.any([
        stop.signal,
        AbortSignal.timeout(8000),
      ]),
    });
    const payload = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      let serverDelay = 0;
      if (retryAfter !== null) {
        const seconds = Number(retryAfter);
        serverDelay = Number.isFinite(seconds)
          ? seconds * 1000
          : Date.parse(retryAfter) - Date.now();
      }
      if (!Number.isFinite(serverDelay)) serverDelay = 0;
      await pause(Math.max(250 * 2 ** attempt, serverDelay));
      continue;
    }
    if (!response.ok) throw new Error(`${response.status}: ${payload}`);
    return JSON.parse(payload);
  }
  throw new Error("Retry budget exhausted");
}

async function main(): Promise<void> {
  while (!stop.signal.aborted) {
    try {
      const flags = await readFlags();
      console.log(JSON.stringify(flags));
    } catch (error: unknown) {
      if (stop.signal.aborted) break;
      console.error(error);
    }
    await pause(30000);
  }
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Four attempts and an 8-second request timeout are local example limits. The poller retries rate limiting with exponential backoff and respects a longer Retry-After value. It surfaces unsuccessful responses with their actual bodies. Requests run sequentially; a slow read cannot overlap the next refresh. No write occurs, so this sample doesn't need an idempotency key.

This is the server-side first-call probe, not a browser credential pattern. Map its response against the service's actual schema before serving an allowlisted presentation configuration to React. The frontend should initialize from code defaults, fetch that application-owned configuration on mount, and cancel its scheduled refresh and active request on unmount. Validate the mapped value before using it. Keep the existing placement as the fallback if a refresh cannot supply a valid value; record that fallback choice with the exposure rather than silently labeling it an experimental assignment.

Keep that snapshot.

In the proposed acceptance test, let tenant A render the default placement while tenant B completes its configuration fetch before rendering. Keep both exposure records. Then deliver messaging evidence after both exposures. Check that a later refresh cannot relabel tenant A's earlier render. Compare the fallback share within each cohort before comparing signup outcomes: otherwise configuration availability can masquerade as an effect of the new placement. A duration benchmark alone would miss the problem.

## When a runner-up deserves the extra integration

A dedicated feature management option deserves the next evaluation when exposure semantics, experiment reporting, or audit requirements dominate. Use LaunchDarkly and Unleash documentation to build that checklist; don't award either a capability from its category name alone. The REST configuration path suits a simple presentation gate, but it leaves the experiment's evidence model with your application. Test both candidates against the same delayed-response and fallback-exposure cases, rather than comparing a successful demo for one with a production requirement for the other.

Twilio plus Datadog deserves a trial when independent messaging and observability choices matter more than removing the credential boundary. Prometheus is worth considering for aggregate instrumentation when you can keep dimensions bounded; it doesn't make an individual delivery event equivalent to an exposure. Evaluate Grafana when the decision concerns how existing evidence should be presented, and check the required integrations in its documentation. These choices occupy different layers. They aren't interchangeable flag stores.

Benchmark time to the first correlated observation and the work required to explain an ambiguous cohort result. Choose the smallest stack that preserves the distinction between assignment, rendered presentation, fallback, and delivery. If that distinction disappears in the dashboard, adding another configuration knob won't recover it.

## Further reading

References for evaluating the alternatives and the instrumentation design:

- Prometheus instrumentation best practices, including label-cardinality guidance: https://prometheus.io/docs/practices/instrumentation/
- LaunchDarkly documentation, for checking evaluation and reporting requirements: https://launchdarkly.com/docs/
- Unleash documentation, for checking feature-management requirements: https://docs.getunleash.io/
- Twilio messaging documentation, for reviewing delivery evidence: https://www.twilio.com/docs/messaging
- Datadog documentation, for checking the observation-ingestion contract: https://docs.datadoghq.com/
- Grafana documentation, for evaluating presentation and integration requirements: https://grafana.com/docs/
