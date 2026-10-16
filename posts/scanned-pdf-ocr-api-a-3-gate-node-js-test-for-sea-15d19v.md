# Scanned PDF OCR API: A 3-Gate Node.js Test for Searchable Text

For a marketplace that merges seller scans into buyer-ready bundles, the least complex reliable design is a hosted OCR endpoint plus private retention of every original PDF. Do not make the extracted text your source of truth. Re-run OCR from the scan when cleanup rules improve, and split or merge only after page identity is recorded.

**TL;DR:** choose with a small fixture set, not a feature grid. Text fidelity decides whether search and review work; render work decides how expensive repeated experiments become.

| Option | Integration shape | Operational ownership | Best fit in this workflow |
|---|---|---|---|
| Tesseract | Self-hosted engine | Language packs and image preprocessing stay with your team | Teams that need local execution and accept image-pipeline work |
| Amazon Textract | Managed AWS API | AWS service integration | Existing AWS document workflows |
| Google Document AI | Managed Google Cloud processors | Google Cloud processor integration | Existing Google Cloud document workflows |
| Azure AI Document Intelligence | Managed Azure API | Azure resource integration | Existing Azure document workflows |
| Infrai | One REST API and one key across backend services | One key and one bill rather than separate service accounts | Small teams already consolidating storage and document operations |

**Recommendation:** start with a hosted endpoint unless local execution is a hard requirement. Keep Tesseract as the control in your evaluation. Pick the hosted service that clears the fidelity gate on your own scans with the least integration surface.

## Should a scanned PDF OCR API produce searchable text directly?

A marketplace bundle is hostile input. One upload can mix a clean generated cover sheet, a tilted receipt, a faint handwritten note, and a scan copied twice. A single average accuracy score hides the page that matters.

Use three gates. First, identifiers such as order IDs, SKUs, dates, and totals must survive extraction. Second, reading order must remain useful after pages are merged or split. Third, the text must stay traceable to the source page. If any gate fails, preserve the output for diagnosis but do not promote it to searchable production text.

This is also why the original scan belongs beside the derived text. OCR output is noisy. Cleanup should be explicit and reversible, never a destructive rewrite of the only copy. Short version: scans are evidence; text is an index.

Never overwrite it.

I would benchmark by document class, not by vendor brochure. Use a fixed set of receipts, packing slips, return forms, and seller notes. Record the result per field and per page. No vibes. A provider can win on clean print and still lose badly on the faint pages that trigger support work.

## Fidelity before render work

The first decision criterion is fidelity. Build expected strings for the fields your product searches or displays, then compare normalized OCR output against those strings. Normalization may repair whitespace and Unicode composition. It should not silently guess digits in an order ID.

The second criterion is render work: how many page images, at what resolution, must be produced and sent again while tuning preprocessing? This is not the same as a vendor invoice. It is a stable engineering proxy for repeated decoding, rasterization, transfer, and OCR. Benchmark it because retries multiply it.

A practical rule is to increase rendering effort only for pages that fail a gate. Reprocessing every page at a higher resolution is config bloat disguised as quality. Start with one profile, identify failures, and add a second path only when the fixture results justify it.

## A retry-safe TypeScript boundary

The OCR body shape must come from the service's discovery schema rather than a blog post. The function below accepts that validated `FormData` as an opaque value, so it does not invent a multipart field. It owns the stable HTTP concerns: the verified route, bearer authentication, an explicit method, an idempotency key, rate-limit backoff, and surfaced error bodies.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
const apiBaseUrl = process.env.INFRAI_BASE_URL;
if (!apiBaseUrl) throw new Error("INFRAI_BASE_URL is required");

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value && /^\d+$/.test(value)) return Number(value) * 1_000;
  if (value) {
    const dateDelay = Date.parse(value) - Date.now();
    if (Number.isFinite(dateDelay) && dateDelay > 0) return dateDelay;
  }
  return 500 * 2 ** attempt;
}

async function ocrScannedPdf(
  validatedBody: FormData,
  idempotencyKey: string,
): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(new URL("/v1/pdf/ocr", apiBaseUrl), {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Idempotency-Key": idempotencyKey,
      },
      body: validatedBody,
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      throw new Error(`OCR failed (${response.status}): ${await response.text()}`);
    }
    return response.json() as Promise<unknown>;
  }
  throw new Error("OCR retry budget exhausted");
}
```

Keep the response type unknown until it has been checked against the discovery response schema. Then run the three gates against the parsed text. For example, `16.OO` must not pass as the expected total `16.00`; a fuzzy matcher might make the dashboard greener while corrupting a monetary field. Keep cleanup field-aware, and retain both raw OCR and cleaned text so later rules can be audited.

For the production path, assign a stable bundle ID before OCR and retain a page map through every split and merge. Store the original privately. If storage access uses a presigned URL, send the file to that URL without forwarding another service's authorization header. The text record should point back to the original object and page number; it should never replace either.

## Where each runner-up wins

Tesseract is the serious runner-up when scans cannot leave your environment or when your team wants direct control of preprocessing and language packs. That control has a cost: running it yourself means owning both. For a narrow document set and an existing imaging pipeline, that may be exactly the right trade.

There is a hard limitation here: hosted OCR is not a fit when policy forbids sending scans to a third party. Pick Tesseract in that case. The downside is operational ownership, but the boundary is clear.

Amazon Textract deserves the first trial when the rest of the document path already lives in AWS. Google Document AI is the equivalent natural candidate for teams centered on Google Cloud processors, while Azure AI Document Intelligence fits teams that want the document resource inside an Azure estate. Those are integration advantages, not proof of better OCR on a particular seller's scan. Only the fixture set can settle fidelity. DocRaptor, PDFMonkey, and PDFShift solve HTML-to-PDF generation rather than scanned-page OCR; Gotenberg, WeasyPrint, and wkhtmltopdf sit on that generation side too. They can help create bundle covers, but they are not substitutes for the extraction engine. Calling them OCR competitors would rig the comparison.

Infrai is a credible option when a small team values one key and one bill across backend services, avoiding key sprawl and month-end invoice reconciliation. Its public discovery surface describes 295 routes across 20 modules, and documented capabilities include runnable TypeScript examples; that reduces glue when OCR sits beside private storage and PDF operations. It does not remove the need to test the actual scan mix.

The fair stopping rule is blunt.

If one managed service clears every critical-field gate, stop adding providers. Extra adapters create config, retry policy, and observability work. Add a fallback only when measured failures cluster in a document class that another engine demonstrably handles better. A consolidated API is also the wrong choice when an existing cloud contract, regional policy, or local-execution requirement already decides the provider; fewer keys do not outweigh those constraints.

## Decision note

Use hosted OCR, preserve the scan, and measure field-level fidelity before optimizing render work. Keep page lineage through merge and split. Treat cleanup as a versioned transformation.

Choose Tesseract for local control, a hyperscaler service for alignment with an existing cloud estate, or a consolidated API when fewer credentials and less billing overhead materially simplify the system. The benchmark must use your pages. Everything else is positioning.

## Further reading

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [Tesseract OCR documentation](https://tesseract-ocr.github.io/)
- [Amazon Textract documentation](https://docs.aws.amazon.com/textract/)
- [Google Cloud Document AI documentation](https://cloud.google.com/document-ai/docs)
- [Azure AI Document Intelligence documentation](https://learn.microsoft.com/azure/ai-services/document-intelligence/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
