# Configure Automatic Error Grouping in Node.js Tracking for 3 Logistics Signals (and Why)

To configure automatic error grouping in a Node.js error tracking service, build a stable fingerprint from the failure class, import job, and normalized source location. Keep the raw message as evidence, not as the identity of the issue. For a logistics import that stops producing results, this gives operators one actionable group instead of a new alert for every shipment file.

TL;DR: group on a bounded set of structural signals, attach changing values as attributes, and alert only after a result-level threshold is missed. A stack trace alone is too noisy. A message alone is worse. The useful decision is signal quality versus noise, not which tracking dashboard has the most switches.

| Grouping choice | Signal quality | Noise risk | Good fit |
| --- | --- | --- | --- |
| Raw error message | Low | High | Temporary debugging only |
| Full stack trace | Medium | Medium to high | Small, stable code paths |
| Error class + normalized frame + import identity | High | Controlled | Scheduled logistics imports |
| Manual incident key only | Predictable | Misses new causes | A deliberately narrow alert |

The recommendation is the third row. Make the group key deterministic, cap its dimensions, and test it with representative failures before connecting it to paging.

That is the whole decision.

## How should Node.js tracking configure automatic error grouping?

Consider two failed imports:

```ts
type ImportFailure = {
  error: unknown;
  source: "warehouse-a" | "warehouse-b";
  jobName: string;
  schedule: string;
  fileName: string;
  receivedAt: string;
};

function errorClass(error: unknown): string {
  return error instanceof Error ? error.name : "UnknownError";
}

function stableFrame(error: unknown): string {
  if (!(error instanceof Error) || !error.stack) return "no-stack";

  const frame = error.stack
    .split("\n")
    .find((line) => line.includes("/src/imports/"));

  return frame
    ?.replace(/:\d+:\d+\)?$/, ")")
    .replace(/\\/g, "/")
    .trim() ?? "no-import-frame";
}

function groupKey(failure: ImportFailure): string {
  return [
    errorClass(failure.error),
    stableFrame(failure.error),
    failure.jobName,
    failure.source,
  ].join(":");
}
```

The file name, object ID, shipment number, and timestamp stay out of the key. Those values belong in event attributes. If `manifest-2026-10-28-001.csv` and `manifest-2026-10-28-002.csv` fail at the same parser frame for the same warehouse job, they are probably one issue. If warehouse A and warehouse B use different mappings, separating them is useful even when the JavaScript exception is identical.

There is a trade-off here. Adding `source` can split one code defect into several groups. Omitting it can merge two incompatible upstream contracts. I would keep a dimension only when an operator can take a different action because of it. That is a better rule than adding every field available in the event payload.

The key should be boring.

The normalizer above is intentionally modest. It removes line and column churn from a frame, but it does not pretend that arbitrary stack parsing is a standard. Production code should use a parser suited to the Node.js runtime and source-map policy in use, then lock the output down with tests. The group key is an internal contract. Treat it like one.

That contract deserves a changelog entry.

## Why does a quiet import need a separate signal?

An exception event proves that some code threw. It does not prove that the scheduled import produced a result. A queue can accept a job and never emit a completed batch. A parser can skip every record and return an empty success. A source can stop delivering files while the worker remains healthy.

That is why the exception group should sit beside a result-level watchdog. Track at least the expected schedule, last accepted input, last successful output, and count of imported records. Alert when the missing-result condition lasts beyond the allowed interval, then attach the latest grouped exception if one exists.

This two-signal model prevents a common mistake: paging on every retry. Retries are execution behavior. A missing import result is the business failure. The former helps explain the latter; it should not replace it.

OpenTelemetry describes logs as a signal with attributes and an observed timestamp, which maps cleanly to this split: stable event fields support grouping, while changing fields provide evidence around an event. Keep the event schema small enough that a CLI or SDK caller can produce it without a configuration ritual.

For example, an emitted event can carry `error.group`, `import.job`, `import.source`, `import.result_count`, and `import.last_success_at`. The exact transport is an implementation choice. The semantics are the part worth preserving across transports.

## How do I configure grouping without creating a second config language?

Put the policy beside the error boundary that knows the import context. Do not scatter grouping rules across scheduler settings, worker environment variables, and a dashboard form. That arrangement feels flexible until a renamed job silently changes the alert topology.

```ts
type ErrorEvent = {
  message: string;
  name: string;
  stack?: string;
  attributes: Record<string, string | number | boolean | null>;
};

function buildErrorEvent(failure: ImportFailure): ErrorEvent {
  const error = failure.error instanceof Error
    ? failure.error
    : new Error(String(failure.error));

  return {
    message: error.message,
    name: error.name,
    stack: error.stack,
    attributes: {
      "error.group": groupKey(failure),
      "import.job": failure.jobName,
      "import.source": failure.source,
      "import.schedule": failure.schedule,
      "import.file": failure.fileName,
      "import.received_at": failure.receivedAt,
    },
  };
}
```

A tracking service can use `error.group` as the grouping field, or the adapter can translate it to that service's equivalent. The application should not need to know the destination's UI vocabulary. This is the boundary that keeps migration and local testing cheap.

The code still records the file name because it is useful evidence. It is not used to identify the issue. That distinction matters for cardinality. High-cardinality values are excellent for investigation and poor for alert identity.

I started by treating the exception message as the obvious key. That looked reasonable in a local test with three fixtures. It failed as soon as a message included a file name and record number. The correction was not a clever regular expression; it was deciding which fields describe a cause and which fields describe an occurrence.

Test the policy with a small matrix:

```ts
const cases = [
  ["same parser frame, different files", true],
  ["same exception, different source mappings", false],
  ["same failure, different retry attempt", true],
  ["different parser frames", false],
] as const;
```

The assertions are omitted here because the useful part is the decision table, not a fake universal parser. Your fixtures should contain real stack shapes from the Node.js versions and build settings you deploy. A source-map change can alter grouping without changing application behavior, so include a deploy in the test review.

## When is the runner-up better?

Full-stack grouping is the runner-up when the import worker is small, the deployment artifact is stable, and a frame-level distinction is operationally meaningful. It preserves more context with less policy code. The cost is churn when bundling, transpilation, or source maps change.

Manual incident keys are better for a narrow contractual alert such as `warehouse-a:nightly-orders`. They are easy to explain and easy to audit. They are also blind to a new failure mode inside that contract, so they should not be the only error grouping mechanism.

Raw-message grouping has one legitimate use: short-lived diagnosis during development. It should not be the production default for scheduled imports. Dynamic values turn ordinary input variation into a stream of unrelated issues, which hides the fact that the importer has one broken path.

The operational check is simple: can a responder tell what action changes between two groups? If not, merge them. Can they safely act on one group without touching the other? If yes, split them. Then validate the result against the missing-result watchdog, because a tidy error list is not proof that imports are healthy.

## References

- https://opentelemetry.io/docs/concepts/signals/logs/
- https://opentelemetry.io/docs/specs/otel/logs/data-model/
- https://nodejs.org/api/errors.html
- https://nodejs.org/api/async_context.html
