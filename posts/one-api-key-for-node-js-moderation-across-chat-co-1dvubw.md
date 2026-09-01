# One API Key for Node.js Moderation Across Chat Completions with Structured Safety Output

Short answer: make the tenant boundary an input to the moderation request, and make the safety result a versioned application contract. A provider-neutral Node.js adapter can put several chat backends behind one key, but it cannot make their judgments equivalent. Parse structured output, record usage with the decision, and send disagreement to review instead of pretending the classifier is an oracle.

That is the useful answer for an e-commerce marketplace scoring candidates against a job rubric. The safety gate comes first. Rubric scoring comes second. Tenant attribution belongs to both records. These are three small contracts, and merging them into one model prompt creates an audit problem disguised as a DX improvement.

## How can Node.js moderation keep a structured-output safety classifier portable across chat providers?

Start with the application boundary, not a catalog of models. The rest of the service should know about messages, a schema name, tenant metadata, and normalized usage. It should not know a vendor SDK's response type or a transport-specific option. That keeps the switching decision local to an adapter and makes the policy code testable with ordinary fixtures.

The interface can stay this small:

```ts
type ChatMessage = {
  role: "system" | "user";
  content: string;
};

type RuntimeReply = {
  content: string;
  usage?: {
    inputTokens: number;
    outputTokens: number;
  };
  provider: string;
};

type ChatRuntime = {
  complete(input: {
    messages: ChatMessage[];
    schemaName: string;
    metadata: { tenantId: string; requestId: string };
  }): Promise<RuntimeReply>;
};

type SafetyDecision = {
  allowed: boolean;
  categories: string[];
  severity: "low" | "medium" | "high";
  reason: string;
};

function parseSafety(content: string): SafetyDecision {
  const value = JSON.parse(content) as Partial<SafetyDecision>;
  if (
    typeof value.allowed !== "boolean" ||
    !Array.isArray(value.categories) ||
    !["low", "medium", "high"].includes(value.severity ?? "") ||
    typeof value.reason !== "string"
  ) {
    throw new Error("Invalid safety decision");
  }
  return value as SafetyDecision;
}
```

The parser is a gate, not a policy. `allowed` says what the classifier returned; it does not say what a particular tenant must do with a high-severity result. That rule belongs in application code, where it can be reviewed and unit-tested without making a model call.

There is a sharp edge here. Valid JSON is not valid judgment. A quoted threat from a novel, mixed-language slang, or a description of a restricted product may be syntactically perfect and still need a human disposition. Portability means the response shape travels. It does not mean the decision does.

## How does one API key support Node.js moderation and tenant cost visibility?

The cost question is answered by metadata, not by a dashboard added later. Generate a stable `requestId` at the application boundary and carry `tenantId`, `applicationId`, `policyVersion`, and the request timestamp into the usage record. Store the provider label returned by the adapter, too. One key is useful for credential management; it is not an attribution system.

Keep it boring.

| Record | Must answer | Do not infer later |
| --- | --- | --- |
| Decision | Which policy allowed, held, or rejected the content? | Tenant ownership from a log line |
| Usage event | Which tenant consumed the request? | Token counts from character length |
| Review event | Which application needs human attention? | A second event from an untracked retry |

I use four identifiers because fewer leave an uncomfortable ambiguity: a tenant can have several applications, and a policy can change while an application is being reviewed. The ledger should capture the normalized input and output token fields when they exist. If usage is unavailable, record `usageUnavailable`; don't estimate tokens from character count and call that visibility.

The ordering matters. First validate the response. Then apply the tenant's policy. Then write an idempotent decision and usage event keyed by `requestId`. A retry before the write may repeat inference; a retry after the write must not create a second review ticket or a second tenant charge. Keep that behavior explicit in the consumer. In a real marketplace flow, this means `shop-17` can submit a short candidate message while `shop-42` submits a longer, mixed-language application and both records remain attributable even when they share credentials, a schema, and a review queue. If the adapter returns only a boolean, the application loses the policy version and usage context needed to explain the result. If it writes usage before parsing, malformed output becomes a billing event; if it writes after an unprotected review enqueue, a retry can duplicate the side effect. The ledger is therefore a boundary around an application decision, not a callback owned by whichever chat backend answered first.

