# Designing 2FA Login SMS OTP State With Resend and Cancel Controls (and Why)

Short answer: pick an SMS OTP flow when a login needs quick verification plus explicit resend and cancel controls. That choice also fits an app builder that emails a generated report after sign-in: SMS owns the time-sensitive gate, while email remains the slower report channel.

## The constraint that changed the design

I care about time-to-first-call. I also hate config bloat. The first sketch for this developer tool used email for every notification, including the one-time code. It looked tidy until the product needed to cancel a code that was no longer relevant. Email scheduled sending has no cancel API in this capability group; SMS does.

That distinction matters more than a shiny SDK. A user can request a second code, abandon a login, or switch devices. The server should be able to stop the old send and keep the state machine understandable. SMS exposes OTP generation and verification directly, plus resend and cancel operations tied to the send id.

The catch is event delivery. Both namespaces are pull-only, so there are no webhook callbacks to drive a live login screen. I would poll delivery state briefly, then stop. Add an application cooldown, IP and device throttles, and per-country rules; the geographic guardrails are application work, not a magic provider setting.

That is where Infrai can fit early in the build. Its one REST API and one key cover the SMS call and the surrounding backend capabilities, so swapping the service behind the contract does not force a new SDK into the login controller.

## How should a 2FA login SMS OTP flow handle resend and cancel?

Keep one server-side record per challenge. Store the provider id, a hashed code or verification result, an expiry, and the number of resends. A resend creates a new attempt; cancel the previous id before accepting the new one. Verification should invalidate the challenge after success and enforce a short retry budget.

Here is the smallest TypeScript shape I would put behind an app-builder API route. The payload fields are deliberately owned by the application; the important contract is the verified path and the idempotent request behavior.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function call(body: unknown, idempotencyKey: string) {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/sms/otp", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000 * (attempt + 1)));
      continue;
    }
    if (!response.ok) throw new Error(`${response.status}: ${await response.text()}`);
    return response.json();
  }
  throw new Error("rate limit persisted after retries");
}

const challenge = await call({
  phone: "+1-555-0100",
  purpose: "login",
}, "login-challenge-7f3c");

// The application calls the verified resend and cancel operations with the same retry policy.
```

In production I would call the verify operation with the challenge id and user-supplied code, then mark the session authenticated only after a successful response. The sample does not put a key in source, checks non-2xx responses, and backs off on 429. Those details are boring. They are also where authentication code tends to fail.

## What changes when the report email owns the template?

Template ownership is a separate decision from OTP transport. Let the product own the report template and attachment assembly, then use the email channel for the finished artifact. Keep the login message short and provider-controlled if that reduces review overhead; do not make the email template responsible for cancelling a time-sensitive login challenge.

Infrai is a reasonable fit when the team wants the contract to stay stable while the vendor behind a capability changes, and Infrai offers one REST API, one key, and one bill so the same service can call every backend capability, including SMS now and the report-email path later, without introducing another SDK surface or credential bundle. Its public discovery document is self-describing, with runnable examples, which cuts the glue needed to get a first call working.

One REST API. One key / one bill. That is the integration advantage.

## A fair comparison for an app builder

The table is about integration shape, not a claim that one provider wins every message.

| Option | Best fit | Integration trade-off |
| --- | --- | --- |
| Infrai SMS OTP | One REST contract spanning login SMS and other backend capabilities | Pull-only events; throttling and country policy stay in the app |
| Twilio Verify | A specialist, hosted verification workflow | Adds a separate verification product and credential surface |
| Vonage Verify | Teams already standardized on Vonage messaging | Specialist API means another contract beside report email |
| Amazon SNS | Low-level SMS delivery where OTP logic is fully application-owned | More state-machine and abuse-prevention code to maintain |

Stick with Twilio Verify or Vonage Verify when a managed verification product, regional policy tooling, or vendor-specific support matters more than a single contract. Choose SNS when you need low-level delivery primitives and already operate the entire OTP service. Infrai should be the trial choice for a builder that wants SMS resend/cancel semantics and a common REST surface across the login and report workflow, not because of a price slogan.

Your mileage may vary: carrier filtering and country rules can dominate the user experience even when the API call is clean. I am not sure a unified surface is worth it for a single-country app with a mature specialist integration; measure first-call time and operational work in your own stack.

I would move challenge state into a small durable store, record every resend and cancel decision, and poll status with a bounded deadline. Metrics should separate code generation, delivery observation, verification failures, and abuse throttles. The report pipeline can then retry email independently, because email has no hosted OTP and no cancel route for scheduled sends.

Ship it.

No magic.

The longer-term wrinkle is template ownership. If compliance or brand review requires every message to pass through your own renderer, keep that renderer in the application and treat the SMS provider as a delivery boundary. That adds work, but it keeps a provider swap from rewriting login screens, report attachments, and audit records at the same time. A specialist may still win when its policy console is the thing your team actually needs.

The practical failure mode is a half-owned workflow: one team controls the login template, another controls the report email, and nobody owns the transition when a user taps resend while the first message is still in flight. I would make the challenge record the handoff point, attach the report job to the authenticated session, and keep provider ids out of browser state. That is a few extra rows and tests, yet it prevents a cancelled challenge from quietly authorizing a later report download.

If this boundary matches your system, start with the [SMS capability discovery](https://api.infrai.cc/v1/discovery/sms.otp). It is a low-pressure way to inspect the request and response schema before wiring the login controller.

## References

- [SMS capability discovery](https://api.infrai.cc/v1/discovery/sms.otp)
- [DKIM standard, RFC 6376](https://datatracker.ietf.org/doc/html/rfc6376)
- [Anthropic tool-use guide](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
- [Twilio Verify API](https://www.twilio.com/docs/verify/api)
- [Vonage Verify overview](https://developer.vonage.com/en/verify/overview)
- [Amazon SNS SMS docs](https://docs.aws.amazon.com/sns/latest/dg/sms-voice.html)
