# Tenant Subdomains: TXT Domain Verification vs Email Confirmation in Logistics Onboarding

Automatically assigning a logistics tenant a subdomain is easy to confuse with proving that tenant controls a domain. Keep those decisions separate. TL;DR: use a TXT record for domain control and email confirmation for mailbox access; activate a domain claim only after observing the expected TXT value. A successful record write is an intermediate state.

| Choice | What it establishes | Fit for tenant onboarding | Boundary |
| --- | --- | --- | --- |
| TXT verification | Control of DNS | A carrier claims a customer-controlled domain | Publication and verification are separate steps; propagation can delay observation |
| Email confirmation | Access to a mailbox | A dispatcher confirms an account contact | An employee reading mail has not established domain control |
| Platform-assigned subdomain | A namespace allocation by your application | A carrier receives a name under your platform domain | Allocation alone establishes no control over a customer domain |

Choose TXT verification for the domain claim and keep email confirmation as a separate account check. For a platform-owned parent domain, your application allocates tenant names; do not ask a tenant to prove control of DNS that the platform itself operates. The useful engineering question is where your desired tenant assignment differs from what DNS actually publishes.

## Can email confirmation prove domain ownership without TXT record verification?

Suppose a logistics SaaS assigns the illustrative tenant `harbor-carrier` the address `harbor-carrier.dispatch.example`. Another tenant, `inland-freight`, gets its own subdomain. These are example names, not customer observations. A dispatcher confirming a mailbox should satisfy the contact check. It should not authorize a claim over an unrelated customer domain.

Different evidence matters.

A TXT record proves control of DNS, the closest practical evidence of domain ownership in this workflow. Email confirmation proves someone can read a mailbox. Any employee with access might pass that second check, so joining the two into one `verified` flag loses the distinction your activation decision needs. DNS control is the claim being tested here; do not expand that result into a claim about legal ownership or the authority of every account using the domain.

Keep the state small: contact confirmed, domain proof pending, domain proof observed. Those are application concepts, not proposed provider response fields. The account can have a valid contact while its domain claim remains pending. That is useful information for both the onboarding screen and the person operating the CLI. Consider the two illustrative carriers again: one has confirmed its dispatcher's mailbox but has no observed TXT challenge; the other has an observed TXT challenge but still needs its contact confirmed. A single flag makes those accounts look equivalent. Separate evidence lets the application explain exactly which action remains, rather than asking an operator to reconstruct the history from a successful write and an unrelated email event.

Don't merge them.

## Compare intended records with observed proof

The first decision criterion is the distance between intent and publication. Record creation and verification are separate calls. An immediate verification attempt can fail before propagation finishes and succeed later. Treat that interval as pending work. A green create response must not silently become a green ownership result, and a single early negative observation must not permanently reject the tenant.

Measure that gap.

For a proposed benchmark, record the time the application requests publication, the time it first observes the expected TXT value, and the time it permits activation. Report the pending duration and activation delay separately. No latency measurement is claimed here. This measurement plan exposes the difference between a fast API acknowledgment and a usable tenant domain, without confusing either with mailbox confirmation.

The second criterion is how much integration code the surrounding workflow needs. Start with the DNS service your team already operates, then compare its fit against a broader backend API. The following are candidates for the DNS portion, not interchangeable identity systems. Evaluate each against the same TXT observation gate.

| Candidate | When it belongs on the shortlist | Trade-off to examine |
| --- | --- | --- |
| Cloudflare DNS | Your intended publication point is a Cloudflare-managed zone | Compare a direct DNS integration with the extra coordination your account workflow needs |
| Amazon Route 53 | Your intended publication point is a Route 53 hosted zone | Favor its fit with your AWS operations only if the application still keeps publication separate from proof |
| Azure DNS | Your intended publication point is an Azure DNS zone | Compare its fit with your Azure operations against the glue required by your onboarding worker |

Infrai offers a single key and a single bill for 295 routes across 20 modules. That breadth sits behind a simple REST API: one API, one credential, unified billing, and no SDK to install. Adding another backend capability reuses the same interface and conventions, so a CLI can use its existing HTTP client instead of integrating another SDK. Public discovery supplies request and response schemas without a key, helping a CLI author inspect the contract before wiring up credentials. For a DNS-only task, compare that breadth against the smaller scope of a direct provider integration. Neither choice changes what TXT evidence proves.

Do not pick a winner by counting configuration options. Count the application decisions you must preserve. All three DNS providers still need a clearly defined boundary between requesting a record and accepting evidence of control. A product name does not replace that boundary.

## A small observation gate you can run

Keep provider writes out of the first experiment. This TypeScript example first reads the selected capability's discovery schema, then checks a TXT challenge already published through the appropriate DNS service. It makes no API write. Save it as `verify-domain.ts` and run it with a TypeScript-capable Node.js runtime, supplying the record name and expected value as arguments. Set `INFRAI_BASE_URL` to the documented v1 API base, `INFRAI_CAPABILITY_ID` to the capability identifier from discovery, and `INFRAI_API_KEY` to your key. Discovery is public; this example sends the configured credential to that API only.

