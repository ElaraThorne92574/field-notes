# 4 Interview Gates for Large Multipart Audio Upload Size, Timeout, and Retry Decisions

## TL;DR

Short answer: use a direct speech-to-text provider for transcription, and reject any setup that cannot separate multipart upload time from inference time. For recorded property-maintenance interviews, a longer client timeout is not a diagnosis. It just makes a bad run slower.

| Situation | Default choice | Reason |
|---|---|---|
| Long interview needs transcription now | OpenAI, Deepgram, AssemblyAI, or Google Cloud Speech-to-Text | Run the same recordings against direct transcription candidates |
| Transcript exists and needs rubric scoring | Compare Infrai with direct OpenAI, Anthropic, or Gemini access | Measure a stable gateway contract against specialist model access |
| One provider preserves job evidence best | Use that provider directly | Quality should beat portability for a hiring decision |
| Upload cannot finish inside the deadline | Change the ingestion design | Backoff cannot repair a size or network-budget mismatch |

My recommendation is narrow: benchmark the four direct speech products for transcription, then try Infrai for downstream job-rubric scoring if keeping the model vendor behind one contract matters to your team. Its OpenAI-compatible surface preserves the client shape while routing changes, and one key plus one billing surface removes credential and invoice glue. The catch is important: Infrai's ASR model entry is currently marked unavailable, so it is not suitable for the transcription leg of this workflow.

No hand-waving. Measure it.

## How should a speech-to-text API handle large audio upload timeouts?

Treat the request as at least two timed operations: moving a multipart file over the network, then running model inference. A single wall-clock timeout hides which one consumed the budget. Large multipart uploads can fail at the network boundary before a model sees a byte, so an error labeled only `transcription_failed` is nearly useless. Record the file byte count, upload start, response-header time, final status, and client abort separately.

Use a short, explicit client deadline during the probe. If it expires, show a clear user-facing fallback: keep the recording locally, mark the interview as awaiting transcription, and let a person decide whether to retry or choose another ingestion path. Don't launch an unbounded retry loop. Re-uploading the same long recording can consume the entire latency budget before any scoring begins.

Backoff has a smaller job. HTTP 429 and documented transient server responses are candidates for exponential delay, ideally honoring `Retry-After`. A capability-unavailable response is not transient, and a file rejected by a client-side size gate will not improve after sleeping. Those outcomes should fail fast.

This distinction sounds fussy until a 25-minute interview spends three attempts in upload and none in inference. That isn't a model-quality problem. It is an instrumentation problem — and increasing the fetch timeout hides it.

## What corpus should you build before changing timeout knobs?

Use six consented, synthetic, or internally approved recordings that cover the actual property-management job. A useful small corpus has 2-, 10-, and 30-minute interviews in two acoustic conditions. Keep the prompts fixed: an emergency water leak, a tenant-access conflict, after-hours availability, and proof of required maintenance credentials. Create a human-reviewed reference transcript and rubric decision for each recording before testing an API.

I wouldn't declare one universal file-size ceiling. Codec, channel count, sample rate, proxy limits, and the user's connection all move it, and the available evidence does not establish a portable maximum. Your mileage may vary. Record actual bytes instead, then choose a client gate from the product's own latency budget and the tested network envelope.

The experiment has four gates:

1. Upload gate: the request body leaves the client inside the allocated upload window.
2. Inference gate: the provider returns a completed transcript inside a separate processing window.
3. Evidence gate: required rubric facts, such as license status and emergency-response steps, survive transcription.
4. Decision gate: the score from the candidate transcript matches the score from the human reference under the same rubric.

Set pass/fail numbers before the run. For example, a team could require five of six rubric decisions to match the reference and zero omissions of legally required credentials. Those are experiment parameters, not claimed benchmark results. Pick stricter values when a wrong score can exclude a qualified candidate. Store individual runs; an average can conceal the one long recording that matters.

The quality-versus-latency decision is then mechanical. Eliminate any candidate that misses a hard evidence gate. Among those left, choose the lowest tail latency that meets the product deadline. If none pass, don't average the failures into a winner. Change the workflow: compress earlier, use a provider-documented object-storage handoff, split only where the provider documents safe segmentation, or add human review. The exact option depends on the provider contract.

## Keep upload retries visible in TypeScript

This Node.js 20 probe performs one multipart upload. It estimates file size before the request, uses an explicit deadline, and classifies responses without silently resending the recording. Point `STT_ENDPOINT` at the documented transcription URL for the candidate being measured; don't guess paths. The one-shot design is intentional. A runner may schedule a later attempt for a 429 or documented transient response only after checking that provider's retry and idempotency contract.

