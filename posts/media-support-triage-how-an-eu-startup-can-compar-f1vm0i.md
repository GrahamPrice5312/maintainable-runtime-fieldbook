# Media Support Triage: How an EU Startup Can Compare Speech-to-Text API Pricing Per Minute

The cheapest speech-to-text API for a startup is the one that remains cheap after short clips, retries, and tenant-level accounting are included. For a media support queue, I would choose by measured cost per resolved ticket, then use transcription quality and delivery behavior as hard gates. A raw per-minute quote is only an input.

Short answer: keep transcription behind a narrow adapter, attach a tenant id before the first byte is uploaded, and compare current quotes against the actual distribution of support recordings. Do not rank APIs from a single long audio sample or a blended monthly total.

This constraint changes the build. A media company may have one enterprise tenant sending long interview files and hundreds of smaller tenants sending ten-second voice notes. One aggregate average hides who is creating the bill. I want the first successful call quickly, but I also want the first invoice to explain itself.

## The ticket is the unit of truth

Start with an audio manifest, not a provider dashboard. Each row needs a tenant id, clip id, duration in seconds, language, channel count, and the final ticket outcome. Keep the tenant id pseudonymous in test data. Include short voice notes, long recordings, silence, accents, and the languages that the support team will actually receive. The transcript is an input to triage, not the record of billing truth. Store the two records separately and join them with the ticket id.

The billing model belongs in the same manifest review. Providers may differ in minimum increments, rounding, asynchronous options, and how retries are charged. I am not sure a public price page answers all of those questions for a particular contract; dated quotes and a small production-shaped corpus resolve that uncertainty. In practice, the dangerous case is a tenant that sends mostly tiny clips: a six-second note can consume a whole billing unit, while a 247-second recording may be close to its raw duration. If the test set contains only long recordings, the spreadsheet rewards a rate that does not describe the queue. I would export one week of sanitized duration buckets, replay those buckets against every dated quote, and keep the raw and billed seconds side by side. Then I would attach the transcript acceptance result to the same ticket id. That gives support operations a way to ask, "Why did tenant-a cost more?" and get an answer from data rather than a vendor console.

Quality is a gate, not a discount. Measure word error rate on a labeled sample, but also measure whether the transcript contains the account number, product name, and requested action needed for triage. A transcript that looks acceptable to a human can still send a ticket to the wrong queue.

Measure the bill.

## How should an EU startup compare speech-to-text API pricing?

Compare the candidates only after the ticket ledger can answer three boring questions: which tenant uploaded the clip, which attempt produced the transcript, and which transcript led to a resolved ticket. This order matters. A provider dashboard can show aggregate minutes while hiding a retry storm or a short-clip-heavy tenant.

## Build the ledger before the adapter

The first version needs no billing platform. It needs one usage event per audio attempt and one final event for the ticket. The event must record the quoted billing unit, not just wall-clock duration, so a later pricing review can replay the result.

```ts
type UsageEvent = {
  tenantId: string;
  ticketId: string;
  provider: string;
  clipSeconds: number;
  billingUnitSeconds: number;
  usdPerMinute: number;
  attempt: number;
};

function billedSeconds(event: UsageEvent): number {
  if (event.clipSeconds < 0 || event.billingUnitSeconds <= 0) {
    throw new Error("durations must be non-negative and billing unit must be positive");
  }

  return Math.ceil(event.clipSeconds / event.billingUnitSeconds) * event.billingUnitSeconds;
}

function eventCost(event: UsageEvent): number {
  return (billedSeconds(event) / 60) * event.usdPerMinute;
}

function costByTenant(events: UsageEvent[]): Map<string, number> {
  const totals = new Map<string, number>();

  for (const event of events) {
    const current = totals.get(event.tenantId) ?? 0;
    totals.set(event.tenantId, current + eventCost(event));
  }

  return totals;
}

const sample: UsageEvent[] = [
  { tenantId: "tenant-a", ticketId: "t-101", provider: "candidate-a", clipSeconds: 11, billingUnitSeconds: 1, usdPerMinute: 0.01, attempt: 1 },
  { tenantId: "tenant-a", ticketId: "t-102", provider: "candidate-a", clipSeconds: 247, billingUnitSeconds: 1, usdPerMinute: 0.01, attempt: 1 },
  { tenantId: "tenant-b", ticketId: "t-203", provider: "candidate-a", clipSeconds: 6, billingUnitSeconds: 1, usdPerMinute: 0.01, attempt: 2 },
];

console.table([...costByTenant(sample)].map(([tenantId, usd]) => ({ tenantId, usd })));
```

The sample rates are placeholders for the dated quotes under review, not a claim about any provider. The useful property is the accounting boundary: every attempt retains enough information to explain a tenant's total. A retry is visible. A short clip is visible. So is the second minute created by rounding.

I benchmark everything I can reproduce. For the API comparison, record time-to-first-call, authentication steps, adapter lines, transcript acceptance, completion latency, retry count, and cost per resolved ticket. Config bloat is a real tax. Keep it in a separate column instead of pretending it is part of the rate.

## Failure modes hiding in the per-minute quote

Run the same manifest through each candidate: OpenAI, Deepgram, AssemblyAI, and Google Cloud. Treat the names as rows in an experiment, not as a recommendation. The worksheet should preserve the quote date and billing increment alongside the measured output.

| Gate | Evidence to collect | Reject when |
|---|---|---|
| Tenant accounting | Per-attempt usage events and replayable totals | A monthly total cannot be attributed to a tenant |
| Audio fit | Required language and triage-field accuracy | The support action is missed |
| Billing | Current rate, rounding, minimum unit, retry treatment | The quote cannot model the clip distribution |
| Operations | Async completion, idempotency, rate-limit behavior | Duplicate or late results corrupt routing |
| Data review | Processing location, retention, contract terms | The tenant's data requirements are not met |

Then calculate two numbers: billed spend per tenant and billed spend per resolved ticket. The second one catches a bad bargain that the first one misses. If an API has a lower minute rate but produces more manual corrections, the ticket metric exposes the difference without inventing a dollar value for every engineer hour.

No magic ranking.

## When is this setup too much?

Keep the provider response out of the ticket domain. An adapter should emit a transcript, confidence metadata when available, a completion state, and a usage record. The queue should make completion handling idempotent. The same audio event must not become two routed tickets because a webhook was delivered twice.

Watch submitted, completed, retried, rejected, and manually corrected jobs separately. Alert on a tenant whose short-clip mix changes sharply; that can move its billed total even when monthly audio seconds stay flat. Retain the manifest and decision worksheet with the deployment record so a later quote change can be tested against old traffic.

The catch is that per-tenant visibility adds instrumentation and a data review. It is not suitable when the project is a one-off local transcription job with no customer billing boundary. In that case, a simple batch script may be the right tool. Stick with the smallest qualifying setup until there is a real queue, then add the adapter and usage ledger before usage becomes opaque.

Your mileage may vary — language mix, clip shape, and correction policy will move the result. The stable decision rule is narrower: reject candidates that fail the audio, data, or delivery gates; among the survivors, choose the one with the clearest replayable cost per resolved ticket and the least glue code.

## References

- https://github.com/openai/tiktoken
- https://python.langchain.com/docs/integrations/chat/openai/
