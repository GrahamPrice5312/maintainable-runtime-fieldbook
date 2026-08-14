# Node.js Event Notifications: Email, SMS Opt-Outs, Preferences, and Suppression Lists

Short answer: keep user channel preferences and suppression state in your application, render the order-receipt template from a versioned source you own, and put email or SMS delivery behind an idempotent queue. A provider should transport a decision your system has already made. It should not decide whether a healthtech customer is allowed to receive a receipt.

That boundary matters after payment settles. The receipt is operational, but it still contains personal and order information. A retry can duplicate it. A stale preference can violate consent. A blanket suppression list can hide a useful transactional message. The event notification system needs an explicit policy before it needs a second channel.

## Data governance starts with the settled-order event

An order receipt should begin with a durable business event, not with an HTTP call to an email or SMS service. The settlement transaction records the order outcome and an outbox row in the same database commit. A publisher moves that row to a queue. The notification worker then resolves the current channel policy, loads the approved template version, and records its decision. This gives the team a chain it can inspect: settlement, command creation, policy evaluation, rendering, transport acceptance, and later delivery evidence.

Start there.

The failure chain is worth spelling out because the individual steps look harmless. Payment settles, the API process returns before the event is published, and the receipt never enters the queue. Or the event is published twice, two workers read the same preference, and both send. Or the first remote request is accepted, the worker loses its database connection before saving the remote ID, and the retry creates a second receipt. Or a user opts out while a command is waiting, but the worker trusts the decision made at enqueue time. Each failure has a different repair: an outbox for publication, an idempotency key for duplicate commands, a durable accepted-state transition for transport ambiguity, and a just-in-time policy check for consent. Combining them into a generic `sendReceipt()` helper hides the useful distinctions. I would rather have four explicit states and boring records than one clever abstraction that cannot explain why a customer received two messages.

That is the part teams tend to skip. It is also the part that saves the incident review.

## How should a Node.js event notification system handle email, SMS opt-out, and suppression lists?

Start with one immutable notification command. It should contain an order reference, a user reference, the event type, the selected channel, a template version, and an idempotency key. It should not contain more personal data than the worker needs. Resolve the recipient and policy close to delivery, because a user may opt out between payment settlement and queue consumption.

For an order receipt, I use a preference decision with three distinct outcomes: allowed, opted out, and suppressed. An opt-out is a user or consent choice for a channel. Suppression is a delivery-safety state, such as an address or phone number that must not be attempted. They may lead to the same skip action, but they need different audit reasons and different recovery paths.

Do not model this as a single `marketingOptOut` boolean. Transactional email and promotional email have different purposes, and SMS consent rules can depend on jurisdiction and message type. CTIA's messaging guidance is a useful compliance input, but it is not a substitute for counsel or the rules that apply to your deployed program.

The smallest policy function can stay boring:

```ts
type Channel = "email" | "sms";

type NotificationCommand = {
  idempotencyKey: string;
  eventId: string;
  userId: string;
  orderId: string;
  channel: Channel;
  templateVersion: string;
};

type DeliveryDecision =
  | { action: "send" }
  | { action: "skip"; reason: "opted_out" | "suppressed" };

type PreferenceRecord = {
  emailReceipt: boolean;
  smsReceipt: boolean;
  suppressedEmail: boolean;
  suppressedPhone: boolean;
};

function decideDelivery(
  command: NotificationCommand,
  preferences: PreferenceRecord,
): DeliveryDecision {
  const optedOut = command.channel === "email"
    ? !preferences.emailReceipt
    : !preferences.smsReceipt;

  const suppressed = command.channel === "email"
    ? preferences.suppressedEmail
    : preferences.suppressedPhone;

  if (suppressed) return { action: "skip", reason: "suppressed" };
  if (optedOut) return { action: "skip", reason: "opted_out" };
  return { action: "send" };
}
```

The order of checks is deliberate. A suppression state is a transport safety barrier; it should not be cleared merely because a user turns a preference back on. Clearing it needs an explicit process, such as a confirmed address change or a reviewed appeal. Every skip should be observable without putting the full email address, phone number, receipt body, or payment details in general logs.

## An implementation example: template ownership changes the architecture

The primary decision is not email versus SMS. It is who owns the message contract.

For a healthtech receipt, the application should own the event schema, template source, locale, template version, and the rule that chooses email or SMS. A delivery service can own transport state, provider message IDs, and channel-specific mechanics. This keeps the receipt tied to the settled order rather than to an editable dashboard template that can change without an application release.

There is a useful exception. A support or operations team may need to edit copy without deploying code. In that case, store templates as reviewed records with an immutable version, approval metadata, a preview, and a rollback pointer. Do not let a mutable template be selected halfway through a retry. Resolve the version before enqueueing and record it on the command.

The SMS representation needs its own constraint. A receipt that fits comfortably in HTML email may be too long or ambiguous in a text message. Keep the SMS body short, identify the order without exposing unnecessary clinical information, and provide a preference-management path that can be authenticated. Never assume that an email unsubscribe automatically proves SMS consent has changed.

Template ownership also affects testing. Snapshot the rendered receipt for each locale and channel, then test the policy separately from rendering. I benchmark time-to-first-call, but I also measure the less glamorous numbers: queue age, skip rate by reason, duplicate rate, and the percentage of sends that have a template version. A fast API with no audit trail is not a fast system to operate.

## Failure handling is where delivery correctness lives

