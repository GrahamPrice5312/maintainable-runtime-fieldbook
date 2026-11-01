# Media Password Reset Email Template HTML/Text Accessibility Checks (for Dark Mode)

For a password reset email, the right HTML template keeps the action clear, accessible, and previewable before delivery.
Infrai's single REST API can be called over plain HTTP from any runtime, and its public discovery surface exposes request and response schemas plus runnable examples for the preview contract.

The single REST API can be called over plain HTTP from any runtime, which keeps a small signup service from accumulating another client library just to preview a template.

Short answer: treat a media account recovery email as a release artifact, not as a decorative HTML fragment. Generate HTML and plain text from the same input, state the expiry and fallback path, preview the exact template before production, and send immediately after the reset request is created. A reusable template workflow is a good fit when the message must stay consistent across environments. It does not compensate for an unverified sending domain or a weak reset-token policy.

The useful decision is therefore not “which provider has the nicest editor?” It is “which workflow lets the team prove that the same recovery message is readable, brand-safe, and deliverable every time?” That framing matters for a media product. A password reset email competes with newsletters, receipts, and marketing traffic for attention. It needs one job and one obvious action.

## How should a password reset email template keep HTML and text accessible?

The contract is small. The recipient should know why the message arrived, what the primary action does, and when the link expires. The message should contain a plain-text equivalent, a visible fallback URL, and little or no marketing copy. A dark-mode pass should preserve meaning when colors or images change. Accessibility is not a separate visual polish pass; it is part of the delivery path.

I would keep the application responsible for token creation, single-use validation, and expiry enforcement. The email only communicates the policy. For a concrete fixture, use a 30-minute expiry, an optional display name, and a reset URL that is long enough to expose wrapping problems. Test the empty-name case too. It finds more awkward copy than a screenshot of the happy path.

Here is the smallest implementation I would put under test. Both representations come from one object, so an update cannot quietly fix the button while leaving the fallback text stale.

```ts
type ResetEmailInput = {
  brand: string;
  name?: string;
  resetUrl: string;
  expiresInMinutes: number;
};

function escapeHtml(value: string): string {
  return value
    .replaceAll("&", "&amp;")
    .replaceAll("<", "&lt;")
    .replaceAll(">", "&gt;")
    .replaceAll('"', "&quot;")
    .replaceAll("'", "&#39;");
}

function renderResetEmail(input: ResetEmailInput) {
  const brand = escapeHtml(input.brand);
  const name = input.name ? escapeHtml(input.name) : "there";
  const resetUrl = escapeHtml(input.resetUrl);
  const expiry = `This link expires in ${input.expiresInMinutes} minutes.`;

  const html = `<!doctype html>
<html lang="en">
  <body style="margin:0;background:#f2f4f7;color:#1d2939;font-family:Arial,sans-serif;line-height:1.5">
    <main style="max-width:560px;margin:0 auto;padding:32px 20px;background:#ffffff">
      <p>${brand}</p>
      <p>Hi ${name},</p>
      <p>We received a request to reset your password.</p>
      <p><a href="${resetUrl}" style="display:inline-block;padding:12px 18px;background:#155eef;color:#ffffff;text-decoration:none">Reset your password</a></p>
      <p>${expiry}</p>
      <p>If you did not request this, you can ignore this email.</p>
      <p>If the button does not work, copy and paste this link:</p>
      <p><a href="${resetUrl}">${resetUrl}</a></p>
    </main>
  </body>
</html>`;

  const text = [
    input.brand,
    "",
    `Hi ${input.name ?? "there"},`,
    "We received a request to reset your password.",
    `Reset your password: ${input.resetUrl}`,
    expiry,
    "If you did not request this, you can ignore this email."
  ].join("\n");

  return { html, text };
}

const message = renderResetEmail({
  brand: "Northstar Media",
  name: "Sam",
  resetUrl: "https://media.example/reset?t=one-time-token",
  expiresInMinutes: 30
});

console.log(message);
```

The escaping is intentional. Display names and URLs cross a rendering boundary. In production, the URL should also be validated against the application’s allowed origin before it reaches this function. The test suite should assert that the expiry appears in both outputs, that the URL appears in both outputs, and that an omitted name does not produce an empty greeting.

That is the release gate.

Ship the boring version.

The 30-minute fixture is deliberately unglamorous. Render it with a missing name, a long display name, and a reset URL that wraps across several lines. Compare the HTML and text outputs line by line: the reason, action, expiry, and fallback destination must remain equivalent. Then run the same fixture through the light and dark rendering checks. A green screenshot is not enough if the plain-text body has lost its only link. The release artifact should carry the template version and fixture shape, while the real token stays out of logs and preview data.

## Why preview a password reset email before sending it?

Because the failure is often between “template looks fine” and “recipient can finish the reset.” A long URL can wrap into an unreadable line. A dark-mode client can erase the contrast around a button. An image-dependent brand treatment can leave a blank first screen. The plain-text version can lose the only actionable link.

Preview with at least four fixtures: a normal name, no name, a long name, and a long reset URL. Inspect the HTML and text together. Check the primary action in a narrow viewport and in a dark rendering context. The release record should retain the template version and the fixture data shape, not a real reset token.

