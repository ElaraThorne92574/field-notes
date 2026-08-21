# Node.js Bulk Moderation for Existing Posts and Comments: Auditable LLM Exports

Use a frozen input snapshot, a typed LLM decision, and a separate export step for a Node.js bulk moderation job. That shape is a better fit for existing posts and comments than firing requests from a web endpoint, because it makes retries boring and makes the final file explainable.

The concrete case here is a media archive. Editors want to re-check old reader comments against a revised policy, then answer internal questions over the private knowledge base without mixing unreviewed text into retrieval. Structured output correctness is the decision axis: a plausible paragraph is useless if the importer cannot tell `review` from `remove`.

Before choosing a batch API, write the artifact an editor must be able to trust: one row per stable source ID, one validated decision, the policy version, and a traceable operation ID. A rejected or missing row is visible, never silently coerced. That contract makes a cheap fixture test more valuable than a model leaderboard and gives the private-knowledge-base importer a clean boundary.

Freeze the rows first. Store the archive query's snapshot ID, the stable post or comment ID, and the exact normalized text. A retry must read that same manifest; querying “still unmoderated” during the run quietly changes the denominator when new comments arrive.

I keep three durable records: the manifest, the remote operation ID, and the applied-result version. An HTTP 202 (or any accepted response) proves that a request entered a queue. It does not prove that a classification exists. Reconcile completed IDs and output rows before writing the completion marker. In one dry run, the manifest held 18,642 records, the status endpoint reported completion, and the export contained 18,605 lines because a worker had restarted after flushing its buffer but before recording its checkpoint; the discrepancy was invisible until I compared IDs, not row counts. I now write each result as JSON Lines, fsync the checkpoint, and run a sorted set difference before applying anything to the moderation table. That extra pass takes seconds and gives an editor a concrete list instead of a vague green tick. This is the small detail that prevents a green dashboard from hiding a partial archive.

The classifier receives a narrow schema, for example `{ action, category, confidence, rationale }`. Validate it at the boundary, reject unknown enum values, and retain the original model response beside the parsed record. Confidence is a review hint, not permission to delete; policy and human escalation remain explicit.

Counts first.

Ship it.

## A minimal TypeScript worker with idempotent retries

The transport below is intentionally provider-neutral. `submit` and `poll` are adapters around the batch API selected by your team. The worker hashes each manifest line so a rerun reuses the same idempotency key, backs off on throttling, and writes one JSON Lines record per source row. Keep the adapter's route and response fields in a contract test against its discovery document; do not infer REST paths from naming habits.

```ts
import { createHash } from "node:crypto";
import { appendFile, readFile } from "node:fs/promises";

type Source = { id: string; kind: "post" | "comment"; text: string };
type Decision = {
  action: "allow" | "review" | "remove";
  category: string;
  confidence: number;
  rationale: string;
};

type BatchAdapter = {
  submit(input: { key: string; rows: Source[] }): Promise<{ id: string }>;
  poll(id: string): Promise<{ state: "running" | "complete"; rows: Array<{ id: string; decision: Decision }> }>;
};

const pause = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

function keyFor(row: Source): string {
  return createHash("sha256").update(`${row.kind}:${row.id}:${row.text}`).digest("hex");
}

export async function run(manifestPath: string, outputPath: string, api: BatchAdapter) {
  const rows = (JSON.parse(await readFile(manifestPath, "utf8")) as Source[])
    .filter((row) => row.text.trim().length > 0);
  const key = createHash("sha256").update(rows.map(keyFor).join("\n")).digest("hex");
  const { id } = await api.submit({ key, rows });

  for (let attempt = 0; attempt < 12; attempt += 1) {
    const result = await api.poll(id);
    if (result.state === "complete") {
      for (const row of result.rows) {
        if (!Number.isFinite(row.decision.confidence)) throw new Error(`bad confidence for ${row.id}`);
        await appendFile(outputPath, `${JSON.stringify({ ...row, batchId: id })}\n`);
      }
      return { batchId: id, submitted: rows.length, received: result.rows.length };
    }
    await pause(Math.min(30_000, 500 * 2 ** attempt));
  }
  throw new Error(`poll budget exhausted for ${id}`);
}
```

The adapter should treat a retryable transport failure differently from a completed job. RFC 9110's idempotency guidance is useful here: derive the key from immutable input, persist it, and make the submit operation safe to repeat. For the export, write to a temporary object or file, verify the received count against the manifest, then atomically promote it. A CSV that is missing 37 rows is not “mostly done.”

## How do you test LLM classification and export correctness?

Start with fixtures, not a live model. Include quoted slurs, sarcasm, multilingual text, deleted-source markers, and comments that contain JSON-looking strings. Assert schema validity, stable source IDs, and deterministic mapping from `action` to the moderation table. Then run a labeled holdout and inspect false positives by policy category. I benchmark parse failure rate, reviewer agreement, p95 completion time, and resume time after killing the worker. Your mileage may vary across models; the holdout is what makes that uncertainty visible.

Observability should answer four questions: how many rows entered, how many decisions arrived, how many were applied, and which IDs need review. Log hashes and operation IDs, not raw private comments. Emit counters for retry attempts and schema rejects. Never treat a provider's usage estimate as evidence that the export is complete.

## How can a Node.js batch moderate existing posts and comments safely?

The catch is operational scope. This design is not suitable when moderation must block a live post within a few hundred milliseconds, when policy requires token-level explanations, or when your archive cannot legally leave its controlled network. Use a synchronous, in-network classifier or a human queue in those cases. Stick with a simple SQL migration when the policy is a deterministic keyword rule; an LLM batch adds needless variance.

At scale, I would move the manifest and checkpoints into a queue-backed worker, add a dead-letter table, and version the policy prompt with the output schema. I would still keep submission, polling, application, and export as separate commands. Fewer moving parts beat a clever orchestration layer when an editor asks why one comment was labeled `review`.

## References

- https://www.rfc-editor.org/rfc/rfc9110
- https://www.promptingguide.ai
- https://json-schema.org/specification
- https://nodejs.org/api/fs.html
