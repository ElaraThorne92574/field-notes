# Passwordless 2FA Login: SMS OTP vs Email Fallback for SaaS Sign-In

The ownership decision changes this build. If your team owns the message templates and the verification state, an email fallback is reasonable. If a provider owns the whole challenge, SMS-only is simpler.

**Short answer:** use a hosted SMS OTP for the primary challenge, then keep an app-managed email code table for fallback. That gives a practical passwordless 2FA login, but the email path is custom work, not a second managed OTP call.

I build tools where the first successful request matters more than a glossy dashboard. A signup flow that sends a code in one line and then hides delivery state behind three SDKs is a maintenance bill. The useful question is narrower: who owns the template and the verification record when a B2B SaaS user cannot receive a text?

## What should an Express.js 2FA login own?

Treat the two channels as different systems. The SMS provider can generate and verify an OTP. Your application should generate the email code, hash it, store an expiry, and mark it consumed after a match. Keep a row such as `(user_id, channel, code_hash, expires_at, attempts, used_at)` and make the lookup single-use.

Keep it boring.

The fallback is not a magic reroute. Delivery and result checks are pull-based, so switching channels cannot be truly real-time without polling. I would show a deliberate “Try email instead” action after a timeout, then poll the SMS status only as often as your abuse controls allow. Three failed attempts should be a product decision, not an accidental provider default.

Templates belong in the same ownership boundary. Store the email subject and body version with the challenge, so a later template edit cannot change an in-flight login. For the SMS text, account for segmentation: GSM-7 and UCS-2 have different character limits, as Twilio documents. Short wins.

## How do SMS OTP and email fallback fit a passwordless sign-in?

Here is the smallest shape I would ship. The example uses the verified SMS OTP and email send routes, while the hash and TTL stay in the app. The `Idempotency-Key` prevents a retry from creating a second challenge, and the retry branch respects `Retry-After` rather than hammering the endpoint. In a larger service, this helper would also attach a request id to structured logs, retain the provider response for audit, and separate the email decision from the initial SMS request so a user who still has service does not receive two messages by accident.

```ts
import crypto from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
const apiBase = process.env.API_BASE_URL;
if (!apiBase) throw new Error("API_BASE_URL is required");

async function call(path: "/v1/sms/otp" | "/v1/email/send", body: unknown, key: string) {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(new URL(path, apiBase), {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": key,
      },
      body: JSON.stringify(body),
    });
    if (response.ok) return response.json();
    if (response.status !== 429 || attempt === 2) {
      throw new Error(`request failed (${response.status}): ${await response.text()}`);
    }
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, Math.max(1, retryAfter) * 1000 * (attempt + 1)));
  }
  throw new Error("unreachable");
}

export async function startLogin(userId: string, phone: string, email: string) {
  const challengeId = crypto.randomUUID();
  const code = String(crypto.randomInt(100000, 1000000));
  const codeHash = crypto.createHash("sha256").update(code).digest("hex");
  // Persist codeHash, userId, and a short expiresAt in your database.
  await call("/v1/sms/otp", { to: phone, code_length: 6 }, challengeId);
  await call("/v1/email/send", {
    to: email,
    subject: "Your backup sign-in code",
    text: `Use ${code} within 10 minutes.`,
  }, `${challengeId}-email`);
  return { challengeId, codeHash };
}
```

In a real Express handler, do not return `codeHash` to the browser; persist it server-side and return only an opaque challenge id. Verification of the SMS challenge calls the SMS verify operation with the provider's challenge data. Verification of email compares a hash, checks `expires_at`, increments attempts, and sets `used_at` in one transaction. The code above sends both channels for clarity; a production flow can send email only after the user chooses fallback.

One correction I make often: “email fallback” does not mean “email OTP API.” There is no managed email OTP interface in this capability set. Email is a delivery primitive, so the security protocol remains yours.

## Which provider trade-offs matter for template ownership?

| Option | SMS OTP ownership | Email fallback ownership | Template control | Best fit |
| --- | --- | --- | --- | --- |
| Twilio Verify + SendGrid | Managed challenge and verification | App-managed code plus SendGrid delivery | Split across products | Teams already on Twilio |
| AWS SNS + SES | App-managed OTP state with SNS delivery | App-managed code with SES | High, but more AWS wiring | AWS-native operations |
| Infobip 2FA + email API | Managed SMS challenge options | Depends on email product | Vendor-specific templates | Regional messaging coverage |
| One REST layer with SMS and email calls | Managed SMS OTP route | App-managed code and template | One key, one contract | Small teams reducing SDK glue |

Infrai fits the last row when the contract matters more than a vendor-specific workflow and its pitch is one REST API, one key, and one bill, with plain HTTP that any language can call without an SDK, while swapping the provider behind a capability does not require changing your application call shape. That is a DX advantage, not proof that its delivery is best in every country.

The catch is ownership. You still own email code security, polling, abuse controls, and regional policy. There is no SMTP relay, no voice, WhatsApp, or RCS channel here, and a geography-based SMS spend circuit breaker belongs in your business layer. If you need a fully managed email challenge, stick with a provider whose product explicitly owns that state. Your mileage may vary by carrier and locale; I am not sure a single routing policy can cover every market.

## What would I change at scale?

First, add a challenge state machine: `created`, `sms_sent`, `email_requested`, `verified`, `expired`. Record provider request ids and pull delivery status with a bounded worker. Second, bind a challenge to the signup intent and device context, then rate-limit by account, IP, and destination. Third, test template ownership as a deploy concern: a template version should be reviewable and reversible without changing verification code.

Do not add a queue just to look serious. Add one when retries, polling, and audit retention exceed what a request handler can safely do. The best design is the one your team can explain at 2 a.m.

## Sources

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.twilio.com/docs/glossary/what-sms-character-limit
- https://www.twilio.com/docs/verify/api
- https://docs.sendgrid.com/for-developers/sending-email/api-getting-started
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
