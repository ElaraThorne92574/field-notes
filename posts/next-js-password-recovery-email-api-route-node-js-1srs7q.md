# Next.js Password Recovery Email API Route — Node.js HTML and Deliverability

Use one application-owned email contract for password resets and post-payment receipts, and keep provider details behind the adapter. The deciding constraint is integration effort: token creation, suppression policy, template data, idempotency, and delivery inspection belong in the backend even when a vendor handles transport.

TL;DR: In a Next.js or Node.js backend, create the reset token in your app, check suppression before sending, preview the custom HTML during development, and poll delivery events when debugging. Do not make the email provider your source of truth for payment state or token validity.

## How should a Next.js password reset email API route work?

A receipt after payment settles and a password-reset message look like two tiny API calls. They are not the same workflow, but they expose the same useful boundary: the application decides whether a message should exist; the delivery service accepts a template plus data. That boundary keeps a provider swap from leaking through every route. It also keeps the payment webhook from owning HTML and the account route from owning vendor response shapes.

The password-reset path has a sharper security rule. Return the same public response for known and unknown addresses, mint a short-lived single-use token, store only a digest, and never put account existence into an error message. Those are application concerns. Provider suppression is a second gate, not authorization.

Receipts add a different constraint: retries are normal after payment settlement. Give each logical message a stable key such as `receipt:<paymentId>`. Without that, a retried handler can send two receipts even though the charge settled once.

Small boundary. Big payoff.

No SDK required.

## The smallest working Node.js boundary

The route below is intentionally vendor-neutral. It uses Node crypto, takes dependencies explicitly, and contains no vendor SDK types. The same `TransactionalEmail` interface can be backed by Resend, Postmark, Amazon SES, SendGrid, or a plain REST service. Every implementation must translate the stable `idempotencyKey` into the provider's supported deduplication mechanism or persist the key before making the call.

Infrai's advantage here is one REST API with one API key, so the application contract stays put when the vendor behind the capability changes. `payload` is deliberately typed as `unknown`: obtain and validate its current JSON Schema from public discovery rather than freezing undocumented fields into an article. The function itself is runnable. It uses the verified send path, keeps the key in an environment variable, sends an idempotency key, surfaces non-success bodies, and honors `Retry-After` on `429` before exponential backoff.

```ts
const sleep = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

export async function sendEmail(
  payload: unknown,
  idempotencyKey: string
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  const apiOrigin = process.env.INFRAI_API_ORIGIN;
  if (!apiOrigin) throw new Error("INFRAI_API_ORIGIN is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(new URL("/v1/email/send", apiOrigin), {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey
      },
      body: JSON.stringify(payload)
    });

    if (response.ok) return response.json() as Promise<unknown>;

    const detail = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Email send failed (${response.status}): ${detail}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 500 * 2 ** attempt;
    await sleep(delayMs);
  }

  throw new Error("Email send retry budget exhausted");
}
```

That is the whole transport.

```ts
import { NextRequest, NextResponse } from "next/server";
import { createHash, randomBytes } from "node:crypto";

type ResetUser = { id: string; email: string; displayName: string };

type ResetStore = {
  findUserByEmail(email: string): Promise<ResetUser | null>;
  saveToken(input: {
    userId: string;
    digest: string;
    expiresAt: Date;
  }): Promise<void>;
};

type TransactionalEmail = {
  isSuppressed(email: string): Promise<boolean>;
  sendTemplate(input: {
    to: string;
    template: "password-reset";
    data: { displayName: string; resetUrl: string };
    idempotencyKey: string;
  }): Promise<{ messageId: string }>;
};

const normalizeEmail = (value: string): string => value.trim().toLowerCase();

export function buildResetHandler(store: ResetStore, email: TransactionalEmail) {
  return async function POST(request: NextRequest): Promise<NextResponse> {
    const body = (await request.json()) as { email?: unknown };
    if (typeof body.email !== "string") {
      return NextResponse.json({ error: "A valid email is required" }, { status: 400 });
    }

    const address = normalizeEmail(body.email);
    const accepted = NextResponse.json({ accepted: true }, { status: 202 });
    const user = await store.findUserByEmail(address);
    if (!user || (await email.isSuppressed(address))) return accepted;

    const token = randomBytes(32).toString("base64url");
    const digest = createHash("sha256").update(token).digest("hex");
    const expiresAt = new Date(Date.now() + 15 * 60 * 1000);
    await store.saveToken({ userId: user.id, digest, expiresAt });

    const resetUrl = new URL("/reset-password", process.env.APP_ORIGIN);
    resetUrl.searchParams.set("token", token);

    await email.sendTemplate({
      to: address,
      template: "password-reset",
      data: { displayName: user.displayName, resetUrl: resetUrl.toString() },
      idempotencyKey: `password-reset:${digest}`
    });

    return accepted;
  };
}
```

Set `APP_ORIGIN` to an HTTPS origin you control. The token's 32 random bytes provide 256 bits before encoding; the database receives its SHA-256 digest, not the bearer token delivered in the link. The 15-minute lifetime is an explicit product choice in this example, not a universal default. Test the complete redemption path, including expiry and one-time use.

