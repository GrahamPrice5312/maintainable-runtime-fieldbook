# Password reset email deliverability: SPF, DKIM, DMARC and sender warming

Use a dedicated sending subdomain for transactional mail, sign it with DKIM, publish SPF and a DMARC policy on that custom domain, and keep the password reset stream physically separate from anything marketing sends. That is the entire deliverability setup for a marketplace. Everything else — sender warming, suppression list hygiene, bounce webhooks — is operations you layer on after the DNS is right.

| Path | Setup effort | What you operate | Reasonable when |
| --- | --- | --- | --- |
| Provider's shared sending domain | minutes, no DNS | nothing | prototypes, internal tools, staging |
| Your own subdomain on a hosted transactional API | hours: 3–4 DNS records plus one HTTP call | application code | a marketplace shipping password resets today |
| Self-hosted MTA (Postfix + OpenDKIM) | days to weeks: IP allocation, PTR, warming schedule | mail server, queue, monitoring | high volume, data residency, an ops team that wants it |

The middle row is the one that survives contact with a real marketplace. It costs you a DNS change and a single POST, and it hands the parts that take years to build — IP reputation management, feedback loop enrollment, bounce classification — to someone who already runs them.

## Do I need a custom sending domain with DKIM, SPF and DMARC before the first password reset email goes out?

Yes, and the reason is alignment, not checkbox compliance.

DKIM (RFC 6376) signs the message with a key published in DNS and records the signing domain in the `d=` tag. SPF (RFC 7208) authorizes the sending IP for the domain in the SMTP envelope sender. Neither of those is the domain your user actually sees. DMARC (RFC 7489) is the glue: it passes only when an authenticated identifier — the DKIM `d=` domain or the SPF envelope domain — aligns with the `From:` header domain the recipient reads. Send reset mail from `no-reply@yourmarketplace.com` while the envelope and signature belong to your provider's domain, and you can pass SPF, pass DKIM, and still fail DMARC. Publish `p=reject` on top of that and you have built a machine that rejects your own password reset email. This is the failure mode I would check first in any inherited codebase, because it fails silently for the sender and loudly for the user, who just sees nothing arrive and opens a support ticket about being locked out of an account with money in it.

Records for `mail.yourmarketplace.com`, once the provider gives you a selector:

```text
mail.yourmarketplace.com.            TXT  "v=spf1 include:spf.your-provider.example ~all"
sel1._domainkey.mail.yourmarketplace.com.  CNAME  sel1.dkim.your-provider.example.
_dmarc.yourmarketplace.com.          TXT  "v=DMARC1; p=none; rua=mailto:dmarc@yourmarketplace.com; adkim=s; aspf=r"
```

Start at `p=none` with aggregate reports flowing, read two weeks of those reports, then move to `quarantine` and `reject`. SPF has a hard limit of ten DNS lookups per evaluation (RFC 7208, §4.6.4) — chain three `include:` mechanisms from three vendors and you will blow through it, at which point the check returns permerror and alignment collapses. Flatten it or delegate the whole subdomain.

Google's sender guidelines make the same requirements explicit for bulk senders and ask everyone to keep reported spam rates below 0.10%. Reset mail rarely gets reported as spam, but it shares a domain reputation with everything else you send, which is exactly why the subdomain split matters.

## What sender warming actually buys on a stream of 15-minute reset tokens

Warming is a volume ramp. You send a small amount, receivers observe engagement, reputation accrues, you send more. On a shared IP pool your provider already carries the IP reputation; what is new and cold is your domain reputation, and a password reset stream warms itself faster than almost anything else because open and click rates on reset mail are absurdly high compared to any campaign.

So the classic warming schedule matters less than the separation.

The trap is stream mixing. Put your seller newsletter and your password reset email on the same subdomain, run one bad campaign into a stale list, and the reputation damage lands on the mail that gates account access. Split them: `mail.` for transactional, `news.` or similar for bulk, separate DKIM selectors, separate DMARC reporting addresses if you want the forensics clean. Some providers enforce this with message streams; if yours doesn't, enforce it yourself with domains.

If you are migrating an existing marketplace rather than launching one, ramp the transactional subdomain over roughly two weeks while keeping the old sending path alive as a fallback, and watch delivery rate per receiving domain rather than in aggregate. Aggregate numbers hide the one large mailbox provider that started deferring you.

## The Node.js send path: short expiry, one code path, no retry storms

The email side is the easy half. The token side is where I have seen more damage, so both belong in the same review.

