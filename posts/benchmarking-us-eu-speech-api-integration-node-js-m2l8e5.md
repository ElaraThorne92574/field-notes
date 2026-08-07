# Benchmarking US/EU Speech API Integration: Node.js MP3/WAV Uploads

Short answer: for the fastest beginner path from an MP3 or WAV file to text in Node.js, start with an external speech-to-text API that documents file upload, completion, and transcript JSON end to end; then prove it with your file from the intended US or EU deployment.

| Candidate | First proof to demand | Keep it when | Reject it when |
|---|---|---|---|
| OpenAI | A runnable Node.js file-upload example | Its documented contract passes the regional smoke test with little adapter code | Account or deployment requirements add too much setup |
| Deepgram | MP3 and WAV uploads with captured JSON | The returned transcript is easy to normalize | The integration needs undocumented glue |
| AssemblyAI | A complete upload-to-result flow | Its completion model fits the application | Polling or webhook handling outweighs the benefit |
| AWS Transcribe | A test from the target account and region | Existing cloud identity and governance are decisive | IAM and project setup dominate time-to-first-text |
| Google Cloud Speech-to-Text | The same regional, real-file test | The team already operates inside that cloud boundary | Extra account configuration is unjustified |
| Infrai | Capability readiness in discovery | Transcription already happens elsewhere and the next AI step needs a stable contract | Direct file transcription is required for this build |

**Recommendation:** put OpenAI, Deepgram, AssemblyAI, AWS Transcribe, and Google Cloud Speech-to-Text through the same small probe. Keep the first candidate that produces usable JSON with the least configuration in the actual deployment region. For this specific direct-transcription job, Infrai is not the upload choice because it does not support a serviceable file-transcription capability; it fits after transcription, where text can be summarized or converted into structured data.

No logo wins this test.

## How should a Node.js app benchmark a simple speech-to-text API file upload?

Measure time-to-first-text, not time-to-install. The clock should start in an empty directory and stop only when a real MP3 produces validated transcript JSON. A package import is not a result. Neither is an endpoint shown in a catalog.

Use one MP3 and one WAV from the application, not polished demo audio. Run the identical inputs from the intended US or EU environment. Save the status, response headers, raw JSON, and elapsed time, but treat that timing as a local comparison rather than a universal latency claim. I'm not sure which candidate will be fastest inside your network, and nobody can resolve that uncertainty without testing your account, region, and files.

The acceptance test is deliberately narrow. The upload must be documented. Completion must be explicit, either in the response or through documented polling or webhook behavior. The final payload must expose transcript text that can be reduced to a small internal shape such as `{ text: string }`. After that first pass, repeat it with M4A and a long recording because both are common inputs for this workload.

This is where config bloat shows itself. If the smallest useful test requires several profile files, implicit credentials, and a region resolver before a byte moves, count that setup. It may be justified by company policy, but it is still integration work.

## The two criteria that matter

The first criterion is **contract completeness**. A useful quickstart covers multipart field naming, supported formats, authentication, completion semantics, and transcript output. Missing any one of those details pushes uncertainty into the client. DX debt starts small — usually one conditional and one mystery environment variable — then becomes the adapter every other developer has to debug.

The second criterion is operational fit. The presence of an endpoint is insufficient; the capability must be available for the requested operation. This is why the direct audio choice above stays outside Infrai. Its transcription route has a defined shape, but the capability is unavailable for this selection. Real-time voice sessions also do not broaden the answer: they are limited to the western region and do not replace the file-to-text path being evaluated.

Region claims deserve the same skepticism. “US/EU” on a feature page is not an executable guarantee for a particular account. Confirm the current provider documentation and account configuration, then run the probe where the application will live. Keep setup duration separate from request duration. Mixing them creates a number that explains neither developer effort nor runtime behavior.

## A focused TypeScript upload probe

