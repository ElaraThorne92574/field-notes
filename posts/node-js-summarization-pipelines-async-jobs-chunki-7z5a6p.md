# Node.js Summarization Pipelines: Async Jobs, Chunking, and Export Integrity

Short answer: for multiple-document summarization, submit a durable async job, poll with backoff, and treat the exported file as an item-by-item ledger. Keep a bounded worker loop for small or interactive runs. The deciding factor is not a flashy model demo; it is whether you can recover, reconcile, and re-export after a process restart.

Here is the choice matrix I use before writing code:

| Pattern | Best fit | Operational burden | Failure you must design for |
| --- | --- | --- | --- |
| Inline sequential calls | One or two short documents | Low | Request timeout and partial retries |
| Bounded worker loop | Small, latency-sensitive batches | Medium | Lost in-flight state after a restart |
| Managed async job | Large, variable batches with a file deliverable | Medium | Queue delay and partial item outcomes |
| Self-hosted queue and workers | A pipeline you already operate | High | Broker, worker, and dead-letter recovery |

The least complex option that meets the deadline wins. For a run that can wait, a managed async job usually reduces glue code. It is not suitable when a user needs a streamed answer in seconds; use a bounded worker or synchronous request in that case.

## What should a batch summarization API guarantee for multiple documents and exported results?

Start with the contract, not the SDK. A useful API exposes four durable facts: a submission identifier, a status resource, a terminal state, and a result export that can be fetched again. HTTP `202 Accepted` means the server accepted work for later processing; it does not mean every document succeeded. `Retry-After` is a useful hint for polling cadence, but your client still needs a cap and a deadline.

The input needs a stable application id for every document. Do not use the array index. A retry, a reordered upload, or a deduplicated source can make index-based reconciliation silently attach a summary to the wrong document. Persist the document id, content hash, prompt version, and submission id before enqueueing work. Those fields are cheap insurance when an export arrives a day later.

The export is a data contract. Newline-delimited JSON (JSONL) can be streamed and diffed without loading thousands of summaries into memory. A JSON array is easier to read but forces buffering or a full parser pass. Either shape can work; the important requirement is one explicit outcome per input id, including a typed rejection or policy decision. A job-level `completed` flag must never be treated as proof that every item has a summary.

## What fails between async jobs, chunking, and result export?

Think in layers. The intake layer validates text, assigns ids, and records a content hash. The planner measures tokens and splits oversized documents into chunks. The job layer submits those chunks with an idempotency key. The collector polls status, downloads the export, and reconciles it against the planned set. A finalizer joins chunk summaries, writes the document-level result, and records provenance. That sounds tidy on a diagram, but the edges are where systems drift: an intake retry can create a second run, a chunk can be accepted while its parent document is rejected, and an export can be available before every object has arrived in your object store. Keep a state record for each transition, include the run key in every log line, and make the join operation repeatable. You don't want a successful retry to create a second summary or silently overwrite a newer prompt version.

Chunking changes the unit of accounting. A 500-document request may become 2,700 chunks, so rate limits, retries, and storage quotas must be sized for chunks. Character counts are a poor proxy for tokens across code, tables, and non-English text. Use a tokenizer appropriate to the target model; `tiktoken` is one public example of a BPE tokenizer library, but the general rule is to measure with the same family of tokenizer used by the inference service.

Keep polling out of the web request. A small durable worker can wake up, fetch status, and stop when the job is terminal. Exponential backoff with jitter avoids a thundering herd when many jobs are waiting. Persist the next poll time so a deploy does not reset every job to an immediate request.

There is a practical distinction here. Queue latency is part of the product's latency even when it is hidden behind a neat progress bar. Measure submit-to-first-progress, submit-to-terminal, export-download, and reconciliation duration separately. I benchmark those phases independently because a fast model cannot rescue a slow queue or a download that cannot be resumed.

Measure it.

## A small TypeScript collector that fails loudly

The following example uses generic HTTP paths. Adapt the paths and response fields to the service contract you have selected. It deliberately keeps the provider-specific surface behind three functions and makes reconciliation the last mandatory step.

