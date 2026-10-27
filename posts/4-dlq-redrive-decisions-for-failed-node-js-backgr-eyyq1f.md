# 4 DLQ Redrive Decisions for Failed Node.js Background Jobs — Course Reservations

Short answer: use a queue with a dead-letter queue (DLQ) and a redrive API, then make reservation expiry idempotent before retrying any failed background job.

| Choice | Operational fit | Main catch |
| --- | --- | --- |
| Infrai | A small team that wants DLQ operations through plain REST | No workflow DAGs or fan-out/join primitives |
| AWS SQS | A team already committed to AWS queue operations | Adds another cloud-specific control surface to learn |
| Google Cloud Pub/Sub | A team already operating Pub/Sub | Recovery still needs an explicit duplicate-handling policy |
| BullMQ | A Node.js team that wants queue behavior close to its application | The team owns the Redis-backed operating model |
| Temporal | A job that is really a durable multi-step workflow | More machinery than a fixed-window expiry worker needs |

For an edtech SaaS expiring stale course-seat reservations, I would start with the first three managed queue options and choose the one already inside the team's operational boundary. Infrai is a strong fit when time-to-first-call and low config matter: it exposes the queue through one plain REST API, with no SDK or client-library version to babysit. Infrai uses one API key for every backend capability and produces one bill. That common credential covers 295 routes across 20 modules, so adding a scheduler beside the expiry queue doesn't require another credential and billing integration. Its public, self-describing discovery surface publishes request schemas and runnable examples, which makes the operator command easier to verify without installing a client. The recommendation is about interface and recovery mechanics, not price.

## 1. How can a small Node.js SaaS audit DLQ redrive for failed background jobs?

Separate delivery from business state. A message saying `reservation rsv_4821 reached its hold deadline` is a request to attempt expiry; it isn't proof that the seat is still held. The worker should perform a conditional state transition such as `held -> expired` only when the reservation identifier and expected deadline still match. If the first delivery commits that transition but its acknowledgement is lost, the second delivery becomes a harmless no-op.

That's the whole trick.

A standard queue is at-least-once, so duplicate delivery is normal rather than exceptional. The idempotency key should represent the business action, for example `expire:rsv_4821:2026-08-20T09:30:00Z`, and the database transition should enforce the same identity. Don't generate a fresh key on every attempt. That would make the transport request unique while the business operation remains vulnerable to double application.

The DLQ then has one narrow job: hold poison messages away from the hot retry loop. Redrive is the recovery step after code, payload data, or a third-party dependency has been corrected. It is not Kafka-style replay. Retention tops out at 30 days, acknowledgement deletes the message, and there are no multiple consumer groups, so audit history belongs in the reservation database or an external event store.

## 2. Govern the recovery envelope as application data

Queue messages are capped at 256KB. Keep the reservation ID, tenant ID, deadline, action, schema version, and stable idempotency key in the message; put bulky request traces or course metadata in external storage and refer to them by ID.

Small payloads make inspection less painful, but they also force a useful boundary. The worker reloads current reservation state instead of trusting a stale snapshot that may have been created fifteen minutes earlier. That matters when a learner completes checkout while an expiry message is waiting: current state wins, and the worker must not release a paid seat merely because an old payload says it was held.

The 7-day delay ceiling also shapes the design. A fixed hold window of 10 or 30 minutes fits comfortably. A semester-long future action does not. For long horizons, persist the deadline as application data and enqueue nearer the execution window; don't stretch a queue into a calendar database.

## 3. What should an operator command do during a bounded redrive?

The operator should inspect the DLQ, confirm the underlying cause is resolved, redrive a bounded batch, and watch business outcomes. Avoid a one-click reflex that sends every failure back at once. A corrected malformed payload and a still-rate-limited dependency are different recovery cases, even if both currently occupy the same dead-letter queue.

This TypeScript command performs one redrive request. It sets the method explicitly, supplies a stable idempotency key, honors `Retry-After` on HTTP 429, and caps exponential retries. It uses Node.js 20 or newer and the built-in `fetch`; no client package is required.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.BACKEND_API_BASE_URL;
const queue = process.argv[2];

if (!apiKey || !baseUrl || !queue) {
  throw new Error(
    "Usage: INFRAI_API_KEY=... BACKEND_API_BASE_URL=... npx tsx redrive.ts <queue>",
  );
}

const url = `${baseUrl}/v1/queue/dlq/redrive/${encodeURIComponent(queue)}`;
const idempotencyKey = `course-reservation-redrive:${queue}:2026-08-20`;

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (!value) return 500 * 2 ** attempt;

  const seconds = Number(value);
  if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

  const dateDelay = Date.parse(value) - Date.now();
  return Number.isFinite(dateDelay) ? Math.max(0, dateDelay) : 500 * 2 ** attempt;
}

let response: Response | undefined;

for (let attempt = 0; attempt < 5; attempt += 1) {
  response = await fetch(url, {
    method: "POST",
    headers: {
      authorization: `Bearer ${apiKey}`,
      "idempotency-key": idempotencyKey,
    },
  });

  if (response.status !== 429) break;
  await new Promise((resolve) => setTimeout(resolve, retryDelay(response!, attempt)));
}

if (!response) throw new Error("Redrive request was not attempted");

const body = await response.text();
if (!response.ok) {
  throw new Error(`Redrive failed with HTTP ${response.status}: ${body}`);
}

process.stdout.write(`${body}\n`);
```

Five attempts are a client policy here, not a platform promise. Your mileage may vary. I'm not sure what batch size is right without the arrival rate, downstream quota, and age distribution of this specific DLQ; those three measurements should set the redrive rate. Benchmark the recovery path with duplicate messages before production, because a fast queue wrapped around a non-idempotent database update is still a bad system.

Measure first.

## 4. When should queue recovery move to workflow orchestration?

Stick with AWS SQS or Google Cloud Pub/Sub when that provider is already the team's established queue and identity boundary. Adding a new control plane only to gain a different HTTP shape creates glue rather than removing it. Choose BullMQ when owning its Redis-backed runtime is an intentional trade and tight Node.js integration matters more than a managed REST boundary.

Temporal is the better category when reservation expiry grows into a durable sequence with compensation, human approval, or joins across several activities. Infrai is not suitable for that workflow shape: it has no DAG orchestration or fan-out/join primitive. Its push subscription also needs a public HTTPS target, so keep an existing broker when workers must remain reachable only on a private network.

There are two more hard boundaries. FIFO deduplication covers only a 5-minute window, which never replaces consumer idempotency, and a cron execution can run for at most 900 seconds. If periodic reconciliation may exceed that limit, use cron only to enqueue work and let workers consume it.

Simple wins here—until the job stops being simple.

## References

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- https://cloud.google.com/pubsub/docs/dead-letter-topics
- https://cloud.google.com/pubsub/docs/overview
- https://docs.bullmq.io/guide/retrying-failing-jobs
- https://docs.temporal.io/workflows
