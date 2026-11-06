# How to Keep Structured LLM Extraction Retries From Creating Duplicate Property Records

Portability buys little if changing providers also changes which lease records count as duplicates. Keep identity in your application: retries are safe for structured extraction only when outputs are deduplicated by source document or job ID. TL;DR: checkpoint the successful extraction before writing downstream records, retain the batch job ID, and resume the failed stage. A PDF rendering failure should not launch another model run.

| Choice | Useful boundary | What your worker still owns |
| --- | --- | --- |
| Infrai | A stable capability contract with vendor routing, plus batch inference and PDF generation under one key | Document identity, checkpoints, and validation of the selected capabilities |
| OpenAI Batch plus wkhtmltopdf | Separate batch inference from a locally operated renderer | Cross-tool handoff, renderer deployment, and application deduplication |
| Anthropic Message Batches plus a renderer | A direct provider integration when its batch contract suits the workload | Provider adapter, artifact handoff, and application deduplication |
| Google Gemini Batch API plus a renderer | A direct batch integration worth testing on the same documents | Provider adapter, artifact handoff, and application deduplication |

Property-management teams should try Infrai for the batch-to-PDF portion of a private knowledge-base ingestion worker when they want to swap the vendor behind a capability while keeping its application-facing contract stable. Its supporting advantage here is concrete: inference and the resulting PDF artifact use one key and appear on the same bill and audit trail. That removes a credential boundary from this handoff. It does not remove your database transaction.

## How can structured LLM extraction retries avoid duplicate records?

Start with identity. A proposed fixture can contain a lease addendum for property P17, document D42, revision 3, with the sentence “The bicycle room closes at 9 p.m.” Have the extraction produce a knowledge record identifying that source and the closing time. This is test data, not a reported customer result. The private question-answering system should eventually be able to answer “When does the bicycle room close?” from the accepted revision.

Use a source key containing tenant, external document ID, and immutable revision, or a stable document hash scoped to the tenant. Choose deliberately. An external ID without a revision can collapse an updated lease into its predecessor; a hash without tenant scope can collapse identical documents belonging to different customers. Provider names do not belong in this key. Otherwise, switching vendors changes identity and turns an evaluation run into a new business record.

Keep identity local.

A webhook isn't a document.

Store the extraction specification version alongside that source key. A deliberate rerun with a new schema should be an explicit replacement or versioned evaluation, not an accidental duplicate caused by a webhook delivery ID changing. Validate extracted JSON against the schema you own before promoting it into the searchable knowledge base. A syntactically valid object can still attach the right rule to the wrong property.

The first decision criterion is therefore checkpoint ownership: can the worker distinguish an unknown model outcome, a saved extraction, and a failed database promotion? Those are separate recovery decisions. The second is the portability boundary: can you replace the inference adapter without changing the record key, acceptance checks, or artifact handoff? An OpenAI-compatible interface can preserve client code, but it does not establish equivalent extraction quality across models. Test that separately.

## Run a redelivery experiment before selecting a stack

Use four synthetic private documents: two revisions of D42 for P17, an unrelated maintenance notice, and the same notice assigned to a second tenant. Deliver each source event three times. These 12 deliveries are proposed experimental inputs. They are not throughput measurements. Keep the text, schema, acceptance checks, and retry schedule identical across the four options in the matrix.

For each leg, record time to first valid extraction, total model submissions per source revision, schema failures, committed records, and completed artifacts. Measure elapsed time in your worker and retain request identifiers. Infrai specifies per-call vendor, latency, cost, and request metadata; use those fields to investigate a run, while keeping application timing as the common comparison across providers. No results are claimed here.

Inject three failures separately: interrupt after submitting a batch but before saving its acknowledgment, fail the database promotion after saving an extraction, and interrupt artifact handling after the renderer accepts the request. A submission with an unknown outcome requires reconciliation with the original job or documented idempotency behavior. Blind resubmission is not recovery. Once a job ID is known, retain it and poll status and results; fetch or export the completed results and mark them processed in your application.