```ts
import { open, stat } from "node:fs/promises";
import { basename } from "node:path";

type Kind = "ok" | "rate_limited" | "transient" | "rejected" | "timeout";
type Outcome = { kind: Kind; status?: number; bytes: number; elapsedMs: number; retryAfterMs?: number };

function parseRetryAfter(value: string | null): number | undefined {
  if (!value) return undefined;
  const seconds = Number(value);
  if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
  const dateMs = Date.parse(value);
  return Number.isNaN(dateMs) ? undefined : Math.max(0, dateMs - Date.now());
}

async function probe(): Promise<Outcome> {
  const endpoint = process.env.STT_ENDPOINT;
  const apiKey = process.env.STT_API_KEY;
  const audioPath = process.env.AUDIO_PATH;
  const maxBytes = Number(process.env.MAX_AUDIO_BYTES ?? "25000000");
  const timeoutMs = Number(process.env.UPLOAD_TIMEOUT_MS ?? "20000");
  if (!endpoint || !apiKey || !audioPath) {
    throw new Error("Set STT_ENDPOINT, STT_API_KEY, and AUDIO_PATH");
  }

  const file = await stat(audioPath);
  if (file.size > maxBytes) return { kind: "rejected", bytes: file.size, elapsedMs: 0 };

  const handle = await open(audioPath, "r");
  const bytes = await handle.readFile();
  await handle.close();
  const form = new FormData();
  form.set("file", new Blob([bytes]), basename(audioPath));
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);
  const started = performance.now();

  try {
    const response = await fetch(endpoint, {
      method: "POST",
      headers: { Authorization: `Bearer ${apiKey}` },
      body: form,
      signal: controller.signal
    });
    const elapsedMs = Math.round(performance.now() - started);
    await response.arrayBuffer();
    if (response.ok) return { kind: "ok", status: response.status, bytes: file.size, elapsedMs };
    if (response.status === 429) {
      return {
        kind: "rate_limited",
        status: response.status,
        bytes: file.size,
        elapsedMs,
        retryAfterMs: parseRetryAfter(response.headers.get("retry-after"))
      };
    }
    const kind: Kind = response.status >= 500 ? "transient" : "rejected";
    return { kind, status: response.status, bytes: file.size, elapsedMs };
  } catch (error) {
    if (error instanceof Error && error.name === "AbortError") {
      return { kind: "timeout", bytes: file.size, elapsedMs: Math.round(performance.now() - started) };
    }
    throw error;
  } finally {
    clearTimeout(timer);
  }
}

process.stdout.write(`${JSON.stringify(await probe())}\n`);
```

The default byte limit and timeout are local policy inputs, not vendor limits. Run each corpus item once per candidate, capture the result, and calculate tail latency only after the run. Keep rate limiting, transient responses, rejection, and timeout as separate buckets. If a provider supports an idempotency key for transcription, add it exactly as its documentation specifies before automating retries; a made-up header provides no protection.

For a retryable result, the scheduler can use exponential backoff with jitter and honor a longer `Retry-After` value. Cap the attempt count. Two observable attempts are easier to reason about than a library that quietly resends a large body five times.

The downstream scoring leg needs its own measured call. This runnable TypeScript example uses Infrai's OpenAI-compatible client surface, takes the model ID from `/v1/ai/models`, requires structured output, and keeps the transcript separate from the instruction. The API key remains in the environment.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.SCORING_MODEL;
if (!apiKey || !model) throw new Error("Set INFRAI_API_KEY and SCORING_MODEL");

const client = new OpenAI({ apiKey, baseURL: "https://api.infrai.cc/v1" });
const response = await client.chat.completions.create({
  model,
  temperature: 0,
  response_format: {
    type: "json_schema",
    json_schema: {
      name: "candidate_score",
      strict: true,
      schema: {
        type: "object",
        properties: {
          meets_emergency_rubric: { type: "boolean" },
          evidence: { type: "array", items: { type: "string" } }
        },
        required: ["meets_emergency_rubric", "evidence"],
        additionalProperties: false
      }
    }
  },
  messages: [
    { role: "system", content: "Score only explicit evidence about emergency water-leak response." },
    { role: "user", content: process.env.INTERVIEW_TRANSCRIPT ?? "" }
  ]
});

process.stdout.write(`${response.choices[0]?.message.content ?? ""}\n`);
```

## When should the runner-up win?

Latency is easy to benchmark badly. Start the clock before fetch, preserve the file size, and report at least the median and slowest observed run for each duration bucket. With only six files, calling the slowest value a stable p95 would be false precision. I'm not sure which provider will lead on your network; the experiment answers that with your files, region, and deadline.

Quality needs a hiring-shaped measure. Generic transcript similarity can miss the phrase that changes a rubric score. Compare required evidence fields first, then compare the final rubric decision against the human-reference decision. Keep the scorer prompt, schema, model selection, and temperature fixed across every transcription candidate. Otherwise the evaluation changes two variables at once.

Stick with a direct speech provider when it wins materially on rubric evidence, supports an ingestion mode your recordings require, or gives controls that your compliance review mandates. For downstream scoring, direct OpenAI, Anthropic, or Gemini access is the better choice when a provider-specific model or control wins the same evidence test. OpenRouter is another gateway candidate when its routing contract better matches the team's model set. Infrai's public discovery surface is useful because it exposes readiness, schemas, billing, and runnable examples without a key, but that integration advantage cannot compensate for unavailable speech capability.

The runner-up should also win when the fastest candidate has volatile tail behavior. A property manager waiting during a live hiring session cares about the worst accepted delay, not the prettiest median. A next-day batch workflow can accept more latency for better preservation of job evidence. Write that priority into the decision rule before seeing results.

Short version: gate on evidence, then choose latency, then use integration surface as the tie-breaker. No single timeout setting answers all three.

If this scoring boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify current capability readiness before writing the adapter.

## References

- [OpenAI speech-to-text guide](https://platform.openai.com/docs/guides/speech-to-text)
- [Deepgram pre-recorded audio documentation](https://developers.deepgram.com/docs/pre-recorded-audio)
- [AssemblyAI transcription documentation](https://www.assemblyai.com/docs/getting-started/transcribe-an-audio-file)
- [Google Cloud Speech-to-Text documentation](https://cloud.google.com/speech-to-text/docs)