This sample returns `202` even when an address is absent or suppressed. It also avoids logging the token. Those two details matter more than shaving an import from the adapter.

For a receipt, keep the interface and change the template data. Use the settled payment identifier for idempotency, and build line items from the authoritative order record rather than trusting webhook display fields. The payment handler should acknowledge only after the send has been durably queued or recorded; exact transaction mechanics depend on the application's store.

## Why check suppression before rendering?

A suppression lookup prevents repeated attempts to an address already blocked or bounced. Do it before expensive template work and before calling send. A local policy table may still be useful because provider suppression states do not encode every application decision, such as a user disabling optional mail.

There is a trap here: suppression cannot silently turn account recovery into a dead end. The public endpoint should still avoid account enumeration, while the product needs a separate recovery route through support or another verified factor. Email does not supply a hosted OTP fallback in the capability considered here, so that fallback remains application work.

This trade-off is easy to miss.

Template preview belongs in development and review. Render the longest realistic display name, an encoded reset URL, and narrow mobile layouts. Then send seed messages to the mailbox providers your users actually use. Preview catches HTML and link mistakes; it does not prove inbox placement.

Deliverability starts below the API call. Authenticate the sending domain, align the visible sender with the authenticated domain, and monitor bounces and complaints. DKIM signing is standardized by RFC 6376. DMARC policy and reporting are defined by RFC 7489. Neither standard promises that a message lands in the inbox.

## Which provider minimizes glue?

There is no universal winner. Count the code and operational surfaces required by this workflow, then test with one template and one delivery query. I would time five tasks: domain setup, first accepted send, preview iteration, suppression lookup, and locating a failed delivery. A stopwatch is more useful than a feature-grid checkmark.

| Option | Integration shape | Strong fit | Boundary to inspect |
| --- | --- | --- | --- |
| Resend | REST API and Node.js SDK, with React Email support | Teams already expressing templates as React components | Verify how suppression and event inspection fit the desired workflow |
| Postmark | Server-scoped API and hosted templates | Transactional email with separate message streams and template management | Account for provider-specific template aliases and server tokens |
| Amazon SES | AWS SDK/API integrated with AWS identity and monitoring | Backends already operating inside AWS | More assembly is typically needed around templates, events, and suppression workflows |
| Twilio SendGrid | REST API/SDK with dynamic templates | Teams using SendGrid's template and contact ecosystem | Keep dynamic-template data and delivery-event handling behind the adapter |
| Infrai | One REST contract spanning capabilities, with public discovery and runnable TypeScript examples | A backend that values swapping the vendor behind a capability without changing application code | Email events are pull-based, there is no SMTP relay, and domestic email vendor readiness cannot be treated as compliance evidence |

The last option is attractive when the contract itself is the product decision. Its concrete advantage is **one key and one bill** across 295 routes in 20 modules, so adding another backend capability does not add another credential and invoice reconciliation path. Suppression checking fits the reset flow. Its limitations are material. Delivery investigation must poll the email event list rather than wait for webhook pushes. Scheduled email has no cancellation operation. It also does not cover voice, WhatsApp, or RCS, so a future omnichannel recovery plan needs another boundary.

It is not a fit when webhook-driven delivery events are required in real time, when SMTP relay is mandatory, or when domestic email vendor readiness is a compliance prerequisite. Choose Resend when React Email is the center of the workflow, Postmark when its transactional template and message-stream model matches the operating setup, or Amazon SES when the team already wants AWS identity and monitoring primitives. This is a real constraint, not a footnote: an adapter can isolate syntax, but it cannot manufacture an event model the provider does not expose.

Do not choose from the table alone. Build the adapter against two candidates. If provider B cannot replace provider A without edits to the route above, vendor details have escaped. Fix that before production.

Measure the glue.

## What would I change at scale?

First, move sending out of the request path. Commit the reset-token record and an outbox record together, then let a worker claim the outbox item. The worker owns bounded retries and records the provider message ID. The HTTP response stays fast, while an email outage does not discard the intent to send.

Second, make the worker idempotent. A queue may deliver twice. Put a unique constraint on the logical message key and treat a duplicate as success. For receipts, the key can derive from the payment ID. For resets, each newly minted token digest identifies one email intent.

Third, add polling with restraint. Poll delivery events for active investigations or a bounded recent window, checkpoint the last observed position, and back off between empty reads. Pull-only events limit real-time orchestration; they are still adequate for a support dashboard and scheduled reconciliation. Do not hammer the endpoint.

I would benchmark adapter effort before throughput. Count required environment variables, SDK packages, vendor types crossing the interface, and lines in the first working adapter. Then measure send acceptance and event visibility in a test account. No invented latency number belongs in an architecture decision.

The final decision rule is blunt: choose the provider that completes suppression, preview, send, and delivery inspection with the least application-specific glue, provided its event model meets the recovery SLA. Keep tokens and payment state in your database. Keep HTML behind a named template contract. Keep the provider replaceable.

## References

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Resend documentation](https://resend.com/docs)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
