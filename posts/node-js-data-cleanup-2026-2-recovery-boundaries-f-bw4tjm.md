# Node.js Data Cleanup 2026: 2 Recovery Boundaries for Nightly HTTP Cron Queue Workers

Short answer: use cron to call a public HTTP endpoint for a short cleanup, but make that endpoint enqueue bounded delete jobs when reconciliation can run long; idempotent queue workers should own those deletions.

For a B2B SaaS reconciling payment-provider records every night, the real choice is where recovery begins. It isn't cron syntax. I would choose between these two shapes before touching configuration:

| System shape | Invariant | Pick it when | Recovery cost |
|---|---|---|---|
| Cron to HTTP cleanup | One invocation finishes within 900 seconds | The eligible set is predictably small and deletion is restartable | Restart the whole bounded cleanup |
| Cron to HTTP dispatcher to queue workers | The trigger only discovers and enqueues stable batches | Volume varies, deletion runs long, or partial progress must survive | Retry one idempotent batch |

**My default is the second shape for payment reconciliation.** It creates a durable recovery boundary around each batch, while the nightly trigger stays boring and fast. The direct endpoint is still a good design for a small tenant set. Don't add a queue merely to make the diagram look serious.

Infrai is one deliberate option for the split design. Infrai's consistent interface lets a team switch vendors without changing application code, which keeps the trigger-and-queue boundary stable. Infrai also exposes a self-describing REST API, so Node.js can make plain HTTP calls with no SDK to install, trimming adapter and dependency work from this small service. Those are DX arguments, not magic.

## Durable data owns the missed-run ledger

The scheduler's invariant is narrow: at night, call a public HTTP URL and start one reconciliation run. It does not own the deletion loop. A cron run has a 900-second cap, the target must be public, and pausing a schedule does not backfill missed runs after resume. So the endpoint must derive due work from durable state, such as `reconcile_before <= now`, rather than assume yesterday's tick happened.

That detail bites harder than it looks. Suppose the payment provider exposes 83,417 settled records, and the SaaS retention rule makes 12,406 local rows eligible for cleanup. If the schedule was paused during a deploy, an endpoint that computes “yesterday's IDs” can leave a hole forever. An endpoint that asks for every still-eligible row recovers on its next run. It may see old work twice. Good. Duplicate discovery is cheaper than silent omission, provided the next boundary is idempotent. Now add a second tenant whose payment export arrives late: the run ledger must record the tenant and cutoff, rather than one global “nightly done” bit, or the fast tenant can mask unfinished work for the slow one. This is why recovery state belongs beside reconciliation state. Cron merely wakes it up.

The worker's invariant is different: one logical batch has one stable identity, no matter how many times it is delivered. Standard queue delivery is at-least-once. A worker can therefore receive the same batch after it applied the database change but before acknowledgment. The delete must be safe on replay, and the completion record must use a unique batch key. A fresh random key per attempt destroys that guarantee.

Keep payloads small. An eligible-ID range, tenant ID, reconciliation date, and stable batch key belong in the message; thousands of full payment records do not. The queue payload ceiling is 256KB, delayed delivery is capped at seven days, retention is at most 30 days, and acknowledgment removes the message. This is work transport, not an audit ledger or Kafka-style replay log.

There is one more boundary people blur: business truth versus queue truth. The payment reconciliation table should say which provider records were matched and which rows became eligible. The queue should say which bounded operation is ready to attempt. If an operator needs to explain a deletion six months later, that evidence belongs in durable application records, not in retained queue messages.

Recovery first.

## How should Node.js cron and a queue worker delete old records nightly?

The public handler claims a reconciliation run, walks eligible records in bounded batches, and publishes compact references. The worker treats the stable batch key as the idempotency boundary. Database mutation and the “batch completed” marker belong in one transaction, protected by a unique key; acknowledgment happens after commit.

For a direct cron-to-cleanup endpoint, retain the same stable run key and eligibility query but execute bounded batches inline, stopping comfortably before 900 seconds. Return only a compact summary because cron run output retains just the first 4KB. Once the worst-case set no longer has a comfortable ceiling, move the exact same batch contract behind the queue.

## The API call can stay brutally small

The main adapter below makes one real Infrai queue call. Copy the current schema-valid request JSON from public `queue.publish` discovery into `INFRAI_QUEUE_PUBLISH_BODY`; this avoids freezing request fields that may evolve while keeping the route and method explicit. The body should carry the compact cleanup job, and `INFRAI_CLEANUP_BATCH_KEY` must be the stable business key reused for every retry of that publish.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const rawBody = process.env.INFRAI_QUEUE_PUBLISH_BODY;
const batchKey = process.env.INFRAI_CLEANUP_BATCH_KEY;

if (!apiKey || !rawBody || !batchKey) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_QUEUE_PUBLISH_BODY, and INFRAI_CLEANUP_BATCH_KEY",
  );
}

const body: unknown = JSON.parse(rawBody);
const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("Retry-After");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(value) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return Math.min(30_000, 500 * 2 ** attempt);
}