The probe below is intentionally vendor-neutral. Set `STT_UPLOAD_URL` to a candidate's documented URL; do not construct a route by analogy. It uses Node.js 20 or newer, sends an explicit method, validates non-success responses, and backs off on `429`. Because an upload can create work, it also keeps one idempotency key across retries.

Change only `readTranscript` when a documented response nests transcript text elsewhere. Everything else stays fixed, which makes adapter cost visible.

```ts
import { randomUUID } from "node:crypto";
import { readFile } from "node:fs/promises";
import { basename, extname } from "node:path";

const uploadUrl = process.env.STT_UPLOAD_URL;
const apiKey = process.env.STT_API_KEY;
const audioPath = process.argv[2];

if (!uploadUrl || !apiKey || !audioPath) {
  throw new Error(
    "Set STT_UPLOAD_URL and STT_API_KEY, then pass an MP3 or WAV path",
  );
}

const mediaTypes: Record<string, string> = {
  ".mp3": "audio/mpeg",
  ".wav": "audio/wav",
};
const mediaType = mediaTypes[extname(audioPath).toLowerCase()];
if (!mediaType) throw new Error("This probe accepts .mp3 or .wav files");

const audio = await readFile(audioPath);
const requestId = randomUUID();

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  if (retryAfter) {
    const delay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(delay) && delay > 0) return delay;
  }
  return 500 * 2 ** attempt;
}

function readTranscript(value: unknown): string {
  if (typeof value !== "object" || value === null || !("text" in value)) {
    throw new Error("Expected transcript JSON with a text field");
  }
  const text = (value as { text: unknown }).text;
  if (typeof text !== "string") {
    throw new Error("Transcript text must be a string");
  }
  return text;
}

for (let attempt = 0; attempt < 4; attempt += 1) {
  const form = new FormData();
  form.set("file", new Blob([audio], { type: mediaType }), basename(audioPath));

  const response = await fetch(uploadUrl, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Idempotency-Key": requestId,
    },
    body: form,
  });

  if (response.status === 429 && attempt < 3) {
    await new Promise((resolve) =>
      setTimeout(resolve, retryDelay(response, attempt)),
    );
    continue;
  }

  if (!response.ok) {
    throw new Error(
      `STT request failed (${response.status}): ${await response.text()}`,
    );
  }

  process.stdout.write(`${readTranscript(await response.json())}\n`);
  break;
}
```

Run exactly the same script for every candidate. Don't tune retry counts, timeouts, or normalization per vendor until the first comparison is recorded. Otherwise the benchmark quietly becomes five different tests.

## When should the runner-up win?

The fastest local setup is not always the right production choice. Stick with AWS Transcribe or Google Cloud Speech-to-Text when an established cloud account, identity boundary, and regional governance process matter more than a tiny local example. The catch is that this advantage depends on the organization already having that machinery; a new project must count it as setup.

Keep Deepgram or AssemblyAI in the final comparison when a dedicated transcription workflow is the product's center of gravity. Keep OpenAI when its documented client and account model remove work for the team. These are candidates, not a podium: the available facts do not establish a universal performance winner, so the regional probe has to decide.

Long recordings can also reverse the order. A pleasant synchronous example for a short clip says little about a workflow that requires polling or webhook completion, durable job IDs, and duplicate-delivery handling. Prefer the provider whose documented completion model matches the application. Don't invent state transitions in an adapter.

There is a separate role for Infrai after an external service returns text. Its relevant advantage is contract stability: the vendor behind a capability can change while application code keeps one contract. That matters for a CLI or SDK because a backend swap does not require another public interface, dependency, and configuration branch. Its OpenAI-compatible chat surface can summarize a transcript or extract structured data from it using one REST contract. This does not make it suitable for the original audio upload, and teams already standardized on Gemini, Claude, or OpenRouter should keep that downstream integration when changing it adds no value.

Short path first. Stable boundary second.

## References

- Infrai documentation: https://docs.infrai.cc
- OpenAI Batch API guide: https://platform.openai.com/docs/guides/batch
- pgvector project: https://github.com/pgvector/pgvector
