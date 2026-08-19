# Fintech Call Gates: Cheap LLM Moderation in Node.js Before Text, Images, and JSON Schema

Short answer: use a cheap admission gate before LLM moderation, keep the sales-call evidence small, and require a typed JSON verdict before writing CRM actions. The gate should estimate text and image workload, but it must not pretend that an estimate is a billing receipt.

| Choice | Best fit | Main thing you still own |
|---|---|---|
| Hosted classifier | A narrow policy with an approved external model path | Privacy review, evaluation, and the verdict contract |
| Local classifier | Stable labels, enough examples, and an operator for drift | Deployment, retraining, and incident response |
| Two-stage gate | Mixed sales-call traffic where most items are easy | Thresholds, queue policy, and fallback behavior |

For a fintech sales-call workflow, I would start with the two-stage gate. Normalize the transcript, reject obviously oversized inputs before inference, and send only the uncertain or high-value evidence to the classifier. Put image attachments through the same budget check. Then convert the result into a small set of CRM actions such as `create_follow_up`, `request_review`, or `no_action`.

That is the decision. The interesting work is making it boring under load.

The useful mental model is a conveyor belt with a stop button. A new transcript enters with an event ID, policy version, and evidence references. The gate strips noise and checks the input budget. The classifier proposes one typed action. A policy check compares that action with the account and authorization context, then either writes one idempotent CRM event or sends the item to a human queue. If any stage cannot explain its decision, the item stops. It's less glamorous than a universal prompt, and much easier to replay when a compliance reviewer asks why a follow-up was created.

## The audit trail is the product

Treat estimation as admission control. It answers “can this item enter the expensive path?” before a model call, while classification answers “what should happen to this item?” Mixing those questions makes both harder to test.

The input for a sales call usually has several pieces: a transcript, an account identifier, selected metadata, and perhaps a screenshot or scanned document. Do not pass the entire CRM record because it happens to be available. Select the fields that can change the policy decision. Strip repeated headers and whitespace. Preserve the original item ID outside the model prompt so the model does not become the source of identity.

There are two useful estimates. A rough local estimate can enforce a hard ceiling before any tokenizer or API is involved. A provider-specific count, if the chosen inference path exposes one, is better for comparing candidates. Neither one should be used as a fabricated promise of exact spend. Tokenization depends on the actual message format and model, and an image has its own accounting rules.

Here is the small boundary I want in a Node.js service. The `TokenCounter` is deliberately injected: the application can use an exact counter for the selected model, or a conservative counter in a dry run. The cost calculation is also injected because prices and currencies are deployment configuration, not moderation policy.

```ts
type Evidence = {
  transcript: string;
  imageBytes?: number;
};

type TokenCount = {
  input: number;
  output: number;
};

type TokenCounter = (text: string) => number;

type RateCard = {
  inputPerToken: number;
  outputPerToken: number;
};

function estimateWork(
  evidence: Evidence,
  countTokens: TokenCounter,
  rateCard: RateCard,
): TokenCount & { estimatedCost: number } {
  const text = evidence.transcript.replace(/\\s+/g, " ").trim();
  const input = countTokens(text);
  const output = 160;

  return {
    input,
    output,
    estimatedCost:
      input * rateCard.inputPerToken + output * rateCard.outputPerToken,
  };
}

function admit(evidence: Evidence, inputLimit: number): "classify" | "review" {
  const textLength = evidence.transcript.trim().length;
  const imageTooLarge = (evidence.imageBytes ?? 0) > 8_000_000;

  return textLength <= inputLimit && !imageTooLarge ? "classify" : "review";
}
```

The byte threshold above is an application policy example, not a claim about a model's image limit. Set it from the storage and review contract. The same applies to the output ceiling. A CRM action does not need a paragraph-long essay, so giving the model a large prose budget is usually just giving the parser more ways to fail.

I benchmark this boundary. I care about time-to-first-call, but I also count glue: config branches, credentials, adapters, and test fixtures. A preflight that requires a second policy engine and three subtly different payload builders has not made moderation cheap. It has moved complexity into a darker corner.

## How should Node.js estimate token cost before classifying text and images?

Moderation here is not a generic “safe or unsafe” badge. A sales-call summary can be factually plausible and still create the wrong CRM task. The classifier should return an action, a reason code, and a small evidence note. The application should decide whether that action is allowed to mutate the CRM.

Keep the schema closed. Require every field. Reject unknown actions. A malformed response must enter review or retry according to an explicit policy; it must never silently become `no_action`.

