# Marketplace Event Notifications Explained: SMS Failures and Signature Troubleshooting in Node.js

Treat a delivery failure as an evidence problem before treating it as a resend problem. For a marketplace receipt sent after payment settles, the best default is one durable notification record, one stable message intent, and attempts appended beneath it. Automatic resend is acceptable only for a temporary transport failure. A rejection associated with sender registration, message policy, or carrier filtering should stop automation and enter review.

| Choice | Evidence quality | Duplicate risk | Use it when |
|---|---|---|---|
| Retry every non-delivery | Low | High | Never as the default |
| Retry only classified temporary failures | High | Controlled | The transport result is specific enough to act on |
| Hold and review | Highest for ambiguous cases | Lowest | Registration or filtering may be involved |

**TL;DR:** model the receipt and each send attempt separately. Record the payment event, consent context, sender identity, content fingerprint, provider-neutral outcome, and raw diagnostic reference. Then let a narrow policy decide among retry, hold, and stop. Fast retries feel responsive; unexplained duplicates and missing evidence are much harder to defend.

## What evidence should survive SMS event notification carrier failures?

An accepted request proves very little. It says the messaging transport accepted work. It does not prove that a handset received the SMS, that the sender identity was eligible for the route, or that later filtering did not occur. Those are different events, and flattening them into `sent: true` destroys the exact evidence needed during an investigation.

Stop there.

Keep the state machine boring: `queued`, `accepted`, `delivered`, `temporary_failure`, `policy_hold`, and `permanent_failure`. The names are local domain states, not copied transport statuses. An adapter maps external callbacks into them while preserving the original payload behind a restricted diagnostic reference. That separation costs a little glue. It prevents transport vocabulary from leaking through every CLI, worker, and dashboard.

The payment event also needs its own identity. A receipt is triggered by a settled payment, so the notification intent should be unique on something like `(payment_id, channel, purpose)`. A callback, a worker restart, and a manual replay may all revisit the same intent. None should silently create a second logical receipt. Include a request signature verification result in the event evidence when the settlement notification arrives through a signed webhook; store the result and key identifier, not a reusable secret.

This is the first check I would benchmark: replay the same settlement event 1,000 times in a test harness and assert that storage still contains one intent. The number is a test workload, not a production claim. It exposes a bad uniqueness boundary quickly, without a giant load-testing stack.

## Governance starts with two hard boundaries

The first criterion is **reconstructability**. An operator should be able to answer what triggered the receipt, which sender identity was selected, what content was submitted, what the transport reported, and why the system retried or stopped. Store a content fingerprint instead of copying sensitive receipt text into broad logs. Keep the raw callback in access-controlled storage and put its immutable reference in the attempt record.

Evidence should include timestamps from your system and the transport, but do not pretend they share one clock. Record ingestion time separately from an external event time. Also retain the mapping version that converted a raw result into a local state. When that mapping changes, old decisions remain explainable.

The second criterion is **bounded action**. “Undelivered” is not a retry policy. A temporary transport condition may justify a delayed retry. An unknown result should usually hold. A registration or filtering signal should stop automated sends until the sender, route, and content have been checked. Otherwise the retry loop repeats the same input and produces more noise, not more information.

Security guidance for one-time codes makes the same broader point about message workflows: responses should be consistent, codes should expire, codes should be single use, and excessive requests need protection. A payment receipt is not an authentication secret, but the operational lesson transfers cleanly: resending must be bounded and tied to one intent. Do not turn a receipt endpoint into an unmetered message trigger.

## Make the audit record executable

I dislike configuration that exists only because the code has no opinion. This contract has three policy outcomes and one explicit audit shape. No retry-strategy object graph. No dozen knobs.

