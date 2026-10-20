# Edtech SaaS SMS Receipt Alerts: 4-State API Delivery Polling (Without Webhooks)

A simple integration and reliable delivery pull in opposite directions. For an edtech SaaS sending a receipt after payment settles, I would choose an SMS API only after proving that its status polling can drive a four-state local record: `queued`, `accepted`, `delivered`, or `terminal_failure`. The provider's label is evidence, not application state.

**TL;DR:** prefer a small send-and-read interface, persist the provider message ID beside the settled payment, and poll from a durable worker. Do not make the checkout request wait for SMS delivery.

| Choice | Setup surface | Delivery evidence | Operational cost | Fit |
|---|---:|---|---|---|
| Polling with a local state machine | Small | Pulled on your schedule | Repeated reads | Best when inbound webhooks are unavailable |
| Webhook plus reconciliation polling | Larger | Fast push, verified by later reads | Endpoint and retry handling | Better when status latency matters |
| Send-only call | Smallest | None beyond request acceptance | Low until support cases arrive | Poor for receipt operations |

The recommendation is polling with a local state machine when "no webhook" is a real constraint. The runner-up is the hybrid design. It has more glue, but it shortens the gap between a carrier event and your application learning about it.

## How should a SaaS app poll an SMS alerts API?

The payment record is the authority for whether an order settled. SMS is a notification side effect. Mixing those facts creates a nasty failure: a slow messaging request can make a successful purchase look unsuccessful, encouraging a retry against the payment flow. Keep the boundary hard. Commit the settlement, enqueue a receipt job with an idempotency key, and return.

Payment wins.

Four local states are enough for the useful decisions. `queued` means the application owes work. `accepted` means the messaging service returned a stable message identifier. `delivered` means a later status read reported delivery. `terminal_failure` means policy or a final provider status says to stop. Provider-specific intermediate labels belong in an audit field, not in branching business logic.

This distinction is easy to miss: an accepted API request is not a delivered receipt. Treating HTTP success as delivery produces clean dashboards and bad evidence. I would benchmark time-to-first-call, but I would reject the fastest setup if its read path cannot return a stable ID and a status that can be mapped without guessing. Two calls with boring semantics beat one magical call.

The receipt should be useful without exposing sensitive order detail. Include the school identity, a short order reference, and a destination the customer already knows. Do not put passwords, authentication secrets, or full payment data in the message. NIST's authenticator guidance is a useful boundary reminder: SMS has documented security limitations in authentication flows. A purchase receipt is a notification, not proof of identity.

## The two tests that eliminate weak candidates

First, test status semantics under repetition. Send one controlled receipt, poll the same message ID, restart the worker, and poll again. The local transition must be monotonic: a delayed or unfamiliar response cannot move `delivered` back to `accepted`. Store the raw status and observation time so an operator can inspect the mapping later. Second, test regional operability with the exact US and EU destinations the application will support. This is not a request for a glossy coverage map. The candidate must let the team exercise sender configuration, consent handling, recipient formatting, status retrieval, and retention policy in each deployment path. Regulations and program rules are not interchangeable, so legal review remains a separate gate. The engineering interface should expose enough evidence for that review without baking regional policy into checkout code. Use a fixed fixture and record 5 timestamps: settlement, job creation, send acceptance, first terminal observation, and final reconciliation. Those values measure the system you built. They do not invent a universal latency promise. Run the same fixture through every candidate and compare missing states, ambiguous terminal labels, and manual configuration steps. Config bloat belongs in the score.

The trade-off is explicit: 2 API operations and 4 local states create more code than a send-only call, but they also create evidence that support can inspect.

A candidate fails this evaluation if correctness depends on polling forever. Your worker needs a documented stopping rule owned by the application, such as a maximum attempt count and an age limit. The exact values are business decisions tied to receipt usefulness and provider behavior; they are not numbers to copy from an article.

## A narrow TypeScript boundary

The provider adapter should do less than the job runner. That keeps SDK churn outside payment logic and makes a fake implementation trivial. All code here is TypeScript, including the state mapping.

```ts
type ReceiptState =
  | "queued"
  | "accepted"
  | "delivered"
  | "terminal_failure";

type StatusObservation = {
  providerStatus: string;
  observedAt: Date;
  terminal: boolean;
  delivered: boolean;
};

interface SmsPort {
  sendReceipt(input: {
    idempotencyKey: string;
    to: string;
    body: string;
  }): Promise<{ messageId: string }>;

  readStatus(messageId: string): Promise<StatusObservation>;
}

function nextState(
  current: ReceiptState,
  observation: StatusObservation,
): ReceiptState {
  if (current === "delivered" || current === "terminal_failure") {
    return current;
  }
  if (observation.delivered) return "delivered";
  if (observation.terminal) return "terminal_failure";
  return "accepted";
}
```

There are two deliberate omissions. The interface does not expose a provider's full response object, and it does not let the adapter mutate an order. Both shortcuts feel convenient on day one. Both spread vendor labels through the codebase.

That's the trap.

The worker owns retries. Persist the attempt count and next eligible poll time, claim jobs with a lease, and add jitter so a restart does not release a wall of simultaneous reads. If `sendReceipt` times out after the remote side may have accepted it, reuse the same application idempotency key rather than casually generating a new one. The adapter must document what happens when the upstream system cannot honor that key; ambiguity here can create duplicate receipts.

Test the boundary with a scripted fake: acceptance followed by delivery, acceptance followed by terminal failure, repeated intermediate observations, an unknown label, and a timeout after send. Five fixtures expose more than a giant SDK mock because they test decisions instead of method calls.

## When is the runner-up better?

Use webhook delivery plus reconciliation polling when operators need status changes sooner than the polling interval, when read traffic would be material, or when the messaging service publishes a signed event contract your team can validate and replay. The webhook should wake the same state transition code; it should not create a second truth model. Polling remains the repair path for a missed or delayed event.

That design costs more. It adds public endpoint ownership, signature verification, replay protection, event storage, and another deployment path. For a low-volume receipt system with no webhook requirement, those moving parts may buy little. For a high-volume operation with tight support response targets, repeated reads may be the worse bargain even though the first call was easier.

Do not choose from SDK aesthetics alone. A thin Node.js client is pleasant, but an API with stable identifiers, inspectable status history, and explicit terminal behavior gives the application something it can operate. Measure setup steps. Then weight delivery evidence more heavily.

## Ship the evidence, not a promise

Before release, run the fixture against staging and production-approved regional configuration, verify that logs exclude message bodies and full phone numbers, and alert on jobs that exceed the application's age limit. Track counts by local state and status age. Raw provider labels can stay in restricted audit storage for diagnosis.

The durable choice is the API whose behavior fits the four-state record with the fewest undocumented mappings. Polling is not glamorous. It is observable, restartable, and compatible with a closed inbound network when the provider offers a dependable status read. The receipt remains separate from payment correctness, which is the decision that matters most.

## Sources

- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