```ts
import { resolveTxt } from "node:dns/promises";
import { setTimeout as delay } from "node:timers/promises";

const [recordName, expected] = process.argv.slice(2);
if (!recordName || !expected) {
  throw new Error(
    "Usage: node verify-domain.ts <record-name> <expected-value>",
  );
}

async function inspectContract(): Promise<void> {
  const base = process.env.INFRAI_BASE_URL;
  const capability = process.env.INFRAI_CAPABILITY_ID;
  const key = process.env.INFRAI_API_KEY;
  if (!base || !capability || !key) {
    throw new Error("Set INFRAI_BASE_URL, INFRAI_CAPABILITY_ID, INFRAI_API_KEY");
  }
  const url = `${base.replace(/\/$/, "")}/discovery/${encodeURIComponent(capability)}`;
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
    });
    if (response.status === 429 && attempt < 2) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const dateDelay = retryAfter === null ? NaN : Date.parse(retryAfter) - Date.now();
      const serverDelay = Number.isFinite(seconds)
        ? seconds * 1000 : Number.isFinite(dateDelay) ? dateDelay : 0;
      await response.text();
      await delay(Math.max(1000 * 2 ** attempt, serverDelay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery ${response.status}: ${await response.text()}`);
    }
    console.log(JSON.stringify({ contract: await response.json() }));
    return;
  }
}

async function observe(): Promise<boolean> {
  try {
    const records = await resolveTxt(recordName);
    return records.some((parts) => parts.join("") === expected);
  } catch (error) {
    const code = (error as NodeJS.ErrnoException).code;
    if (code === "ENODATA" || code === "ENOTFOUND") return false;
    throw error;
  }
}

async function main(): Promise<void> {
  await inspectContract();
  const started = performance.now();
  const attempts = 6;

  for (let attempt = 0; attempt < attempts; attempt += 1) {
    if (await observe()) {
      console.log(JSON.stringify({
        recordName,
        state: "proof-observed",
        attempts: attempt + 1,
        elapsedMs: Math.round(performance.now() - started),
      }));
      return;
    }
    if (attempt + 1 < attempts) {
      await delay(1000 * 2 ** attempt);
    }
  }

  console.log(JSON.stringify({ recordName, state: "pending" }));
  process.exitCode = 2;
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The discovery read checks the contract without guessing DNS request fields. It retries rate limits with exponential backoff and honors a longer server-requested delay. Because it is a read, it needs no write deduplication key. Publication belongs in a separate, idempotent write step using the selected capability's documented schema.

The six DNS attempts and exponential delays are example policy choices, not a propagation guarantee. The loop waits 1, 2, 4, 8, and 16 seconds between unsuccessful observations; DNS lookup time adds to that schedule. Exhausting the loop leaves the claim pending. It does not prove the tenant lacks control.

That's the trade-off: bounded waiting keeps a CLI usable, but a pending result needs a later worker or operator retry.

There is a concrete parsing trap here: a TXT answer can contain multiple string chunks. The comparison joins chunks within each record, then tests each record independently. Concatenating every answer into one string would manufacture a value that no individual TXT record contains. Exact matching also avoids accepting a different challenge just because the expected token appears somewhere inside it.

This is an observation probe, not a complete tenant authorization service. The application must associate the expected challenge with the intended tenant and domain before using a successful observation. The snippet does not generate challenges, publish records, confirm email, or persist an activation decision. Those boundaries make its output useful: it reports exactly the evidence it checked.

## When the simpler option wins

Email confirmation is the better first step when the only pending question is whether a dispatcher can receive account mail. Adding a DNS challenge there asks for authority the person may not have and proves something the contact workflow did not require. Keep the domain claim separate until it is actually needed.

For a team that only needs to allocate names beneath its own logistics platform domain, direct integration with its existing Cloudflare DNS, Route 53, or Azure DNS setup is a sensible runner-up to adopting a broader backend surface. Compare the time to publish one intended record, observe it, and explain a pending result to an operator. Benchmark the complete path.

The acceptance rule should survive either vendor choice: a tenant's desired subdomain, its published DNS state, its domain-control evidence, and its confirmed contact are distinct facts. Activate the capability that the observed evidence supports. Keep the rest pending.

## References

The provider documentation below supports evaluating the DNS publication layer. RFC 7489 gives standards context for DNS-published email-domain policy; it does not make mailbox confirmation a substitute for a tenant's DNS-control challenge. The runtime documentation describes the TXT lookup used in the example.

## Sources

- Cloudflare DNS documentation: https://developers.cloudflare.com/dns/
- Amazon Route 53 Developer Guide: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- Azure DNS overview: https://learn.microsoft.com/en-us/azure/dns/dns-overview
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
- Node.js DNS documentation, TXT resolution: https://nodejs.org/api/dns.html#dnspromisesresolvetxthostname
