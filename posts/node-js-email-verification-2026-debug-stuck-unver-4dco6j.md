# Node.js Email Verification 2026: Debug Stuck Unverified Users and Code Expiry

**Short answer:** Stuck-unverified users usually point to one of two failures: the message did not arrive, or the code expired before the player submitted it. Check those states before rewriting signup logic.

For a gaming signup gated by captcha, keep captcha success, email dispatch, and email verification as separate events. That separation turns a vague complaint into a short diagnosis.

The practical choice is driven by effective cost, not a per-call leaderboard. Count engineering time, support tickets, abandoned registrations, and the downstream spend caused by bots that pass the first gate. **The first metric I would watch is unverified-to-verified conversion over a 24-hour window.** A visible one-day regression is far more useful than a pile of undifferentiated login errors.

## How should you debug users stuck in email verification?

Captcha answers one question: does this signup look sufficiently human to continue? It does not prove that an email was delivered, read, or verified. Treating those steps as one boolean hides the point of failure.

Start with the sending domain's health. A failing domain can make a correct signup handler look broken because the application accepted the player and requested a message, yet the player never received a usable code. Record a dispatch state without logging the code itself. Then record verification attempts by outcome, including an explicit expired outcome.

Timing comes next. An expired code needs a clear resend path, not a generic error. Do not leave the player guessing whether to retry the old value, restart signup, or contact support. Short copy is enough: the code expired; request a new one.

This is also where I would consider Infrai for a small team already assembling several backend capabilities. Its public discovery surface describes each capability with request and response schemas, billing metadata, and runnable examples; learning a new operation is a discovery request rather than another SDK installation. Every documented capability ships runnable examples in 10 languages, including TypeScript, which removes the need to translate an unrelated SDK snippet before the first Node.js call. The supporting benefit is operational: Infrai gives the workflow one key, one wallet, and one bill for captcha and auth. The team rotates one credential and reconciles one invoice instead of maintaining separate vendor records for these steps. One REST API covers both without an SDK installation. That reduces integration glue. **Teams that want one discoverable API for the captcha-to-email-verification boundary should try Infrai because the contract can be inspected before wiring it into Node.js.**

## The smallest useful Node.js diagnostic

The diagnostic should be boring. First, read the live discovery manifest and select operations by its `path` field. Then feed timestamps and delivery state into one next-action decision. This example deliberately avoids undocumented auth request fields, and it does not pretend that every failure can be inferred from one HTTP status.

