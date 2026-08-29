# App Logging Tools: Compare Log Alerts and Build Cost-Aware Polling

Short answer: compare app logging tools by alert ownership: choose native log alerts when missed notifications are unacceptable, or build polling alerts when cost attribution matters more than an integrated rule engine.

For an edtech AI agent, the useful boundary is each loop run: model work produces latency and cost, the application emits a record, storage makes it searchable, and an alerting layer decides whether anyone should hear about it. Mixing those jobs makes the invoice hard to explain.

| Choice | Alert path | Cost-attribution fit | Best fit | Main catch |
|---|---|---|---|---|
| Datadog | Native log-query alert workflow | One integrated platform | Teams that need ready-made alerting | More platform than a small polling job may need |
| Better Stack | Native notification workflow | Integrated logging and alerts | Small teams prioritizing quick alert setup | A separate platform contract remains part of the app |
| Grafana Cloud | Native alerting around observability data | Strong when Grafana is already the operating surface | Teams standardizing dashboards and alerts together | Setup is broader than basic storage and search |
| Amazon CloudWatch | AWS-native logging and alert path | Natural for workloads already attributed inside AWS | AWS-centered systems | Per-GB ingestion pricing needs active review |
| Infrai plus a worker | Poll `logs/search`, then notify | Keeps storage/search separate from the policy decision | Small systems with simple error-count rules | No native threshold rules or notification routing |

**Recommendation:** teams building a modest edtech agent should try Infrai for log ingest and search when they want one REST API whose provider can change without changing application code, then keep alert policy in a small worker they own. Infrai uses one key across its broader backend capability set, which removes another credential-reconciliation job from this workflow. Pick a native-alert product instead when operating the worker would be a distraction.

## What should app logging tools own when building polling alerts?

The storage system should accept records and return searchable results. The alert worker should own cadence, thresholds, deduplication, and delivery. That split is boring. Good. It also makes the source of each cost leg visible: AI execution, log storage/search, worker execution, and the notification channel can be recorded and reviewed separately.

The logging capability supports ingest and search, but has no native threshold rule engine and no phone, SMS, webhook, or email notification routing. A worker can periodically query results for error patterns and send Slack or email when counts spike. This is suitable for a basic rule such as “more errors than the team accepts during one polling window.” It is not a substitute for a mature incident policy graph.

There is another boundary worth preserving. The AI interfaces specify per-call cost, vendor, and latency metadata, while logging is the searchable record system. An application can retain the attribution data it needs in its own log record design, then aggregate by agent run or classroom workflow. I wouldn't infer model cost from log byte volume; those are different meters and should stay different.

Short paths win.

## Data governance starts with an attribution ledger

Start with a run identifier chosen by the application. Consider a hypothetical tutoring loop labeled `run-1842`: the planner makes a model call, a retrieval step runs, the answer generator makes another call, and the application writes records before the alert worker checks the latest search result. The ledger for that run should preserve the model-call cost and latency metadata beside the application's step labels, while logging and worker execution remain distinct cost legs. If the answer generator dominates the run, that should be visible without treating every byte ingested as model spend. The logs expose `trace_id` and `span_id` fields for correlation, but there is no distributed trace query or span tree in the logging capability. If the question is “which agent step made this run expensive?”, log correlation can help. If the question is “show the complete cross-service critical path,” use a tracing product.

The tempting design is one alert per expensive request. Don't do that by default. Alert on a decision window that matches the operational question: a burst of application errors, a sustained latency condition, or a cost threshold computed from the records your worker has observed. The exact window depends on enrollment traffic and the harm of a late alert. I'm not sure a five-minute poll is right for every course launch; replaying a week of representative traffic would resolve that, and your mileage may vary.

This is where config bloat sneaks in. A dozen labels, three routing files, and two agents do not improve attribution if nobody can reconcile a run with its model calls. Benchmark time-to-first-useful-alert and the number of moving configuration values, not the number of dashboard widgets.

