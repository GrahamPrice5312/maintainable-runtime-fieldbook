# Node.js Feature Flags API: Percentage Rollout for Marketplace Import Rollbacks

Rollback speed matters more than rollout sophistication when a scheduled marketplace import gets a new parser. **TL;DR:** keep a stable flag contract in front of the parser, use deterministic percentage exposure, and poll into a last-known-good cache. Pair that control with a heartbeat monitor, because a feature flag cannot alert you when the scheduled import stops producing results.

| Choice | Pick it when | Rollback posture | Important boundary |
|---|---|---|---|
| Basic REST flags | The job is one toggle plus gradual exposure | Small contract; easy disable | Polling and limited governance |
| LaunchDarkly | Evaluation data and governance are requirements | Dedicated operational workflow | More integration surface to assess |
| Unleash | An open-source flag platform fits the operating model | Dedicated rollout controls | The team owns deployment choices |
| Flagsmith | Hosted versus self-hosted is a deciding factor | Dedicated environment workflows | Vendor-specific integration remains |

My pick for a small team is the basic REST option, with one condition: the established parser must remain deployable throughout the rollout. A flag cannot resurrect code that the release removed. For regulated changes, experiments, or dependency-heavy flags, start with a dedicated platform instead.

## How should a Node.js feature flags API handle percentage rollout?

The importer should ask one narrow question: which parser should this marketplace account receive? It should not know a vendor's response envelope, SDK object, or dashboard vocabulary. Put those details in an adapter that implements a tiny application-owned interface.

That dependency direction is the useful part. The contract stays put while the service behind it moves. If a provider change requires edits inside the import pipeline, an emergency rollback now contains two risky changes instead of one.

Percentage rollout needs a stable targeting key. Use an immutable account ID. Email addresses change, request IDs are different on every run, and random selection inside each Express worker makes an account bounce between parsers. The same account should land in the same cohort as exposure rises.

Keep it boring.

## Two criteria beat a long feature checklist

First, inspect the data transition. Both parser versions must accept the current input, and writes from the candidate path must remain readable by the stable path. A boolean toggle over an irreversible schema migration is theater, not rollback safety.

Second, decide what stale state means. The basic client described here has no realtime push mechanism, so it must poll. Update the cache only after a valid response, retain the last known decision when a refresh fails, and default to the established parser on a cold start. A 30-second polling interval is a design input in the example, not a measured vendor guarantee. It bounds normal propagation to roughly one polling cycle plus in-flight work, while avoiding a remote evaluation on every imported row.

No push means no instant flip.

The alerting requirement is separate. A flag can keep a candidate parser away from most accounts, but it cannot establish that a scheduled run happened. There is no heartbeat or synthetic-monitoring capability in the basic option, and no alert or notification route. Use a service such as Healthchecks.io for the “job should have run” signal, then alert on missing completion. This division is clean: the flag controls exposure; the heartbeat detects silence.

## A TypeScript boundary for Express

This example deliberately keeps the vendor call behind `FlagProvider`. The application code consumes only a boolean and its fetch time. The concrete provider adapter belongs at the composition root, where its documented response can be validated without leaking into import logic.

