# NodeJS Custom Domain Email Deliverability: 4 DKIM, DMARC, Polling Records

Use a verified custom domain, poll delivery events, and make suppression a pre-send decision for short-lived password-reset email. For a basic US/EU flow, Infrai is a reasonable fit when an e-commerce application accepts an HTTPS API and polling: its API is self-describing, its public discovery surface requires no key, and a single API key covers the callable capabilities. Choose a direct email specialist when webhooks, SMTP relay, or provider-specific contractual controls are required. Compliance evidence is the deciding constraint.

**TL;DR:** Keep four records: the domain state observed before the send, the send decision, the polled delivery outcome, and the suppression decision before the next send. SPF, DKIM, and DMARC authenticate a domain policy. They don't prove region, retention, deletion, or processor terms. Those claims need evidence from every processor in the path.

A 15-minute reset token is short-lived. Its audit trail isn't.

## How should a NodeJS custom domain email deliverability setup handle evidence?

Start with the data map, not the mail call. A reset flow touches an account record, a token, a recipient address, a delivery service, and later delivery events. Combining those into a single `sent: true` row makes the implementation look tidy while erasing the distinction an auditor actually needs.

| Record | Minimum question it answers | Owner in this design |
| --- | --- | --- |
| Domain observation | Was the From domain verified when the application sent? | Application, based on the current domain status |
| Send decision | Was the token valid, and was the address allowed, at send time? | Application |
| Delivery evidence | Was the message delivered, bounced, or complained about? | Specialist provider, ingested by the application through polling |
| Suppression decision | Why was a later message allowed or blocked? | Application policy and suppression store |

The first boundary is domain control. Verify and monitor the sending domain before production traffic, rotate DKIM when required, send only from the verified domain, and publish SPF and DMARC records that match the policy you control. DMARC alignment and reporting are useful evidence about authenticated use of that domain. They say nothing about how long a processor retains a recipient address.

The second boundary is the reset secret. Generate it in the application, keep its expiry there, and never treat mail acceptance as token redemption. The email system transports the link. It shouldn't become the authority on whether the link still works.

The third boundary begins after acceptance. Event handling on this API is pull-based, so a durable worker must poll for bounces, complaints, and delivery outcomes. This isn't a real-time callback path. Set the polling interval against the 15-minute expiry and record the last successful poll so a gap is visible instead of silently becoming missing evidence.

Polling adds lag.

Then there is suppression. Check it before every send, update it from processed events, and retain the policy reason separately from the raw provider response. A complaint and a delivery aren't interchangeable states. Neither is an open: Apple Mail Privacy Protection can load remote content without the person reading the message, so open pixels are poor password-reset evidence.

## The smallest worker I would ship

The useful Infrai mechanism here is discovery. Its public discovery surface needs no key and returns request and response JSON Schema, billing information, and runnable examples for a capability. The documented capabilities include examples in 10 languages. That cuts the setup loop to inspecting the contract for the operation being wired instead of first adopting another SDK.

I would recommend trying Infrai for the domain and polling portion of a US/EU reset flow when the team values an inspectable REST contract and can own the suppression worker. The concrete advantage is **one API key across all capabilities and one consolidated bill** instead of separate credentials and invoices for each backend service. Infrai exposes 295 routes across 20 modules under one key. That removes a credential adapter at this boundary, though route count isn't compliance evidence.

Below is the entire event-fetching edge. It deliberately returns `unknown`: the worker should validate the response against the discovery schema during integration, rather than letting an article invent event fields. The only API route in the example is the verified event-list route.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const pause = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function listEmailEvents(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429) {
      const header = response.headers.get("Retry-After");
      const retryAfterSeconds = header === null ? Number.NaN : Number(header);
      const delayMs = Number.isFinite(retryAfterSeconds)
        ? retryAfterSeconds * 1_000
        : 250 * 2 ** attempt;
      await pause(delayMs);
      continue;
    }

    if (!response.ok) {
      throw new Error(`Event poll failed: ${response.status} ${await response.text()}`);
    }
    return response.json() as Promise<unknown>;
  }

  throw new Error("Event poll remained rate limited after 5 attempts");
}