This is also why rubric scoring should not share the safety schema. A score of zero can mean “poor candidate match” or “held for safety review,” and those are not remotely the same event. Separate records make the distinction visible to recruiters, support staff, and whoever has to explain a tenant's usage.

## Build the smallest working moderation flow

The orchestration layer can now stay boring. It asks for a named result, parses it, and returns the evidence needed by the next stage. It does not calculate a price, infer a tenant from logs, or silently convert parser failure into rejection.

```ts
async function moderateCandidate(
  runtime: ChatRuntime,
  input: {
    tenantId: string;
    applicationId: string;
    requestId: string;
    text: string;
  },
) {
  const reply = await runtime.complete({
    messages: [
      {
        role: "system",
        content:
          "Classify candidate content for marketplace safety. Return only the named structured result.",
      },
      { role: "user", content: input.text },
    ],
    schemaName: "candidate-safety-v1",
    metadata: {
      tenantId: input.tenantId,
      requestId: input.requestId,
    },
  });

  const decision = parseSafety(reply.content);
  return {
    tenantId: input.tenantId,
    applicationId: input.applicationId,
    requestId: input.requestId,
    policyVersion: "candidate-safety-v1",
    decision,
    usage: reply.usage ?? null,
    provider: reply.provider,
  };
}
```

The adapter owns provider-specific authentication and request translation. The moderation service owns the schema and policy. That division is the reason a shared key can reduce configuration glue without making the application dependent on a private SDK type.

At scale, I would add a model-selection policy with explicit fields for geography, latency, capability, and tenant rules. Persist the selected provider next to the decision. I would also keep sensitive candidate text out of routine logs, redact identifiers where possible, and define retention before a reviewer asks for an old case. Access control and audit trails are engineering requirements, not cleanup work; the HIPAA Security and Privacy Rules are a useful reference for that discipline even though they are not a universal checklist for an e-commerce marketplace.

## What should teams test before trusting unified chat completions for moderation?

Build a compact evaluation corpus from the actual marketplace taxonomy. Include clearly allowed applications, borderline language, quoted text, mixed-language messages, empty content, and deliberately malformed output. Keep the fixture ID, policy version, provider label, parsed result, human disposition, and usage record for each run.

Compare disagreement by category and tenant. One blended accuracy number hides the case that matters: a classifier may look acceptable overall while producing too many review holds for one tenant's job family. The release gate should include parse failure rate, review rate, p95 latency, and cost-record completeness per tenant. Measure the boring fields. They are the fields support can actually use.

Offline evaluation and live gating are different jobs. A batch evaluation can be repeatable and richly annotated; a synchronous moderation decision needs bounded latency and a clear review path. The Batch API guide is useful background for the former, but it should not be treated as a live moderation contract.

## When is a unified moderation runtime the wrong fit?

The catch is that a chat classifier is not suitable when a tenant requires a mandated moderation control, a specific data-residency boundary, or a taxonomy the application cannot reproduce and review. Use the required control then. Keep a direct integration when model switching has no operational value and the extra adapter would only add maintenance.

It is also the wrong shape for deterministic rules. Run a denylist, field validator, or conventional rules engine first when the policy can be expressed that way. Reserve model inference for ambiguity, and record why it was needed. Your mileage may vary by language and category; I'm not sure one cross-provider score will predict every tenant's review burden.

The durable design is modest. One key can reduce credential sprawl. A unified chat interface can reduce transport glue. Neither replaces an owned policy, an evaluation corpus, an idempotent ledger, or human review. Keep those boundaries visible, and the next provider change stays an adapter exercise instead of a data-reconstruction project.

## References

- Batch API guide: https://platform.openai.com/docs/guides/batch
- 45 CFR Part 164, Security and Privacy Rules: https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164