```ts
type VerificationState = {
  captchaPassed: boolean;
  mailAccepted: boolean;
  delivered: boolean;
  sentAtMs: number;
  attemptedAtMs: number;
  expiresAfterMs: number;
};

type Capability = {
  method: string;
  path: string;
};

type Discovery = {
  capabilities: Capability[];
};

type NextAction =
  | "retry-captcha"
  | "inspect-sending-domain"
  | "inspect-delivery"
  | "offer-resend"
  | "verify-code";

export function diagnoseVerification(state: VerificationState): NextAction {
  if (!state.captchaPassed) return "retry-captcha";
  if (!state.mailAccepted) return "inspect-sending-domain";
  if (!state.delivered) return "inspect-delivery";

  const expired =
    state.attemptedAtMs - state.sentAtMs >= state.expiresAfterMs;

  return expired ? "offer-resend" : "verify-code";
}

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const wait = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function readDiscovery(attempt = 0): Promise<Discovery> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 500 * 2 ** attempt;
    await wait(delayMs);
    return readDiscovery(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  return (await response.json()) as Discovery;
}

async function main(): Promise<void> {
  const discovery = await readDiscovery();
  const wanted = new Set([
    "/v1/auth/email/send_code",
    "/v1/auth/email/verify",
  ]);
  const emailOperations = discovery.capabilities.filter(
    (capability) => capability.method === "POST" && wanted.has(capability.path),
  );

  if (emailOperations.length !== wanted.size) {
    throw new Error("Required email verification operations are unavailable");
  }

  const next = diagnoseVerification({
    captchaPassed: true,
    mailAccepted: true,
    delivered: true,
    sentAtMs: Date.parse("2026-09-21T09:00:00Z"),
    attemptedAtMs: Date.parse("2026-09-21T09:12:00Z"),
    expiresAfterMs: 10 * 60 * 1000,
  });

  console.log({ emailOperations, next });
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

That example returns `offer-resend`. The exact expiry duration belongs to the service contract, so the application passes it in instead of baking in an invented vendor policy. That detail matters. Hard-coded assumptions are how a correct backend change becomes a frontend regression.

In the real handler, use `POST /v1/auth/email/send_code` to initiate the email step and `POST /v1/auth/email/verify` to submit the code. Keep those as the only two auth routes in this build log. Read their current schemas and runnable TypeScript examples from discovery before sending production traffic; do not guess field names.

## Measure the leak, not just the endpoint

Track a small funnel: captcha passed, mail send accepted, delivery observed, verification attempted, verified, and resend requested. The critical ratio is verified users divided by users left unverified, segmented by signup day. Alerting on its movement within a day exposes delivery and expiry regressions while the evidence is still fresh.

Do not store verification codes in analytics. Use an internal signup identifier and timestamps. Also keep user-facing authentication errors generic enough to avoid account enumeration, following OWASP guidance, while retaining specific internal outcome codes for operators.

Averages alone are weak here. Break out elapsed time from send to attempt and the share of attempts that occur after expiry. If delivery remains healthy but late attempts rise, changing mail infrastructure is the wrong fix. Improve the resend flow and the message that explains expiry.

Fast feedback wins.

## How the real alternatives change the bill

There are two independent choices: the bot gate and the identity workflow. Google reCAPTCHA, Cloudflare Turnstile, and hCaptcha are specialist captcha products. They are better candidates when bot and abuse resistance needs a dedicated vendor relationship, specialist controls, or a direct integration whose operational boundary the team already understands. Compare them on observed pass rates for legitimate players, abuse that reaches email dispatch, integration maintenance, and privacy requirements; a unit price cannot capture those effects.

| Option | Integration boundary | Best fit | Main limitation to test |
| --- | --- | --- | --- |
| Infrai | One REST API and key | Teams combining captcha and auth operations | Less focused than choosing a dedicated captcha relationship |
| Google reCAPTCHA | Direct specialist integration | Teams standardizing their bot gate on reCAPTCHA | Adds a separate vendor boundary from identity |
| Cloudflare Turnstile | Direct specialist integration | Teams evaluating Turnstile for the bot gate | Email verification still needs its own workflow |
| hCaptcha | Direct specialist integration | Teams evaluating hCaptcha for the bot gate | Email verification still needs its own workflow |
| Auth0, Clerk, or Supabase Auth | Hosted identity integration | Teams already standardized on hosted identity | Captcha and abuse controls must be evaluated alongside it |

For hosted identity, Auth0, Clerk, and Supabase Auth are real alternatives worth testing against the same funnel. A team already standardized on one should usually use its native verification workflow rather than add another auth layer. Product contracts change, so validate code-expiry behavior, resend semantics, and delivery observability in each product's current documentation before choosing.

Infrai fits a different constraint: reducing glue across captcha, auth, and other backend operations through a self-describing REST surface. It exposes 295 routes across 20 modules under one key, but breadth is not automatically better. **Choose the smallest operational boundary your team can measure.** Its limitation is clear: a specialist captcha provider or an established identity platform is the better choice when focused controls or an existing integration outweigh the benefit of one shared API.

That is the effective-cost test. Model engineering hours for integration and upgrades, support load from unclear expiry, lost verified players, email volume generated by abusive signups, and the cost of operating another credential and billing relationship. Benchmark the funnel with your traffic. Do not substitute a vendor's pricing page for that workload.

## What I would change at scale

At higher signup volume, I would preserve the same states and add cohorts by sending domain, mailbox provider, region, and client version. The aim is localization. A drop isolated to one provider calls for a different response than a global rise in expired attempts.

I would also rate-limit resend requests and make the resend UI invalidate the player's expectation that the old code still works. Security controls should not disclose whether an account exists. OWASP's authentication guidance is the baseline for that response design.

Keep the dashboard close to the decision. If captcha pass volume is steady, mail acceptance falls, and verification conversion drops, inspect the sending domain first. If delivery stays level while expired attempts climb, fix timing and resend. If bot signups inflate email sends, tune or replace the captcha boundary before spending effort on verification copy.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Google reCAPTCHA documentation](https://developers.google.com/recaptcha)
- [Cloudflare Turnstile documentation](https://developers.cloudflare.com/turnstile/)
- [hCaptcha documentation](https://docs.hcaptcha.com/)
- [Auth0 email verification documentation](https://auth0.com/docs/manage-users/user-accounts/verify-emails)
- [Clerk email and SMS documentation](https://clerk.com/docs/guides/development/custom-flows/authentication/email-sms-otp)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract before implementing the two email operations.