```ts
type Outcome =
  | { kind: "delivered" }
  | { kind: "temporary_failure"; retryAfterMs: number }
  | { kind: "policy_hold"; reason: "sender_registration" | "carrier_filtering" | "unknown" }
  | { kind: "permanent_failure"; reason: string };

type ReceiptIntent = {
  intentId: string;
  paymentId: string;
  destinationRef: string;
  senderId: string;
  contentSha256: string;
};

type Attempt = {
  attemptId: string;
  intentId: string;
  submittedAt: string;
  mappingVersion: string;
  diagnosticRef: string;
  outcome: Outcome;
};

type Decision =
  | { action: "complete" }
  | { action: "retry"; at: string }
  | { action: "hold"; reason: string }
  | { action: "stop"; reason: string };

function decide(outcome: Outcome, attempts: number, now: Date): Decision {
  if (outcome.kind === "delivered") return { action: "complete" };

  if (outcome.kind === "temporary_failure" && attempts < 3) {
    return {
      action: "retry",
      at: new Date(now.getTime() + outcome.retryAfterMs).toISOString(),
    };
  }

  if (outcome.kind === "policy_hold") {
    return { action: "hold", reason: outcome.reason };
  }

  return { action: "stop", reason: "retry limit or permanent failure" };
}
```

The limit of three is an application policy in this example, not a universal carrier rule. Put that distinction in the code review. The transport adapter may supply a retry delay only when it has a classified temporary result; the policy function does not guess one from silence.

Persist the attempt before dispatching follow-up work. Use the intent ID as the idempotency boundary inside your system, and give every actual submission a separate attempt ID. A queue can deliver a job more than once. That should create neither a fresh intent nor an undocumented resend.

For observability, count transitions rather than dumping message bodies: attempts per intent, time spent in `accepted`, holds by reason, callback age, and duplicate callback count. Alert on a change in the distribution. A single failure is a case; a sudden cluster by sender identity or destination region is a system signal.

## Reconstruct filtering before changing the message

Start with one correlation trail: payment ID, intent ID, attempt IDs, sender identity, destination region, submit time, callback time, and the preserved diagnostic reference. Label the destination context, including US or EU when applicable, without assuming one region's sender registration evidence explains another's outcome. Verify that the receipt content and sender match the approved operational purpose. Then check whether the affected messages share a sender, route, content fingerprint, or destination pattern. Compare the first affected attempt with the last known delivered attempt, one field at a time: sender, destination context, content fingerprint, mapping version, and transport result. This is slower than firing another request, but it produces an actual troubleshooting result. If more than one field changed, the evidence cannot isolate a cause, so hold the message and reduce the next test to one reviewed change.

Do not edit content and resend in the same action. That erases the comparison. Create a reviewed revision, give it a new content fingerprint, and link it to the same receipt intent. The audit trail can then show exactly what changed.

For email fallback, sender discipline still matters. Yahoo's published sender guidance calls for valid forward and reverse DNS, DKIM, DMARC for bulk senders, low spam complaint rates, and functioning unsubscribe behavior for promotional mail. A transactional marketplace receipt has a different purpose from promotional mail, but fallback does not excuse weak authentication or mixed intent. Keep operational and promotional flows distinguishable.

The awkward case is silence: no useful final event, no specific rejection, and no proof of delivery. Mark it unknown and hold or move to an independently governed fallback. Do not relabel silence as a temporary failure just to keep the worker busy.

Silence isn't evidence.

The same evidence shape can support telehealth login verification, but the resend policy cannot be copied unchanged. A verification code is short-lived and security-sensitive; OWASP's guidance calls for expiring, single-use tokens and protection against excessive requests. A marketplace receipt is a durable business record. Mixing those purposes under one retry policy is a category error.

## The exception path is slower for a reason

Manual review is the runner-up to classified automatic retry because it is slower. It is still the better path when the diagnostic is ambiguous, a sender identity has recently changed, several destinations show the same filtering pattern, or the next attempt would use materially different content. Humans can compare registration evidence and approve a controlled revision; a timer cannot.

The main limitation of the evidence-first design is operational latency. A hold queue can delay a legitimate receipt, and maintaining mapping versions plus restricted diagnostic storage adds work. The trade-off is deliberate: use automatic retry when the failure classification is strong, and choose manual review when another automated submission would merely repeat an unexplained input.

A channel fallback is also better when the business requirement is “make the receipt available,” not “force an SMS through.” Put the receipt in the authenticated marketplace account first. SMS or email can announce availability without carrying the full record. This reduces sensitive content in the notification and gives support one canonical receipt to reference.

Ship the narrow policy. Measure it. The useful benchmark is not raw send latency; it is how quickly an operator can reconstruct one failed receipt without searching five systems. If the evidence record answers that in one query, the design is doing its job.

## Sources

References:

- OWASP, “Forgot Password Cheat Sheet”: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- Yahoo, “Sender Best Practices”: https://senders.yahooinc.com/best-practices/