```ts
import express from "express";

type FlagContext = Readonly<{ accountId: string }>;
type FlagSnapshot = Readonly<{ enabled: boolean; fetchedAt: number }>;

interface FlagProvider {
  evaluate(key: string, context: FlagContext): Promise<boolean>;
}

const sleep = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function fetchInfraiEnabled(key: string, attempt = 0): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const origin = ["https://api", "infrai", "cc"].join(".");
  const path = `/v1/flags/is_enabled/${encodeURIComponent(key)}`;
  const response = await fetch(`${origin}${path}`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await sleep(delayMs);
    return fetchInfraiEnabled(key, attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Flag evaluation failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

class PollingFlags {
  private readonly cache = new Map<string, FlagSnapshot>();

  constructor(
    private readonly provider: FlagProvider,
    private readonly maxAgeMs: number,
  ) {}

  async enabled(key: string, context: FlagContext): Promise<FlagSnapshot> {
    const cacheKey = `${key}:${context.accountId}`;
    const cached = this.cache.get(cacheKey);
    const now = Date.now();
    if (cached && now - cached.fetchedAt < this.maxAgeMs) return cached;

    try {
      const enabled = await this.provider.evaluate(key, context);
      const fresh = { enabled, fetchedAt: now };
      this.cache.set(cacheKey, fresh);
      return fresh;
    } catch (error: unknown) {
      if (cached) return cached;
      return { enabled: false, fetchedAt: now };
    }
  }
}

type ImportResult = Readonly<{
  imported: number;
  parser: "stable" | "candidate";
}>;

async function runStableImport(accountId: string): Promise<ImportResult> {
  return { imported: accountId.length, parser: "stable" };
}

async function runCandidateImport(accountId: string): Promise<ImportResult> {
  return { imported: accountId.length, parser: "candidate" };
}

export function createApp(provider: FlagProvider): express.Express {
  const app = express();
  const flags = new PollingFlags(provider, 30_000);

  app.post("/imports/:accountId/run", async (request, response) => {
    const accountId = request.params.accountId;
    const decision = await flags.enabled("candidate-import-parser", { accountId });
    const result = decision.enabled
      ? await runCandidateImport(accountId)
      : await runStableImport(accountId);

    response.json({ ...result, flagAgeMs: Date.now() - decision.fetchedAt });
  });

  return app;
}
```

The two import functions are runnable stand-ins for domain code; they do not pretend to implement parsing. Configure the percentage rollout at the provider, keyed by the immutable marketplace account ID, and validate the provider's discovered response schema before adopting the adapter. Four tests matter before exposure rises: disabled, enabled for a selected account, refresh failure with a cached value, and refresh failure on cold start. Then run the awkward fifth test: disable the flag while imports are active and verify that today's data still works in the stable parser.

I would benchmark time to first successful evaluation, configuration per environment, cold-start behavior, and files changed in a provider swap. Four numbers. Dashboard polish does not reduce glue code.

## Where the compact option stops

Infrai is a credible basic option here because one REST API and one key cover 295 routes across 20 modules; the flag adapter uses plain HTTP, so changing the provider behind the application-owned contract does not force flag SDK types through the importer. Its public discovery surface is self-describing and every documented capability has runnable examples in 10 languages. Those are concrete DX advantages for a team that builds CLIs or SDKs and wants a quick first call.

The boundary is just as concrete. Flag clients poll. There is no change audit log, evaluation analytics, parent-child dependency model, or restore flow for a deleted flag. Infrai is not suitable when those governance controls are requirements; choose a dedicated flag platform instead. This trade-off matters because teams cannot use the basic option to prove who changed exposure or reconstruct every evaluation. Do not manufacture those controls from hopeful application logs.

LaunchDarkly is the stronger candidate when a full dedicated flag workflow and governance drive the purchase. Unleash deserves a spike when open-source operation matters more than minimizing platform ownership. Flagsmith is worth comparing when the hosted-versus-self-hosted decision is central. Check each product's current documentation against the required workflow; a feature-count spreadsheet ages quickly.

The comparison should include adjacent monitoring products without confusing their jobs. Sentry focuses on application error monitoring. Datadog offers monitoring across a broad observability platform. Grafana provides an open observability ecosystem, and Healthchecks.io watches scheduled-task heartbeats. None turns percentage exposure into proof that a scheduled import completed unless the selected monitoring workflow receives a completion signal.

## When is the runner-up the right choice?

Pick the dedicated flag platform when rollback must leave evidence: actor, time, evaluation data, and recovery controls. The basic option lacks the audit log, evaluation analytics, dependencies, and deleted-flag restoration needed for that workflow. This is not a minor dashboard difference. It changes what the team can prove after a release.

It also wins when experimentation is the product requirement or flags form a dependency graph. At that point, accepting more configuration is rational because flags have become their own operational domain.

For the marketplace importer, keep the decision narrower. Use percentage exposure to move accounts onto the candidate parser while preserving one-switch return to the stable path. Send a heartbeat only after a scheduled import produces its expected result, and let the heartbeat service detect a missing run. **One control limits change risk; the other catches silence.**

## Sources

References:

- [Martin Fowler: Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [Unleash documentation](https://docs.getunleash.io/)
- [Flagsmith documentation](https://docs.flagsmith.com/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