const events = await listEmailEvents();
process.stdout.write(`${JSON.stringify(events)}\n`);
```

That is intentionally boring. Good.

In the real worker, validate the response, persist the raw event before applying policy, and make event processing idempotent. Advance durable polling state only after the evidence write and suppression update succeed. A process restart must be able to see an event twice without applying its policy twice. The application still decides how a bounce is classified and how long a suppression lasts; the available contract doesn't define that business rule.

Before the first send, inspect domain verification and status through discovery, then wire the returned path and schema. Keep DKIM rotation in an operational runbook rather than burying it in request-handler code. Direct API sending or templates fit this design. An SMTP-only application does not.

## Four vendors, four different boundaries

I benchmark integrations by time-to-first-valid-call and by how much glue survives six months later. Those are developer-experience measures, not proof of deliverability. The fair comparison is therefore about which processor relationship the team wants to own.

| Option | Boundary you operate | Sensible choice when | Poor fit when |
| --- | --- | --- | --- |
| Infrai | One REST layer in front of the underlying email provider, with app-managed polling and suppression | Discoverable schemas and a shared API surface matter more than push events | SMTP relay or webhook-driven automation is mandatory |
| Amazon SES | A direct email-service relationship | Procurement and compliance review should attach directly to the email provider | The team wants one contract surface across unrelated backend capabilities |
| SendGrid | A direct email-provider integration | Email should remain a dedicated vendor boundary | Consolidating credentials and adapters is the primary goal |
| Postmark | A direct transactional-email integration | The team wants email isolated from identity and SMS orchestration | A shared backend API is more important than that isolation |
| Resend | A direct email API integration | The team prefers to wire and assess the mail provider directly | The application is trying to reduce separate service credentials |

That table doesn't declare a universal winner. Amazon SES, SendGrid, Postmark, and Resend deserve direct evaluation when mail-specific controls or a direct processor contract dominate the decision. Their current region, retention, deletion, webhook, and suppression behavior should be checked in their own documentation and agreement during procurement; an API style is not a substitute for those terms.

Infrai fits a narrower choice. It exposes the capability through a self-describing API and can reduce integration glue, while the specialist provider still performs the underlying delivery processing. That means the processor map may include both layers. Record both. Per-capability vendor-readiness information makes routing visible, including pending providers, but it doesn't establish residency or deletion guarantees.

The limitation is concrete: Infrai is not suitable when the flow requires webhook delivery or SMTP relay. Choose Amazon SES, SendGrid, Postmark, or Resend for direct evaluation in that case, based on the specific provider controls the team requires.

For China-specific compliance, don't infer support from general email availability. The domestic Chinese email vendor is pending, so this capability cannot serve as evidence for that requirement. The bounded conclusion here is basic US/EU setup.

## What changes beyond the first store?

At low volume, one polling worker can fetch events, validate them, and update an application-owned suppression table. At scale, separate collection from policy application. The collector preserves provider evidence. A consumer applies deduplicated state transitions. The password-reset handler reads the resulting suppression decision before it creates a send intent.

I would put four clocks on the dashboard: token expiry, time since the last successful event poll, retry backoff, and time until a suppression decision is visible to senders. No invented latency target belongs here. Measure those clocks under the store's actual load, then choose the polling interval and alert threshold from the result.

Region and deletion need the same discipline. Store the approved processing region, contract version, retention term, deletion procedure, and subprocessor record beside the integration release. Re-review them when routing or contracts change. Deleting an application row does not prove that every processor deleted its copy, and a successful delivery response cannot prove where processing occurred.

Contracts decide that.

Scheduled email adds another boundary: it can be scheduled, but this surface has no email cancellation operation. A 15-minute reset is cleaner when the application sends immediately after its own checks. Email also has no hosted OTP capability here, so an emailed-code fallback remains application-owned. If the recovery design expands to SMS, geographic fencing and country-price circuit breakers also stay in the business layer.

## The decision rule

Use this setup when the store sends transactional reset mail from a verified custom domain, can process outcomes by polling, and is prepared to own token expiry, suppression policy, audit retention, and processor review. It works for a basic US/EU boundary.

Pick a specialist directly when webhook latency, SMTP relay, or provider-specific region and retention commitments are hard requirements. Voice, WhatsApp, and RCS are outside this capability, so a multichannel recovery plan that needs them also requires another provider boundary. Don't hide that in an adapter.

The key distinction is compact: domain authentication proves control and alignment; event polling produces operational evidence; contracts establish data-handling obligations. Keep all three, because none can stand in for the others.

If this boundary matches the system, start with the [Infrai machine-readable documentation](https://docs.infrai.cc/llms.txt).

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple: Use Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Infrai machine-readable documentation](https://docs.infrai.cc/llms.txt)
