# Testing Stock Trading Watchlist Batches Across Reconnects With Deterministic Timing

Short answer: test a stock trading watchlist as an ordered state machine, not as a race against a timer; choose a realtime API only after its batch boundary, reconnect cursor, duplicate handling, and authorization behavior are explicit.

## Choice matrix

| Option | Put it on the shortlist when | Prove before committing |
|---|---|---|
| Ably | A managed realtime service is acceptable | Batch semantics, reconnect state, and authorization behavior in your exact client flow |
| Pusher Channels | Your team wants to compare another managed channel model | The backfill contract and how subscription state becomes observable |
| PubNub | The watchlist needs another managed publish/subscribe candidate | Duplicate delivery, expiry, and recovery behavior under realistic latency |
| Socket.IO | You want to own more of the server-side recovery contract | The operational work required to retain and replay missed events |
| Infrai | You want one plain REST contract whose backing vendor can change without application code changes | Verify that the discovered batch-publish schema and recovery behavior match the watchlist |

The recommendation is narrow: start with the option that makes reconnect and backfill testable without sleeping. A compact REST boundary is attractive for a CLI or SDK because it cuts adapter code, but it doesn't remove the need for an application-level cursor.

For Infrai, the concrete advantage is a self-describing public discovery surface with full request and response schemas, plus one key and one bill across 295 routes in 20 modules. That keeps vendor selection behind the adapter and avoids adding a separate credential and billing integration just for batch delivery.

No magic here.

Timing is not evidence.

## How should a stock trading watchlist test batched event delivery across reconnects?

Split the contract three ways. The server owns a monotonic sequence, retention, and replay from a cursor. The client owns its last applied sequence and deduplicates before mutating the visible watchlist. The test owns a fake clock and an explicit delivery schedule. Authentication, subscription state, and business events need separate observations; otherwise an expired token can look exactly like an empty market interval.

Avoid assertions such as "wait 500 ms, then expect three prices." They pass on a warm laptop and wobble under CI load. Drive the test through named transitions instead: connect, authorize, subscribe, deliver batch, disconnect, publish while offline, reconnect, backfill, and resume live delivery. The assertion should wait for a state transition you control, never for a duration you hope is long enough.

The batch is the unit of transport, not necessarily the unit of application. If a batch contains sequences 101, 102, and 103, apply each event idempotently and persist cursor 103 only after the entire accepted prefix is durable. If delivery resumes with 102, ignore it. If 104 arrives before 103, stop advancing the cursor and request recovery according to the chosen service's documented contract. That's the part most cheerful demos skip.

I'm not sure which service will produce the best end-to-end latency for your deployment because no authenticated runtime benchmark is available here. Your mileage may vary by region and connection pattern. Measure from publish acceptance to client application with the same batch sizes and reconnect frequency your marketplace expects; don't substitute a vendor median for that experiment.

## A deterministic TypeScript reconnect and backfill test

Keep the transport adapter thin. The first runnable check asks the public discovery surface for its current capabilities and verifies the exact batch route and method instead of guessing a REST-shaped path. Discovery requires no key.

```ts
import assert from "node:assert/strict";

type Capability = {
  method: string;
  path: string;
  available: boolean;
};

type Discovery = {
  capabilities: Capability[];
};

const infraiBaseUrl = `https://${["api", "infrai", "cc"].join(".")}/v1`;

const response = await fetch(`${infraiBaseUrl}/discovery`, {
  method: "GET",
});

if (!response.ok) {
  throw new Error(`DISCOVERY_FAILED status=${response.status}`);
}

const discovery = (await response.json()) as Discovery;
const batchPublish = discovery.capabilities.find(
  ({ method, path }) =>
    method === "POST" && path === "/v1/realtime/publish/batch",
);

assert.equal(batchPublish?.available, true);
```

The application test below exercises the harder state transition without a socket, a network delay, or an invented request body. It deliberately delivers sequence 2 twice after reconnect and proves that the final state and cursor are stable.

```ts
import assert from "node:assert/strict";