```ts
import { appendFile } from "node:fs/promises";
import { setTimeout as sleep } from "node:timers/promises";

type Document = { id: string; text: string };
type ExportItem = { id: string; summary?: string; error?: { code: string; message: string } };
type Job = { id: string; state: "queued" | "running" | "succeeded" | "failed"; exportUrl?: string };

const endpoint = process.env.SUMMARY_ENDPOINT!;
const token = process.env.SUMMARY_TOKEN!;

async function request(path: string, init: RequestInit = {}): Promise<Response> {
  return fetch(`${endpoint}${path}`, {
    ...init,
    headers: {
      authorization: `Bearer ${token}`,
      "content-type": "application/json",
      ...init.headers,
    },
  });
}

async function submit(documents: Document[], runKey: string): Promise<string> {
  const response = await request("/jobs", {
    method: "POST",
    headers: { "idempotency-key": runKey },
    body: JSON.stringify({
      items: documents.map(({ id, text }) => ({ id, text })),
      instruction: "Write a concise factual summary and preserve dates and names.",
    }),
  });
  if (!response.ok) throw new Error(`submit failed: ${response.status}`);
  return ((await response.json()) as { id: string }).id;
}

async function collect(jobId: string, expected: Set<string>, output: string): Promise<void> {
  for (let attempt = 0; ; attempt++) {
    const statusResponse = await request(`/jobs/${jobId}`);
    if (!statusResponse.ok) throw new Error(`status failed: ${statusResponse.status}`);
    const job = (await statusResponse.json()) as Job;

    if (job.state === "failed") throw new Error(`job ${jobId} failed`);
    if (job.state === "succeeded" && job.exportUrl) {
      const exportResponse = await fetch(job.exportUrl);
      if (!exportResponse.ok) throw new Error(`export failed: ${exportResponse.status}`);
      const lines = (await exportResponse.text()).trim().split("\n").filter(Boolean);
      for (const line of lines) {
        const item = JSON.parse(line) as ExportItem;
        if (item.summary || item.error) expected.delete(item.id);
        await appendFile(output, `${JSON.stringify(item)}\n`);
      }
      if (expected.size > 0) {
        throw new Error(`unaccounted document ids: ${expected.size}`);
      }
      return;
    }

    const delay = Math.min(30_000, 1_000 * 2 ** Math.min(attempt, 5));
    await sleep(delay);
  }
}
```

The idempotency key is tied to a logical run, not a process instance. A restart can safely retry submission. The collector deletes an id only when the export contains either a summary or an explicit item error; missing ids stay visible. That is the useful kind of failure at 3 a.m.

Production code should stream large exports rather than call `text()` and split the whole body. It should also verify the export URL's trust boundary, cap total polling time, and make the output write atomic or append-only. For a large corpus, that means a line parser that carries a partial final line between reads, a checksum recorded beside the downloaded object, and a transaction that marks an item publishable only after its summary and provenance are both durable. These are boring details. Boring details are what keep a retry from becoming duplicate data.

## When is a bounded worker loop the better choice?

Choose a local queue when the batch is small, the caller needs progress quickly, or the provider's queue delay violates your service-level objective. A semaphore limits concurrency; a retry budget handles transient responses; a durable table records each document state. You retain ordering and can stream partial summaries to a UI.

The catch is ownership. You now own backpressure, dead letters, worker restarts, and rate-limit coordination. A loop in one process is not durable just because it is `async`. Persist the work item before starting the call, and make the transition from `running` to `succeeded` idempotent. Stick with a managed job when you do not already operate this machinery and the user can wait for an export.

Security and observability cut across both patterns. Redact secrets before logging, attach a trace id to submission and export records, and retain the exact prompt or instruction version used for each run. Track counts for accepted, rejected, retried, and reconciled items. Do not log document text by default.

There is no universal threshold such as “use batch above 50 documents.” Corpus size, token distribution, queue delay, and the required response time decide it. Your own measurements should settle that boundary; your mileage may vary.

## References

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [Idempotency-Key HTTP header field](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/)
- [JSON Lines](https://jsonlines.org/)
- [openai/tiktoken](https://github.com/openai/tiktoken)
- [Node.js timers/promises](https://nodejs.org/api/timers.html#timerspromisessettimeoutdelay-value-options)

## Further reading

- [ElevenLabs documentation](https://elevenlabs.io/docs)