```ts
type CrmAction = "create_follow_up" | "request_review" | "no_action";

type CrmVerdict = {
  action: CrmAction;
  reasonCode: string;
  evidenceNote: string;
};

function parseVerdict(raw: string): CrmVerdict {
  const value: unknown = JSON.parse(raw);

  if (typeof value !== "object" || value === null || Array.isArray(value)) {
    throw new Error("Expected a JSON object");
  }

  const record = value as Record<string, unknown>;
  const expected = ["action", "evidenceNote", "reasonCode"];
  const keys = Object.keys(record).sort();
  if (keys.length !== expected.length || keys.some((key, i) => key !== expected[i])) {
    throw new Error("Unexpected verdict fields");
  }

  const actions: CrmAction[] = ["create_follow_up", "request_review", "no_action"];
  if (!actions.includes(record.action as CrmAction)) {
    throw new Error("Unknown CRM action");
  }
  if (typeof record.reasonCode !== "string" || record.reasonCode.length === 0) {
    throw new Error("Missing reason code");
  }
  if (typeof record.evidenceNote !== "string" || record.evidenceNote.length === 0) {
    throw new Error("Missing evidence note");
  }

  return record as CrmVerdict;
}
```

The schema should describe those same three fields with `additionalProperties: false`, required fields, and an enum for `action`. The parser still matters. JSON validity is not policy validity, and schema validity is not evidence that the CRM mutation is justified.

A useful test fixture has a short transcript, a long transcript with repeated speaker labels, an image-only submission, and a transcript-image pair that disagree. Add cases with redacted account data and malformed model output. The point is not to collect a heroic benchmark number. It is to make the failure modes visible before a sales representative sees a phantom follow-up task.

## Replay the queue before tuning latency

Quality and latency are the primary axes. Cost is a constraint, not the definition of quality. For each fixture, record the admission decision, input and output counts, end-to-end latency, classifier verdict, parser result, and whether a human later changed the action. That gives the team a way to see a cheap path degrading quality or a fast path filling the review queue.

Measure the tails. A median latency that looks fine can hide a slow image path. A low average review rate can hide one customer segment that is consistently misrouted. Keep the original evidence reference and a version for the prompt, schema, and policy so an incorrect action can be reconstructed without logging sensitive transcript text by default.

Retries need a ceiling. I record `429` as a distinct test outcome, honor a server-provided retry delay when one exists, and cap attempts before sending the item to review. An identical request repeated forever is a second moderation bug hiding inside the first. A 429 is a test case, not a personality trait. For CRM writes, use an idempotency key derived from the source event and policy version. A classifier may be read-like, but the action it enables is not.

Three words: queue the uncertain.

Stop.

Do not force a binary answer when the evidence is incomplete. A `request_review` result is a normal product outcome for a low-latency system. It also gives the evaluation set a useful signal: reviewers can distinguish ambiguous content from a genuinely wrong classification instead of flattening both into a single error bucket. Don't hide that queue behind a success metric.

The replay set should be assembled from the handoffs that actually cost the team time: a short transcript that clearly asks for a callback, a long transcript with repeated speaker labels, an image containing a document that belongs to another account, and a transcript-image pair that disagree about the next step. Add a parser response with an unknown field, an empty reason code, and an action that is valid JSON but absent from the enum. Then replay the same fixtures after changing the prompt, schema, tokenizer, input limit, and retry policy. This is where the quality-versus-latency trade-off becomes legible. If the fast path creates fewer CRM writes but doubles the review queue, the system did not improve; it changed who pays. If a conservative gate sends more items to review but makes account mismatches visible, that may be the correct fintech decision. Your mileage may vary because the right threshold depends on the cost of a missed follow-up and the cost of a false one. The test should make that business choice explicit instead of hiding it in a model score.

## When should the admission gate be removed?

The two-stage design is not suitable when the team cannot maintain a labeled evaluation set or a human-review path. The limitation is operational ownership, not model capability. In that case, adding an admission gate creates thresholds nobody owns. A simpler classifier with a strict input contract may be easier to operate, even if it sends more items to inference.

A local model is a poor fit when the policy changes weekly, the labels are sparse, or no one can monitor drift. Choose it when the label set is narrow and deployment ownership is real. The hosted route is a poor fit when the call transcript or image cannot leave the approved data boundary; choose a local or internally governed path then.

I am not sure a generic token estimate can settle the image question for your workload. The missing fact is the exact tokenizer and image accounting behavior of the selected model. Resolve that with a provider-specific count and a replay set, not with a prettier spreadsheet.

The final rule is plain: keep product logic independent of the classifier, keep the JSON contract smaller than the prompt, and let latency trade against review volume where the business can see it. For fintech sales calls, a typed CRM action is more useful than an impressive summary.

## Further reading

- Cohere Rerank overview: https://docs.cohere.com/docs/rerank-overview
- OpenAI Whisper repository: https://github.com/openai/whisper