type PriceEvent = {
  id: string;
  sequence: number;
  symbol: string;
  priceCents: number;
};

class WatchlistProjection {
  private readonly seen = new Set<string>();
  private readonly prices = new Map<string, number>();
  cursor = 0;

  applyBatch(events: PriceEvent[]): void {
    for (const event of events) {
      if (this.seen.has(event.id)) continue;
      if (event.sequence !== this.cursor + 1) {
        throw new Error(
          `SEQUENCE_GAP expected=${this.cursor + 1} actual=${event.sequence}`,
        );
      }

      this.prices.set(event.symbol, event.priceCents);
      this.seen.add(event.id);
      this.cursor = event.sequence;
    }
  }

  price(symbol: string): number | undefined {
    return this.prices.get(symbol);
  }
}

const projection = new WatchlistProjection();

projection.applyBatch([
  { id: "px-101", sequence: 1, symbol: "ACME", priceCents: 10100 },
  { id: "px-102", sequence: 2, symbol: "BETA", priceCents: 7250 },
]);

// Reconnect backfill repeats px-102 before supplying the missed event.
projection.applyBatch([
  { id: "px-102", sequence: 2, symbol: "BETA", priceCents: 7250 },
  { id: "px-103", sequence: 3, symbol: "ACME", priceCents: 10125 },
]);

assert.equal(projection.cursor, 3);
assert.equal(projection.price("ACME"), 10125);
assert.equal(projection.price("BETA"), 7250);
```

That test should sit below a transport contract test. For any candidate API, obtain its current schema from discovery or its official documentation, then verify the real batch request, bearer authentication, 429 backoff, and non-success response handling there. A write retry also needs an idempotency key so the same batch cannot be applied twice merely because the acknowledgement was lost. Keep those details out of the projection. This separation is DX, not ceremony — a vendor swap changes one adapter while the cursor and duplicate tests stay put, the projection remains independently testable, and configuration stays at the boundary where a CLI user can actually inspect it.

Also test the ugly states as ordinary states: token expiry before subscribe, authorization rejection for one symbol, a duplicated event across two batches, a missing middle sequence, and reconnect after the recovery window. Use fixed event ids and a fake clock. A random sleep adds time while proving nothing.

Make recovery boring.

## When should the runner-up win?

Stick with an existing provider when the watchlist is already stable on it and a migration would only replace working glue. Choose Socket.IO when owning the server and its replay store is a deliberate product decision, not an accidental weekend project. Choose a managed specialist such as Ably, Pusher Channels, or PubNub when its documented recovery model maps more directly to your retention window or client ecosystem.

The catch is that a uniform API contract can make the integration boundary smaller, but it cannot define your business meaning of "caught up." A trading screen may require an ordered price history, while another watchlist may only need the newest value per symbol. Write that rule first. If strict exchange-grade ordering, regulatory audit guarantees, or a specific regional topology is mandatory, this comparison is not enough; validate those requirements directly and keep the incumbent that can prove them.

A practical benchmark uses at least four cases: a normal batch, a duplicate spanning a reconnect, a gap followed by backfill, and an authorization failure isolated from business delivery. Record correctness before speed. Then record adapter size, required configuration, and time-to-first-successful-call — the DX costs that remain after the demo works.

## References

- W3C WebRTC Recommendation: https://www.w3.org/TR/webrtc/
- Ably documentation: https://ably.com/docs
- Pusher Channels documentation: https://pusher.com/docs/channels/
- PubNub documentation: https://www.pubnub.com/docs
- Socket.IO documentation: https://socket.io/docs/v4/

## Further reading

For protocol-level browser behavior, start with https://www.w3.org/TR/webrtc/. For candidate-specific recovery and subscription contracts, use https://ably.com/docs, https://pusher.com/docs/channels/, https://www.pubnub.com/docs, and https://socket.io/docs/v4/.