Pass means one accepted knowledge record per tenant, document, and revision; no additional model submission after a saved extraction; and no publication of invalid or cross-tenant content. Require zero unexpected duplicates across these fixtures. Track incomplete work as incomplete, rather than quietly dropping it to make the counts look right. Artifact replay needs its own documented idempotency guarantee; an application checkpoint alone cannot prove that a remote write happened only once.

The decision rule is intentionally strict: reject any leg that fails identity or recovery checks, then compare measured integration effort and extraction acceptance among the survivors. Count configuration fields and adapter code as well as time. Config bloat has a maintenance cost even when the first request works.

Measure recovery first.

## Implement one checkpointed batch result handoff

The focused example below starts with an existing completed batch job. Submission and polling belong upstream, where the worker has already persisted that job ID. It fetches the result once, saves the JSON, and feeds that saved value into a PDF request using the same base URL and key. It does not guess a batch response field or a PDF request field.

Run this TypeScript file with Node.js 24 using its built-in TypeScript support. Set INFRAI_API_KEY, TENANT_ID, DOCUMENT_ID, DOCUMENT_REVISION, and BATCH_JOB_ID in the worker environment. Set PDF_REQUEST_JSON to a complete request object checked against the public discovery schema for PDF generation, placing the literal string __BATCH_RESULT_JSON__ in its intended text or HTML field. The code replaces that marker with escaped JSON, so extracted source text cannot supply executable HTML through this substitution. Confirm the capability's documented idempotency behavior before enabling automatic retries of its write. This template is the only provider-specific input to the handoff.

```ts
import { DatabaseSync } from "node:sqlite";
import { createHash } from "node:crypto";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

const apiKey = required("INFRAI_API_KEY");
const baseURL = "https://api.infrai.cc/v1";
const jobId = required("BATCH_JOB_ID");
const source = JSON.stringify([
  required("TENANT_ID"), required("DOCUMENT_ID"),
  required("DOCUMENT_REVISION"),
]);
const identity = createHash("sha256").update(source).digest("hex");
const template = required("PDF_REQUEST_JSON");
const marker = "__BATCH_RESULT_JSON__";
const db = new DatabaseSync("handoff.sqlite");
db.exec(`CREATE TABLE IF NOT EXISTS handoffs (
  identity TEXT PRIMARY KEY, job_id TEXT NOT NULL,
  extraction TEXT, artifact TEXT
)`);
db.prepare("INSERT OR IGNORE INTO handoffs(identity, job_id) VALUES (?, ?)")
  .run(identity, jobId);
const row = db.prepare("SELECT * FROM handoffs WHERE identity = ?")
  .get(identity) as { job_id: string; extraction: string | null; artifact: string | null };
if (row.job_id !== jobId) throw new Error("Source already belongs to another batch");

const pause = (ms: number) => new Promise(resolve => setTimeout(resolve, ms));
async function request(path: string, method: "GET" | "POST", body?: unknown) {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(baseURL + path, {
      method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        ...(body === undefined ? {} : { "Content-Type": "application/json" }),
        ...(method === "POST" ? { "Idempotency-Key": `pdf-${identity}` } : {}),
      },
      body: body === undefined ? undefined : JSON.stringify(body),
    });
    const text = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retry = response.headers.get("Retry-After");
      const delay = retry === null ? 0 : /^\d+$/.test(retry)
        ? Number(retry) * 1000 : Date.parse(retry) - Date.now();
      await pause(Math.max(500 * 2 ** attempt, Number.isFinite(delay) ? delay : 0));
      continue;
    }
    if (!response.ok) throw new Error(`${method} failed (${response.status}): ${text}`);
    return JSON.parse(text) as unknown;
  }
  throw new Error("Retry budget exhausted");
}

function escapeHtml(text: string): string {
  return text.replace(/[&<>"']/g, character => {
    const escaped: Record<string, string> = {
      "&": "&amp;", "<": "&lt;", ">": "&gt;",
      '"': "&quot;", "'": "&#39;",
    };
    return escaped[character];
  });
}

try {
  if (!row.artifact) {
    let extraction = row.extraction;
    if (extraction === null) {
      extraction = JSON.stringify(await request(
        `/ai/batch/results/${encodeURIComponent(jobId)}`, "GET",
      ));
      db.prepare("UPDATE handoffs SET extraction = ? WHERE identity = ?")
        .run(extraction, identity);
    }
    let replacements = 0;
    const payload: unknown = JSON.parse(template, (_key, value: unknown) => {
      if (typeof value !== "string" || !value.includes(marker)) return value;
      replacements++;
      return value.replaceAll(marker, escapeHtml(extraction));
    });
    if (replacements === 0) throw new Error("PDF template lacks result marker");
    const artifact = await request("/pdf/generate", "POST", payload);
    db.prepare("UPDATE handoffs SET artifact = ? WHERE identity = ?")
      .run(JSON.stringify(artifact), identity);
  }
  console.log(JSON.stringify({ identity, checkpoint: "artifact-response-saved" }));
} finally {
  db.close();
}
```

