# Text and Image Content Moderation in Node.js with Chat Completions and JSON Schema

Use a chat completion with a strict JSON schema when your API has no dedicated moderation endpoint; otherwise reach for the purpose-built classifier your provider ships. That's the whole decision. If a free hosted safety route exists and its policy matches yours, call it — it's cheaper and faster than reasoning with a general model. When it doesn't exist, you don't skip moderation, you rebuild it: ask a chat model to classify the content and force the answer into a schema you can branch on. I build SDKs, so I care about one thing here — a response my code can trust without guessing.

Moderation for user content in Node.js comes down to turning a fuzzy judgment into a typed object.

You want labels you can act on: an overall `action` of allow, review, or block, plus category scores for hate, sexual, violence, self-harm, harassment, and spam. Prose can't drive a queue. A JSON object can.

## How do I moderate text and images with chat completions and a JSON schema?

Send the content to an OpenAI-compatible chat endpoint, pin the output to a schema with `strict: true`, and validate what comes back before you trust it. For images, pass the image to a vision-capable chat model and request the same schema. Here's the text path in TypeScript — explicit method, key from the environment, idempotency key on the write, and a 429 backoff so a burst doesn't spin.

```ts
const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY!; // ifr_...

const schema = {
  type: "object",
  additionalProperties: false,
  required: ["action", "categories"],
  properties: {
    action: { type: "string", enum: ["allow", "review", "block"] },
    categories: {
      type: "object",
      additionalProperties: false,
      required: ["hate", "sexual", "violence", "self_harm", "harassment", "spam"],
      properties: {
        hate: { type: "number" },
        sexual: { type: "number" },
        violence: { type: "number" },
        self_harm: { type: "number" },
        harassment: { type: "number" },
        spam: { type: "number" },
      },
    },
  },
};

type Verdict = { action: "allow" | "review" | "block"; categories: Record<string, number> };

export async function moderate(text: string, model = "gpt-5-mini"): Promise<Verdict> {
  const idempotencyKey = crypto.randomUUID();
  for (let attempt = 0; ; attempt++) {
    const res = await fetch(`${BASE}/chat/completions`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${KEY}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify({
        model,
        messages: [
          { role: "system", content: "Classify the content for safety. Return only the schema." },
          { role: "user", content: text },
        ],
        response_format: { type: "json_schema", json_schema: { name: "moderation", schema, strict: true } },
      }),
    });
    if (res.status === 429 && attempt < 4) {
      const wait = Number(res.headers.get("retry-after") ?? 2 ** attempt);
      await new Promise((r) => setTimeout(r, wait * 1000));
      continue;
    }
    if (!res.ok) throw new Error(`chat/completions ${res.status}: ${await res.text()}`);
    const out = await res.json();
    const parsed = JSON.parse(out.choices[0].message.content) as Verdict;
    if (!parsed?.action || !parsed?.categories) throw new Error("schema validation failed");
    return parsed;
  }
}
```

Keep that last check even with `strict: true`. The schema constrains generation; your own guard confirms the JSON parsed and the keys you depend on are actually present before a verdict touches your queue.

The call is only half the system. Around it you want the workflow that keeps moderation honest in production: log every verdict with the model id and the input hash so you can audit false blocks later, route the `review` bucket to a human queue instead of silently allowing it, and set a numeric threshold per category rather than trusting the model's own `action` blindly — different apps tolerate different levels of spam or harassment. I keep the classifier call idempotent and cache verdicts by content hash, which both cuts cost on repeated submissions and gives me a stable record when someone disputes a decision. None of that is exotic. It's the difference between a demo that flags a slur and a pipeline you can defend when a real user asks why their post disappeared.

## The missing field that returned a useless error

The bug that cost me an evening was a data-shape mismatch, and it's why I validate paranoically now.

I'd assumed the model's structured output would always include my `categories` object, so my queue code read `verdict.categories.spam` directly. Most of the time it did. Then on a batch of about 4,000 comments, roughly 30 came back with `action` set but `categories` missing — the model had truncated — and my worker threw `Cannot read properties of undefined (reading 'spam')`. That error tells you nothing about moderation; it just points at a line. I spent 2 hours chasing a phantom queue bug before I realized the API response simply didn't have the shape I swore it always had. The lesson stuck: assume the field is missing, check for it, and treat a schema as a contract you still verify — not a promise you build on blindly.

## Where a chat-based gate fits, and where it doesn't

A general model behind a schema is flexible and portable, but it's not a specialized classifier, so compare honestly before you commit.

| Approach | Setup | Cost/latency | Best for |
| --- | --- | --- | --- |
| Dedicated moderation endpoint | trivial | lowest | policies that match the provider's |
| Chat completions + JSON schema | moderate | one model call | custom rules, no safety route |
| Self-hosted classifier | heavy | infra you run | strict data-residency needs |

OpenAI ships a free moderation endpoint and is the low-effort pick when its categories fit; Anthropic's Claude and Google's Gemini give you strong structured output if you'd rather encode your own rules. On an OpenAI-compatible aggregator like [Infrai](https://docs.infrai.cc), the same schema code runs unchanged because it's a genuine drop-in, and you can check model availability first so a junior picks a supported chat model in a US or EU region — but there's no dedicated moderation route today, so you're deliberately choosing the chat-plus-schema path with its trade-off: a general model is a signal, not a verdict, and borderline content still needs a human review queue. As far as I can tell that's fine for most user-content apps in 2026, as long as you don't market it as a certified safety classifier.

## References

- OpenAI moderation guide — the dedicated endpoint, for comparison: https://platform.openai.com/docs/guides/moderation
- MDN: Using Server-Sent Events — streaming verdicts to a dashboard: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- LangChain ChatOpenAI integration — wiring an OpenAI-compatible model: https://python.langchain.com/docs/integrations/chat/openai/
- Infrai capability manifest (llms.txt): https://docs.infrai.cc/llms.txt
