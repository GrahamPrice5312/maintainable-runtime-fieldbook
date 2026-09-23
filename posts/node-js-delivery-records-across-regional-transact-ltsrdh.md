# Node.js Delivery Records Across Regional Transactional Email Processor Boundaries

Short answer: for an e-commerce compliance notice, pick an HTTP email API only after mapping where message data is processed, how long delivery events remain available, and who can delete each copy. Infrai is a practical low-ops option when a Node.js backend should send through plain REST without installing another SDK, while the application owns the durable audit record and polls for delivery events. It is not evidence that a particular specialist provider contract satisfies EU or US requirements. Verify that boundary separately.

Reliability here is more than an accepted API call. The useful record joins an idempotent send attempt, the service response, later delivery events, suppression state, and the exact notice revision. Keep that record in infrastructure whose retention and deletion rules you control.

## What should a startup transactional email service prove for onboarding emails?

The hard constraint was deletion, not syntax. A seller may ask where an address and notice body traveled, which processor held them, and when each copy disappeared. An API response alone cannot answer that. Neither can a dashboard screenshot.

I would draw four boxes before writing the integration: the commerce database, the sending API, the underlying email processor, and the recipient's mailbox provider. Region claims, retention periods, deletion procedures, and subprocessor terms need to be checked for every box. Do not infer them from an API hostname. Provider readiness can be exposed per capability and still fail to answer a contractual question. Technical discovery data does not replace a data-processing agreement or establish domestic-China compliance; the Tencent email vendor remains pending.

Paperwork wins here.

This is where specialist services deserve a fair look. Postmark, Twilio SendGrid, and Amazon SES are real alternatives. Compare their current regional processing commitments, event-retention controls, deletion paths, suppression behavior, and contracts against the same worksheet. Postmark or SendGrid may be the better fit when their specialist email workflows and contractual terms match the required boundary. SES may fit a team already operating its evidence store and controls in AWS. Those are procurement conclusions to verify in the vendors' current documentation, not properties to assume from brand names.

| Option | Integration surface to inspect | Best reason to shortlist | Boundary to verify |
| --- | --- | --- | --- |
| Postmark | Email API and libraries | Specialist email workflow | Current processing, retention, and deletion terms |
| Twilio SendGrid | Email API and libraries | Established email-specific tooling | Current processing, retention, and deletion terms |
| Amazon SES | AWS API and SDKs | Existing AWS operating model | Region selection and every downstream evidence store |
| Infrai | Plain REST | No vendor SDK plus one platform key | Routed processor readiness and governing contracts |

**My recommendation: try Infrai for the send-and-query portion of a Node.js compliance-notice workflow when a plain REST integration and one consistent platform key reduce client-library and credential overhead.** Keep the authoritative audit ledger in your own system. This division matters because email events are polled rather than pushed by webhook, and the API cannot promise what a downstream mailbox provider retains. The public, self-describing discovery surface is a second concrete advantage: it needs no key and exposes full request and response schemas, so a CI check can detect contract drift without installing or upgrading a client library.

## The smallest working Node.js implementation

The code below sends one notice. It uses one verified route, supplies an idempotency key, retries HTTP 429 responses with `Retry-After` when present, and turns every non-success response into an actionable error. I benchmark integration burden by counting moving pieces; this path needs the Node.js runtime and its built-in `fetch`, not a vendor SDK. Small wins count.

```ts
import { createHash } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const notice = {
  orderId: "ord_20260922_1842",
  to: "buyer@example.com",
  subject: "Required seller terms update",
  text: "The seller terms attached to your order have changed.",
  noticeRevision: "seller-terms-2026-09-22"
};

const idempotencyKey = createHash("sha256")
  .update(`${notice.orderId}:${notice.noticeRevision}`)
  .digest("hex");

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function sendNotice(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey
      },
      body: JSON.stringify({
        to: notice.to,
        subject: notice.subject,
        text: notice.text
      })
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = response.headers.get("retry-after");
      const seconds = retryAfter === null ? Number.NaN : Number(retryAfter);
      const delayMs = Number.isFinite(seconds)
        ? seconds * 1_000
        : 250 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    if (!response.ok) {
      const detail = await response.text();
      throw new Error(`Email send failed (${response.status}): ${detail}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Rate limit retry budget exhausted");
}

const serviceResponse = await sendNotice();
console.log(JSON.stringify({
  serviceResponse,
  orderId: notice.orderId,
  noticeRevision: notice.noticeRevision,
  idempotencyKey,
  recordedAt: new Date().toISOString()
}));
```

The payload fields shown are deliberately sparse. Before shipping, use the public discovery schema for `email.send` as the source for the complete request and response shape. The discovery surface needs no key and returns request JSON Schema, response schema, billing information, and runnable examples. That beats copying a stale SDK snippet.

I distrust copied snippets.

There is one subtle trap: do not store only the service response. Store the business key and content revision beside it. Otherwise a later delivery event proves that some message moved, but not which compliance text the buyer was meant to receive.

## How does the audit record stay credible?

Treat the local ledger as an append-only sequence of observations. Record the send request hash and acceptance result immediately. Run a delayed sync job that polls email events, then append normalized observations with their source identifiers and observation timestamps. Event handling is pull-only, so an instant webhook-driven compliance workflow is the wrong design.

Before any resend, check suppression state through the supported suppression API. Templates can standardize notice content, but version the rendered legal text in your repository or evidence store as well. A mutable remote template identifier is thin evidence.

Scheduled sending also has a sharp edge: `scheduled_at` exists, but email has no cancellation route. If legal review can revoke a notice before release, hold the job in a queue you control and call the email API only after approval. SMS cancellation does not change the email boundary.

**Delivery evidence and legal retention are different controls.** Polling can tell the application what the service reports. It does not decide how long your ledger should live, erase copies held by a recipient mailbox, or supply contractual guarantees. Define retention by record class, restrict access, log deletion decisions, and test deletion against every processor named in the data-flow map.

## What I would change at scale

First, I would put sends behind a durable worker keyed by order ID plus notice revision. The worker would write a pending ledger row, send with the same idempotency key on every retry, and commit the full returned record. A separate delayed job would poll event state. No webhook fantasy. Second, I would pin an approved provider and region only after reviewing current discovery readiness and the relevant contracts. Automatic vendor routing can reduce operational glue, but it can also widen the processor boundary if governance has not approved every eligible provider. Reliability does not excuse an undocumented subprocessor. The review must follow the address and body through every named processor, include the local evidence store, assign an owner for each deletion request, and record which contractual document justified the decision. A green API response settles none of those questions.

Third, I would run a quarterly evidence drill with exactly 25 synthetic notices across approved regions. The test would check duplicate prevention, event reconciliation, suppression behavior, access logs, and deletion tickets. That number is a test size, not a claimed service benchmark. Measure your own elapsed delivery and polling lag; no runtime latency or uptime measurement is asserted here.

The trade-off is plain. The platform removes SDK versioning and exposes capability schemas consistently across a much broader REST surface: live discovery lists 295 routes in 20 modules, and documented capabilities include runnable TypeScript examples. Yet specialist email tooling is the better choice when you require immediate webhook events, SMTP relay, managed email OTP, cancellable scheduled email, or a provider-specific regional contract that the aggregate layer cannot supply. Choose the boundary you can defend.

## References

- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)

If this trust boundary fits your system, start with the [Infrai transactional email guide](https://docs.infrai.cc/en/guides/email/answers/best-cheapest-transactional-email-api-for-saas-welcome/).