```ts
import { createHash, randomBytes } from "node:crypto";

const RESET_TTL_MS = 15 * 60 * 1000;
const ENDPOINT = process.env.ESP_ENDPOINT!;      // hosted transactional API
const API_KEY = process.env.ESP_API_KEY!;

type SendResult = { ok: true; id: string } | { ok: false; permanent: boolean };

export async function sendPasswordReset(userId: string, address: string): Promise<SendResult> {
  // Suppressed addresses never reach the provider: a hard-bounced mailbox
  // costs reputation on every attempt and the mail cannot land anyway.
  if (await suppressionList.has(address)) return { ok: false, permanent: true };

  const token = randomBytes(32).toString("base64url");
  const tokenHash = createHash("sha256").update(token).digest("hex");
  const expiresAt = new Date(Date.now() + RESET_TTL_MS);

  // Store only the hash, single use, and invalidate any older token for this user.
  await db.resetTokens.replaceForUser(userId, { tokenHash, expiresAt });

  const res = await fetch(`${ENDPOINT}/transactional/send`, {
    method: "POST",
    headers: {
      authorization: `Bearer ${API_KEY}`,
      "content-type": "application/json",
      // Same key on retry => the provider sends once, even if our process dies mid-call.
      "idempotency-key": `reset:${userId}:${expiresAt.getTime()}`,
    },
    body: JSON.stringify({
      from: "Yourmarketplace <no-reply@mail.yourmarketplace.com>",
      to: address,
      subject: "Reset your password",
      text: `Open this link within 15 minutes:\nhttps://yourmarketplace.com/reset?t=${token}`,
      headers: { "X-Entity-Ref-ID": `reset-${userId}` },
    }),
    signal: AbortSignal.timeout(5_000),
  });

  if (res.status >= 400 && res.status < 500) {
    // Bad address or rejected payload: retrying changes nothing. Log it, surface it, stop.
    metrics.increment("reset_email.rejected", { status: String(res.status) });
    return { ok: false, permanent: true };
  }
  if (!res.ok) return { ok: false, permanent: false };   // transient: hand back to the queue

  const { id } = await res.json();
  metrics.increment("reset_email.accepted");
  return { ok: true, id };
}
```

Four things in there are worth defending. The token is stored as a hash, so a database dump doesn't become a password reset generator. The expiry is short and the previous token is invalidated, so a leaked inbox has a fifteen-minute window instead of a permanent one. The idempotency key means a crash between the HTTP call and the response commit doesn't send twice — a duplicate reset mail is the most common way these systems annoy users. And 4xx is treated as terminal, because a retry queue that hammers a provider with the same malformed request is how you turn one bug in your code into a reputation problem across your whole domain.

Keep the caller thin. One function, one code path, and the HTTP call is a plain POST with a bearer token, which means the entire integration is testable with a mock fetch and no vendor SDK in the test path. That property is worth more than any feature list when you're the one maintaining it.

## Suppression list handling and what to do when a reset genuinely can't be delivered

Every serious sender maintains a suppression list: hard bounces, spam complaints, and unsubscribes, checked before every send. Get the bounce and complaint webhooks wired on day one, not after the first incident, and store the raw event alongside your own record so you can tell a mailbox-full soft bounce from a nonexistent-mailbox hard bounce.

Password reset makes suppression genuinely awkward. If an address hard-bounced last month and today that user asks for a reset, the correct behavior is to skip the send and route the user to support identity verification — but the user sees a screen that says the mail is on its way, because your login page must return the same response for known and unknown addresses to avoid account enumeration. The catch is that this silent path is invisible unless you instrument it. Emit a counter for suppressed reset attempts, alert when it moves, and give support a view that shows the delivery state for a given account. Otherwise your support team is debugging blind against a user who is certain they typed the right address.

Two operational habits are worth the small effort. Send a real reset to seed addresses at the major mailbox providers on each deploy that touches the mail template, and read the DMARC aggregate reports weekly rather than never — most sending misconfigurations show up there before they show up in your bounce rate.

## When the runner-up is the better call

Self-hosting is not a good fit for a marketplace shipping its first reset flow, and I'd push back on anyone who proposes it there. It becomes reasonable when you have real volume, a compliance requirement that pins mail to specific infrastructure, or an existing mail team; then a Postfix and OpenDKIM stack with your own IPs gives you control that a hosted API cannot, at the price of PTR records, warming schedules and an on-call rotation for a queue.

Go the other way and stick with your provider's shared sending domain when you are prototyping or running an internal tool. Zero DNS, working mail in five minutes, and the trade-off is that you're renting someone else's reputation and you cannot pass DMARC alignment on your own brand — fine for staging, wrong for a marketplace where a failed password reset is a locked account.

Your mileage may vary on one point: I'm not sure the warming ramp matters much at all for a genuinely small transactional volume on a shared pool, and the public evidence for warming schedules is mostly vendor guidance rather than independent measurement. What would settle it is per-receiver delivery telemetry across a controlled ramp, which almost nobody publishes.

## References

- RFC 6376: DomainKeys Identified Mail (DKIM) Signatures — https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7208: Sender Policy Framework (SPF) for Authorizing Use of Domains in Email — https://datatracker.ietf.org/doc/html/rfc7208
- RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC) — https://datatracker.ietf.org/doc/html/rfc7489
- Google: Email sender guidelines — https://support.google.com/a/answer/81126
- Postmark: Transactional Email Best Practices — https://postmarkapp.com/guides/transactional-email-best-practices