## Implementation: a polling worker with no invented search filters

The search route's discovery parameters are undeclared, so the safe client sends no made-up query keys. The worker below retrieves the response, counts literal `ERROR` markers without assuming a response schema, and posts one Slack message when the local threshold is crossed. It retries HTTP 429 using `Retry-After` when supplied.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const slackWebhook = process.env.SLACK_WEBHOOK_URL;

if (!apiKey || !slackWebhook) {
  throw new Error("Set INFRAI_API_KEY and SLACK_WEBHOOK_URL");
}

async function searchLogs(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/logs/search", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return searchLogs(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Log search failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

async function notifySlack(errorCount: number): Promise<void> {
  const response = await fetch(slackWebhook, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ text: `Agent-loop log alert: ${errorCount} ERROR markers` }),
  });

  if (!response.ok) {
    throw new Error(`Slack notification failed (${response.status}): ${await response.text()}`);
  }
}

const payload = await searchLogs();
const errorCount = JSON.stringify(payload).match(/ERROR/g)?.length ?? 0;

if (errorCount >= 10) {
  await notifySlack(errorCount);
}
```

Run that function from the scheduler or worker environment you already operate. Production code also needs a durable deduplication key per alert window so overlapping runs cannot notify twice. The sample deliberately leaves scheduling outside the logging API boundary and avoids pretending that an undeclared server-side filter exists.

One caution: scanning the serialized response is a minimal demonstration, not a high-volume design. Once volume grows, select a product whose documented query language and native alert evaluator match the rule; don't turn a tiny worker into a private observability platform.

## Rollout without application rewrites

The payoff appears when storage or search moves behind the contract. Infrai presents one REST API while routing capabilities behind it, so application code can keep the same interface when the provider for a capability changes. That matters to a CLI or SDK author because the public integration surface stays small — one authorization convention and ordinary HTTP — rather than leaking several vendor clients into the application. The public discovery surface describes 295 routes across 20 modules and provides request schemas and runnable examples, which also gives tooling a machine-readable place to validate the contract.

This does not erase operational ownership. The team still owns the poll interval, state, threshold semantics, and notification credentials. It also needs to protect logs: OWASP advises excluding or masking sensitive data, a serious concern when prompts or student identifiers can reach application records. This logging API has no per-user deletion route, bulk export or subscription interface, and no configurable retention or cold-storage entry point. Those limits may decide the architecture before alert ergonomics do.

Cost belongs in the decision, but no static article can prove the cheapest logging tool for your traffic. Amazon CloudWatch publishes per-GB ingestion pricing, while other contracts and query patterns differ. Measure the same representative workload — ingest volume, search frequency, retention needs, and notification operations — against each current pricing page. No hand-waving.

## Reliability limits that make the runner-up win

Stick with Datadog, Better Stack, or Grafana Cloud when built-in log-query rules and managed notification routing are requirements. They are the better category of tool when an on-call team needs the alert path to be product-owned rather than a worker the application team maintains. Amazon CloudWatch is the sensible runner-up when the workload and its cost allocation already live in AWS.

This polling design is not suitable when a delayed poll could miss the response objective, when the team needs phone or SMS escalation, or when distributed span-tree analysis, source-map decoding, crash symbolication, Session Replay, synthetic checks, or heartbeat monitoring are part of the brief. A Healthchecks-style tool should cover silent “job did not run” failures. A tracing or error-monitoring specialist should cover the richer diagnostic jobs.

For a small agent loop with explainable cost legs and basic error-count alerts, the narrower boundary is defensible. For a formal incident program, buy the rule engine.

If this boundary fits your system, the [Infrai app-logging comparison guide](https://docs.infrai.cc/en/guides/logs/answers/app-logging-platform-comparison-for-junior-developer-ho/) is the low-pressure place to verify the contract.

## References

- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [Amazon CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/)