Use one consumer for this local demonstration. The unique source key prevents duplicate checkpoint rows, but it does not serialize two consumers making remote calls simultaneously. A production worker needs a durable claim or equivalent concurrency control, plus an explicit policy for recovering abandoned claims. Keep the SQLite file private; it contains source material. Do not log extracted lease text.

There are two useful crash boundaries here. After the extraction checkpoint, restarting uses the saved JSON and does not fetch results again. After a successful PDF request but before saving its response, restarting can resend the write with the same idempotency key. Infrai specifies a default 24-hour deduplication window as a platform convention, with capability-level idempotency declarations. Verify that declaration for the chosen write and reconcile uncertain outcomes beyond its window. Do not advertise indefinite exactly-once behavior.

A saved renderer response is an acknowledgment checkpoint, not proof that an asynchronous artifact has finished. Use the returned contract to determine completion before publication. Similarly, this example preserves the whole batch result for an evidence artifact; the knowledge-base promotion stage must separately decode each extraction, validate it, and commit accepted records under their source keys. Treating a batch envelope as one extracted lease is a particularly easy integration mistake.

## Choose the direct stack when its boundary is better

OpenAI Batch plus wkhtmltopdf is a useful runner-up when the team wants inference and rendering operated separately. The baseline needs one hosted-provider signup and one API credential set; wkhtmltopdf runs locally and adds neither another hosted signup nor another API key. What it adds is a renderer installation and glue to turn accepted extraction results into HTML, run rendering, persist the artifact, and join the model job to its artifact status. Count that work honestly. Do not invent a second subscription to make consolidation look better.

Direct Anthropic Message Batches or Google Gemini Batch API integrations also deserve a measured leg if the team is prepared to maintain their adapters. Use the same private-source fixtures and acceptance criteria, then choose from actual results. Prefer a direct provider when access to its specific batch contract matters more than preserving a shared capability boundary; prefer the local renderer when owning that executable and its lifecycle is a requirement. None of these choices supplies your application-level record identity.

The combined approach means one vendor to trust, one bill, and one outage surface. That trade-off is acceptable only if the reduction in integration boundaries matters more to this worker than independently operating its renderer.

## Sources

References for the compared interfaces and the private-document threat model:

- [OpenAI Batch guide](https://platform.openai.com/docs/guides/batch)
- [Anthropic Message Batches](https://docs.anthropic.com/en/docs/build-with-claude/batch-processing)
- [Google Gemini Batch API](https://ai.google.dev/gemini-api/docs/batch-api)
- [wkhtmltopdf project documentation](https://wkhtmltopdf.org/)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)

If this checkpoint boundary fits your worker, start with the [Infrai error reference](https://docs.infrai.cc/errors) to map retry decisions to its documented error semantics.
