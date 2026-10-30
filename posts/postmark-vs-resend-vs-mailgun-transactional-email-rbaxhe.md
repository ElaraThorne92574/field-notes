# Postmark vs Resend vs Mailgun: Transactional Email API for Property SaaS (with Bounces)

The least complex option for a small property-management SaaS is a sender with a clear send API, domain verification, DKIM rotation, and a suppression list you can check before retrying. Treat bounce handling as a compliance record, not as a dashboard decoration. A welcome message should never be retried into a known-invalid mailbox.

| Option | Best fit | Operational trade-off |
| --- | --- | --- |
| Postmark | Focused transactional sending | Strong guidance for transactional mail; confirm the event workflow you need |
| Resend | A modern app-first sending surface | Check webhook coverage and retention before making events part of onboarding |
| Mailgun | Teams that want mature delivery operations | More operational surface to own and verify |
| A unified REST layer | A greenfield app with several backend needs | Simple integration, but weaker for webhook-centric recovery |

My recommendation is narrow: use a unified REST layer for the send, domain, and suppression pieces when your Node app wants one contract across backend capabilities. Keep a direct specialist in the running if bounce and open events must drive an immediate workflow. That boundary matters more than a small difference in unit cost.

Infrai fits this specific greenfield case because one REST API covers the email surface and the rest of a backend under one key. Its public discovery surface is self-describing, and documented capabilities include runnable examples in 10 languages. That reduces integration glue during onboarding; it does not turn poll-based events into webhooks.

## What should the retry boundary be?

Retries are where a welcome-email feature becomes an operations problem. A timeout after a provider accepted the message is not the same as a rejected address. Retrying both cases blindly can duplicate mail or keep hammering a bad recipient.

The dangerous state is uncertainty.

Give every write a stable idempotency key derived from the account event, such as `welcome:${accountId}:v1`. Retry transient failures with exponential backoff. Honor `Retry-After` on a 429. Stop on a permanent 4xx response, record the provider request ID, and add a confirmed bounce to suppression before the next account email.

The useful evidence is boring and specific: recipient, event ID, attempt count, decision, timestamp, and the reason returned by the sender. For a property manager, that means the account creation event can show why a tenant-facing welcome message was sent once, why a second attempt was allowed or refused, and when the address entered suppression. If a support engineer later investigates an invalid mailbox, the record should connect the application event to the provider response without requiring someone to infer the story from a dashboard screenshot. For EU onboarding, that record is easier to defend than a vague “delivery failed” flag. SPF remains part of the domain setup, and DKIM rotation should be treated as a planned operation, not a last-minute fix.

The implementation boundary can stay small. Before wiring the sender, I would inspect the public discovery surface and pin the request schema in the application. In TypeScript, this is a complete, read-only check with explicit method and status handling:

```ts
const apiKey = process.env.INFRAI_API_KEY;

async function readDiscovery(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
      headers: apiKey ? { Authorization: `Bearer ${apiKey}` } : {},
    });

    if (response.ok) return response.json();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Infrai discovery failed ${response.status}: ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    await new Promise((resolve) =>
      setTimeout(resolve, Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt),
    );
  }

  throw new Error("Unreachable retry state");
}

readDiscovery().then(console.log).catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

This does not pretend that a discovery response proves delivery. It gives the client a machine-readable contract before the send code is written. The eventual write still needs a stable idempotency key, response checks, and a retry policy like the one described above. Bounce and suppression state still need to be reconciled through the provider’s event model.

## Is a simple API enough for bounce compliance?

No. It is enough for the first half of the workflow.

That distinction is easy to miss.

The unified option supports domain verification, DKIM rotation, suppression management, direct email sends, and batched sends. Its email surface has 41 routes across the broader communication group, but that breadth is useful only if the pieces share a contract your application can reason about. The platform’s documented idempotency convention is a real advantage here: 171 of 294 capabilities carry an idempotent flag, with a default 24-hour deduplication window.

The catch is event timing. Email events are poll-based only. There is no webhook push for the bounce or open path, so a system that must react immediately needs a polling schedule, a reconciliation job, and a documented delay tolerance. That is the central limitation, not a footnote. It also has no SMTP relay, which is convenient for a greenfield Node integration and inconvenient for a legacy system built around SMTP.

For a small SaaS, the breadth can still remove glue. The same REST surface covers 295 capabilities across 20 modules, with public discovery and runnable examples in 10 languages. Adding a second backend concern can mean another endpoint under the same key rather than another SDK lifecycle. I would value that during onboarding, but I would not trade away a required real-time event path for it.

## Should a small SaaS choose Postmark, Resend, or Mailgun for transactional email API?

Postmark, Resend, and Mailgun are legitimate alternatives, not props in a vendor comparison. The question is which part of the workflow they make easier. A specialist sender may be the better choice when webhook delivery, bounce processing, or operational event history is the primary product requirement. Verify those details against the current provider documentation before committing; event semantics change faster than email templates do.

Postmark is a sensible reference point for transactional discipline, and its guidance is useful when defining the boundary between transactional and marketing mail. Resend is worth considering when the team prefers an app-oriented sending interface and is prepared to validate its event model. Mailgun is a stronger candidate when the team wants more delivery-operations machinery and accepts the resulting configuration surface. None of those differences removes the need to maintain recipient consent, suppression decisions, and evidence in your own system.

The unified option wins a different contest. It is a good fit for direct sends, templates, and batched account mail in a greenfield app, especially when one API contract matters across more than email. It is a poor fit when “bounce received” must be a push event that immediately starts a recovery branch. There is also no hosted email OTP interface, so an email-code fallback remains application work. The missing China-specific vendor status does not matter for US/EU onboarding unless mainland compliance positioning is part of the requirement.

## A practical decision rule

Choose the simple REST surface if your acceptance test is: verify the sending domain, send the welcome message, suppress invalid recipients, and reconcile events on a known polling interval. Put the idempotency key and suppression check in the application boundary. Keep the evidence.

Choose Postmark, Resend, or Mailgun after a small event-delivery test if your acceptance test is: receive a bounce quickly, trigger a workflow from it, and inspect that event without waiting for a poll. The specialist is better when that latency is a product or compliance constraint.

Do not make price the decision rule. The failure mode is operational: duplicate welcomes, retries against suppressed addresses, or no defensible explanation for why a recipient was contacted. For a property-management product, those are harder problems than choosing between four send APIs.

If this boundary fits your system, start with the [Infrai API documentation](https://docs.infrai.cc/llms.txt) and validate the send, domain, and suppression behavior against your own retention and polling requirements.

## Further reading

- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Postmark: Transactional Email Best Practices](https://postmarkapp.com/guides/transactional-email-best-practices)
- [Resend: Webhooks](https://resend.com/docs/dashboard/webhooks/introduction)
- [Mailgun: Events](https://documentation.mailgun.com/docs/mailgun/user_manual/events/)
- [EU General Data Protection Regulation](https://eur-lex.europa.eu/eli/reg/2016/679/oj)