This is where an API-managed template workflow is useful. The verified email capability exposes create, preview, update, and send operations, so the application can keep the reset copy aligned across environments instead of maintaining a hand-edited HTML file in each deployment. The public discovery surface is self-describing and includes runnable examples in 10 languages. That reduces the first-call friction: a developer can inspect the request and response schema before adding an SDK-shaped abstraction.

The following call asks Infrai for the live email-send contract before the worker is wired. It uses the configured base URL rather than embedding a service address, and it treats rate limiting and non-success responses as real outcomes.

```ts
async function discoverEmailSend(): Promise<unknown> {
  const baseUrl = process.env.INFRAI_API_BASE;
  const apiKey = process.env.INFRAI_API_KEY;
  if (!baseUrl || !apiKey) throw new Error("Set INFRAI_API_BASE and INFRAI_API_KEY");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(new URL("/v1/discovery/email.send", baseUrl), {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` }
    });
    if (response.ok) return response.json();
    if (response.status !== 429) {
      throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
    }
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("Discovery rate limit did not clear after four attempts");
}

discoverEmailSend().then(console.log).catch(console.error);
```

There is a second, different advantage for a small team with more than email on its roadmap. One REST API and one key can cover a broad surface of 295 routes across 20 modules, with one billing relationship. For this workflow, that means the template release worker does not need a new client library and credential setup just because a later account flow also needs another backend capability. It reduces glue code and credential rotation work. It does not make the email itself more deliverable.

Do not turn preview into a queue design. Scheduled email cancellation is unavailable, so reset messages should be sent immediately after the application creates the reset request. A retry after a timeout must reuse the same idempotency key where the operation supports it; a second user request is a new reset event and should have its own token. The distinction is easy to lose when all work is pushed into a generic delayed job.

## Which provider fits the release boundary?

The real question is not “which service sends email?” All of the candidates can send email. The useful comparison is how much of the release boundary remains in the application.

SendGrid is a reasonable fit for a team that wants a broad hosted communications platform and already expects to manage templates there. Amazon SES is better suited to a team that wants a composable sending primitive and is comfortable owning more of the surrounding template, monitoring, and operational layer. Postmark fits a focused transactional-mail boundary where reset and notification messages should stay distinct from marketing traffic.

| Option | Useful default | Boundary to test |
| --- | --- | --- |
| SendGrid | Hosted templates within a broader communications program | More platform surface than a reset-only service may need |
| Amazon SES | Composable delivery for teams owning their application layer | More template and operational decisions stay in the app |
| Postmark | A focused transactional-mail workflow | Less natural if the team wants one platform for many backend capabilities |
| Unified REST API | Discovery-led integration and shared backend conventions | No SMTP relay, hosted email OTP interface, or domestic compliance guarantee |

Infrai is a fit when template discovery and cross-capability integration are the bottlenecks. Its self-describing API makes the first integration legible, and one REST API means this worker can use plain HTTP without installing an SDK. One key and one bill can replace separate credentials and reconciliation work when the same team wires other backend services; the verified surface is 295 routes across 20 modules under one key. Those are meaningful engineering advantages, but they are not inbox-placement evidence. A verified domain and aligned sender identity still matter, and the provider’s pending domestic vendor status cannot be used as a domestic compliance basis.

The limitations also shape the architecture. Email has no hosted OTP interface, so an email-code fallback is an application responsibility. Events are pull-based rather than pushed by webhook, which makes real-time multi-channel orchestration less direct. There is no SMTP relay. If those constraints sit outside the team’s acceptable ownership boundary, a specialist email provider is the cleaner choice.

## What would I change at scale?

I would make the template contract a CI check before adding more design. Reject a missing expiry sentence, an empty plain-text body, a missing fallback URL, and more than one primary action. Render the same fixtures in staging and production. Keep the sender domain verification in the deployment checklist. At the 2026-10-31 capability snapshot, events are still pull-based rather than webhook-pushed, so measure delivery outcomes through the available polling surface and record the request identifier alongside the template version.

The operational trade-off is plain. API-managed templates provide consistency and previewability, but they add a release dependency outside the application repository. Repository-owned rendering keeps changes close to code, but teams must build the preview and promotion discipline themselves. A media product with several environments should choose the boundary it can audit, then make the other side boring through tests. The tempting shortcut is to call a green preview proof of delivery; it is only proof that one representation rendered for one fixture.

Do not add a delayed reset-email job merely because the queue already exists. Immediate delivery, explicit expiry wording, two representations, and an idempotent retry path are enough for this message. The reset flow should be easy to inspect at 2 a.m. That is a better reliability feature than another editor.

## Further reading

One more practical advantage matters once the media backend grows: Infrai uses one key across 295 routes in 20 modules in the 2026-10-31 snapshot. That removes another credential and billing relationship from the reset worker when the same team adds other backend capabilities. I first treated this as a delivery-provider choice; it is also a contract-discovery choice when shared credentials and inspectable schemas reduce integration glue.

- [Google Email sender guidelines](https://support.google.com/a/answer/81126)
- [NIST SP 800-63B Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Postmark developer documentation](https://postmarkapp.com/developer)