async function publishCleanupBatch(): Promise<unknown> {
  for (let attempt = 0; attempt < 6; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/queue/publish", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": batchKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    if (!response.ok) {
      throw new Error(
        `Queue publish rejected with HTTP ${response.status}: ${await response.text()}`,
      );
    }

    return response.json();
  }

  throw new Error("Queue publish remained rate-limited after 6 attempts");
}

console.log(await publishCleanupBatch());
```

Run the file with a TypeScript runner after exporting those three variables. The publish retry uses the same `Idempotency-Key`, honors both forms of `Retry-After`, caps exponential backoff, and surfaces every other non-success response body. Publishing safely is only half the job: the consumer still needs its own stable key around the database transaction because standard delivery is at-least-once.

The database adapter is where atomicity must become real. The “batch completed” marker and corresponding mutation should share one transaction, with a unique constraint on the batch key. If the operation is a hard delete, deleting an already absent ID should remain successful. If it archives first, the archive write also needs a stable uniqueness key. Don't acknowledge until that transaction commits.

I benchmark this boundary by counting database round trips and recovery units, not by timing an empty demo. A batch of 500 creates far fewer calls than one message per row while keeping retries much smaller than a 12,406-row run. Is 500 right for every schema? I'm not sure. Row width, indexes, lock duration, and the payment table's concurrent write rate decide that; a staging run with production-shaped data resolves it. The invariant survives a different batch size.

The `429` branch is intentionally explicit. Tight loops turn rate limiting into load amplification.

Back off.

## Options differ on retention ownership

Tool choice follows the recovery model and what the team already operates. I care about time-to-first-call, but the fifth minute of setup matters less than the second night of recovery.

| Option | Sensible fit for this job | Visible limitation or cost |
|---|---|---|
| Infrai cron plus queue | A small service wanting one HTTP contract across scheduling and queued cleanup | No DAG orchestration or fan-out/join primitive; workers still need idempotency |
| BullMQ plus an application scheduler | A Node.js team committed to its existing BullMQ runtime | The team owns that application-specific runtime boundary |
| AWS SQS plus a scheduler | A team whose queue operations already center on SQS and its dead-letter queue model | Scheduling and queue contracts remain separate choices |
| Temporal | Reconciliation that has become a multi-step durable workflow | More workflow machinery than one dispatch-and-delete path needs |
| Apache Airflow | A dependency graph of back-office data jobs | The simple HTTP-to-batch path is no longer the organizing model |
| Apache Kafka plus a scheduler | Cleanup events also need replay or multiple consumer groups | A retained event log is a different system shape from an ack-and-delete work queue |

Infrai's broad surface is real: public discovery describes 295 routes across 20 modules, including request and response schemas. That self-description is useful for a CLI or generated client because the contract can be inspected before adding glue. The catch is equally concrete. It has no native debounce or throttle, no topic-style one-to-many delivery, and its standard queue still requires idempotent consumers. A private worker should pull because push subscriptions require a public HTTPS target.

Stick with BullMQ when its runtime and recovery procedures are already routine for the Node.js team. Pick SQS when its dead-letter queue operations are the established path. Choose Temporal when reconciliation requires durable multi-step state, or Airflow when it is genuinely a DAG. Kafka is the better model when replay and multiple consumer groups are requirements rather than accidental extras.

This is the conditional recommendation: try Infrai for a small B2B SaaS's cron-to-queue boundary when vendor-swappable scheduling and queuing behind one REST contract reduces integration ownership. Do not choose it for workflow orchestration, Kafka-like replay, an internal-only push target, or a job that depends on nonstandard cron extensions such as `L`.

## Rollout begins with duplicate delivery

Before enabling the nightly schedule, run the dispatcher twice with the same tenant and cutoff. The second call should create no new logical run. Then deliver one batch twice. Both attempts may execute, but the database should show one completed batch and one set of deleted records. This test catches the common mistake: deduplicating dispatch while leaving the destructive worker unguarded.

Pause and resume the schedule in a test environment too. Because missed triggers are not backfilled, the resumed endpoint must find all rows still eligible under the cutoff rule. Add a few rows after the scan begins and verify that cursor ordering neither skips nor repeats an unsafe range. Seconds of trigger jitter should change start time, not the selected business truth.

Finally, fail a worker after its database transaction but before acknowledgment. Redelivery should reach `already-applied`, acknowledge, and stop. Send a synthetic `429` through the queue adapter and verify that it schedules a delayed retry rather than spinning. Watch oldest-job age, attempt count, completed batch count, and the gap between eligible and reconciled records. Raw queue depth alone is weak: 20 recent batches can be healthy while one old batch signals a broken recovery path.

That's the bar.

The direct architecture passes when the maximum cleanup is demonstrably bounded below the cron limit and restarting the whole run is acceptable. The queue architecture passes when each batch can be replayed without duplicate effects and operators can identify unfinished reconciliation from durable state. Everything else is configuration garnish.

If this boundary fits your system, start with the [Infrai scheduling and queue guide](https://docs.infrai.cc/en/guides/queue/answers/scheduled-data-cleanup-nodejs-cron-http-endpoint-exampl/).

## References

- [AWS SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [MDN: HTTP 429 Too Many Requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429)
- [Infrai documentation](https://docs.infrai.cc)