Payment settlement should publish an event transactionally with the order state, or use an outbox that is committed with it. A worker then turns that event into one or more notification commands. The send operation must be idempotent at the command boundary, because a worker can crash after the remote service accepts a request and before the local acknowledgement is stored.

```ts
type SendAttempt = {
  command: NotificationCommand;
  attempt: number;
  nextAttemptAt: Date;
};

type TransportResult = {
  accepted: boolean;
  remoteId?: string;
  retryAfterSeconds?: number;
};

async function deliver(
  attempt: SendAttempt,
  send: (command: NotificationCommand) => Promise<TransportResult>,
): Promise<"accepted" | "retry" | "discard"> {
  const result = await send(attempt.command);

  if (result.accepted) return "accepted";
  if (attempt.attempt >= 5) return "discard";

  const advised = result.retryAfterSeconds;
  const exponential = Math.min(16, 2 ** attempt.attempt);
  const delay = advised ?? exponential;
  const jitter = Math.random() * 250;

  await new Promise((resolve) => setTimeout(resolve, delay * 1000 + jitter));
  return "retry";
}
```

This example is an orchestration shape, not a vendor request. The adapter should add the provider-specific HTTP method, endpoint, authentication, and payload. Keep that code behind `send`; the policy and receipt contract should not know which transport is in use. Every retry reuses `idempotencyKey`. A new network attempt is not a new business notification.

There is a small trap here: waiting inside a worker can waste a consumer slot. At modest volume that may be acceptable. At higher volume, persist `nextAttemptAt` and release the job so the queue can schedule it later. Honor a remote `Retry-After` value if the transport supplies one, add jitter so a cohort does not wake together, and stop retrying after a useful deadline. A receipt that arrives long after the user's support workflow may be technically delivered and operationally confusing.

I treat HTTP 429 as flow control, not as a reason to increase concurrency. I also do not retry invalid input or authentication failures. Your mileage may vary on the attempt ceiling; it depends on queue age, the settlement event's freshness, and the transport's rate window. I'm not sure a universal number exists, but five attempts is a reasonable test fixture for this state machine, not a production promise.

## How can delivery records explain email and SMS decisions?

Record enough to explain a decision later. A delivery record might include the event ID, command ID, channel, template version, policy result, suppression reason, attempt count, timestamps, remote message ID, and a redacted recipient reference. Keep the raw response out of normal application logs unless it has been scrubbed and access-controlled.

The state machine should distinguish `queued`, `skipped`, `accepted`, `delivered`, `failed`, and `expired`. `accepted` means the transport accepted the message. It does not mean a person saw it. Email and SMS providers expose different post-acceptance signals, so the reconciler needs channel-specific adapters and a shared internal vocabulary. Do not let a delayed delivery status enqueue another receipt.

At minimum, I want these fields on the command and delivery record:

- `eventId`, `orderId`, and `userId` for correlation without copying the receipt into logs.
- `channel`, `templateVersion`, and the policy result for reconstructing the decision.
- `idempotencyKey`, `attemptCount`, `nextAttemptAt`, and `remoteId` for retry and transport diagnosis.
- `createdAt`, `acceptedAt`, `completedAt`, and a redacted recipient reference for operational timelines.

That list is intentionally less exciting than a provider feature matrix. It is more useful.

An opt-out event is also a normal domain event. When a user opts out, update the preference record and audit the effective time. A command already queued before that time should re-evaluate policy immediately before transport. If it was already accepted, the application cannot pretend a later preference change unsent it; it can prevent future sends and follow the relevant legal and support process.

Suppression handling deserves a separate path. Permanent suppression should normally discard a command with a clear reason. Temporary transport failure belongs in retry. A user who changes an address or phone number should create a new recipient identity or verified destination, rather than silently mutating the history of the old one. That distinction makes incident review possible.

## Rollout trade-offs shape the final channel design

Use email when the receipt needs detail, formatting, or a durable record. Use SMS when the user has explicitly allowed the channel and the message can be short without leaking sensitive information. Offer both only when each channel has its own preference, suppression, and audit path. More channels create more state, not just more reach.

| Decision | Prefer email | Prefer SMS |
| --- | --- | --- |
| Content | Detailed receipt or durable record | Short status and order reference |
| Consent | Separate email preference | Separate SMS preference and applicable consent review |
| Failure handling | Reconcile delayed delivery signals | Treat carrier and destination states as channel-specific |
| Privacy | Avoid unnecessary clinical or payment data | Keep the message minimal and authenticated |

The approach here is not suitable when a team needs a fully managed visual template editor with nontechnical authors and cannot operate approvals, versioning, and rollback. It is also a poor fit when the organization requires synchronous delivery confirmation before the payment request can complete; notification delivery is asynchronous by design. Stick with a specialized workflow that provides those contracts when they are hard requirements.

The portable choice is an internal notification command plus a narrow transport adapter. That adds some code up front. In return, a change in provider, template store, or queue does not rewrite settlement logic or consent records. A single REST API can reduce SDK and credential sprawl for a small team, but it does not remove ownership of preferences, suppression, rendering, compliance review, or observability.

Ship one receipt flow first. Test duplicate delivery, opt-out races, suppressed destinations, a crash after remote acceptance, template rollback, SMS length, and a 429 response. Then measure the worker under representative queue pressure. The design is doing its job when every send or skip has a reason that an engineer can reconstruct without guessing.

## Further reading

- Resend official documentation: https://resend.com/docs/introduction
- CTIA messaging interoperability and compliance best practices: https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
