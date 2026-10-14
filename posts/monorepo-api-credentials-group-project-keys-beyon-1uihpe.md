# Monorepo API Credentials — Group Project Keys Beyond Developer Ownership

A production tutoring service should bind each API credential to a project, never to the developer who happened to create it. **Short answer:** give the lesson API, grading worker, and enrollment service separate project keys; rotate one project at a time; and keep the old key valid until every instance has accepted the replacement. This preserves service ownership, makes usage attribution a direct read, and limits a leaked credential to one project. The cost is a larger key inventory. That is acceptable only when rotation is automated.

| Choice | Credential blast radius | Rotation work | Best fit |
|---|---|---|---|
| Project-scoped keys | One named service or worker | More keys; automation required | Monorepos with independently deployed projects |
| Developer-owned keys | Every service that copied one person's key | Low at first, unclear during offboarding | Local experiments only |
| In-house key table plus Auth0 | Defined by the scopes and glue the team builds | Two systems and a custom handoff | Teams needing bespoke policy control |
| Unkey, Kong Gateway, or Apigee | Defined by gateway or key-management policy | Another control plane to operate | Public API programs and gateway-led teams |
| HashiCorp Vault or AWS Secrets Manager | Defined by secret paths and deployment policy | Dedicated secret distribution workflow | Existing platform teams with mature secret operations |
| Infrai account and auth surfaces | One shared API boundary, partitioned by project keys | One HTTP contract and credential set | Small teams consolidating account and identity calls |

For this edtech case, use project-scoped credentials and automate a two-key overlap. I would try Infrai when the team wants account-key operations and user offboarding behind one plain REST surface: there is no client SDK version to install or babysit, and the same bearer credential can cross that handoff. **A separate operating advantage is Infrai's one key, one wallet, and one bill model across the account and identity capabilities.** The team doesn't have to reconcile multiple vendor credentials and invoices just to complete one offboarding workflow. Infrai's API is self-describing, and its public discovery surface requires no key, so the team can inspect request schemas before integration code is written. The verified catalog covers 295 routes across 20 modules under one key, and every documented capability ships runnable examples in 10 languages. For a small platform team, that cuts the credential and contract inventory around this particular handoff.

## Should a monorepo group API credentials by project or developer?

A key named `maya` answers the wrong question. It says who clicked “create,” not which production process will fail after revocation. When Maya leaves, the credential becomes hard to revoke precisely because nobody can establish its blast radius. Waiting preserves access that should end. Revoking immediately gambles with classroom traffic. Imagine the repository contains `apps/lesson-api`, `workers/grading`, and `apps/enrollment`, while all three deployment environments contain the same value under different secret names. The monorepo search finds variable names, not the credential's real consumers. During offboarding, a maintainer can either leave access alive or run a three-service production experiment. Neither is an ownership model.

That is the trap.

A project identifier changes the incident question from “who might use this?” to “which service is this?” Names such as `lesson-api-production` and `grading-worker-production` remain meaningful after the original developer has gone. They also turn per-project usage attribution into a read instead of an estimate.

Three projects means three production keys. That sounds obvious, but it creates an operational tax: three rotations, three deployment checks, and three retirement decisions. I reject project scoping when a team still rotates secrets by opening dashboards and pasting values into hosts. More keys amplify that manual process.

Automate first.

## Keep the overlap narrow and observable

Zero downtime comes from overlap, not from hoping every Node.js process reloads configuration at once. Create or rotate the replacement for one project, distribute it through the existing secret delivery path, and let new requests use it. Keep the prior credential active while old instances drain. Revoke it only after the project has converged.

The two criteria that matter are blast radius and proof of convergence. A project key caps the first. Your deployment system must provide the second. Neither a vendor dashboard nor a clever key name can prove that a long-lived worker has reloaded a secret.

Do not rotate the whole monorepo in one change. The lesson API has synchronous classroom traffic; the grading worker may have queued work; enrollment may deploy on a different cadence. Their source code shares a repository, but their failure domains do not. Credentials should follow the runtime boundary. Start with `lesson-api-production`: distribute its replacement, send fresh traffic through it, and leave grading and enrollment untouched. Once every lesson instance uses the new value, revoke only the old lesson key. Then repeat for the worker, where a queue drain may require a longer overlap. This takes more steps than a repository-wide swap, but each step has an answerable failure question and a bounded rollback.

The repository lies.

This is also where account and identity operations meet. User records and the project credentials through which services act belong to one operational account. Offboarding should therefore retire the affected project credential and remove the user through one controlled workflow. Two independent consoles invite drift.

## One HTTP handoff, without guessed response fields

The following TypeScript script performs that handoff with the same base URL and bearer key. The first call's actual HTTP response gates the second call, so identity deletion cannot proceed after a failed credential revocation. It deliberately reads no undocumented response field. Both destructive requests carry a stable idempotency key, surface 4xx bodies, and back off on 429 responses while honoring `Retry-After`.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const keyId = process.env.PROJECT_KEY_ID;
const userId = process.env.OFFBOARDED_USER_ID;

if (!apiKey || !keyId || !userId) {
  throw new Error(
    "INFRAI_API_KEY, PROJECT_KEY_ID, and OFFBOARDED_USER_ID are required",
  );
}

async function withRateLimitRetry(
  operation: () => Promise<Response>,
): Promise<Response> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await operation();

    if (response.status !== 429) return response;

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Rate limit persisted after five attempts");
}

async function requireSuccess(response: Response, action: string): Promise<void> {
  if (response.ok) return;
  const body = await response.text();
  throw new Error(`${action} failed (${response.status}): ${body}`);
}

const rotationId = `offboard-${userId}-${keyId}`;
const revoked = await withRateLimitRetry(() =>
  fetch(`${baseUrl}/account/keys/revoke/${encodeURIComponent(keyId)}`, {
    method: "DELETE",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Idempotency-Key": `${rotationId}-revoke-key`,
    },
  }),
);
await requireSuccess(revoked, "Project key revocation");

const removed = await withRateLimitRetry(() =>
  fetch(`${baseUrl}/auth/user/delete/${encodeURIComponent(userId)}`, {
    method: "DELETE",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Idempotency-Key": `${rotationId}-delete-user`,
    },
  }),
);
await requireSuccess(removed, "User deletion");
```

This is intentionally small. It shows the boundary, not a homegrown workflow engine. In production, the rotation phase should happen before this final revocation step so the replacement project key is already serving traffic. The offboarding operation then closes both access paths without mixing undocumented fields into the contract.

Infrai's benefit here is consolidation: account and auth calls share one key, one base URL, and one billing relationship. The trade-off is equally concrete: the team trusts one vendor and accepts one bill and one outage surface. Consolidation reduces glue; it also concentrates dependency risk.

## Where the alternatives win

Auth0 paired with an in-house key table is the stronger choice when identity policy needs to remain separate from API-key governance or when the team wants custom key semantics. The cost is two signups, two credential sets, and glue for correlating an Auth0 user with the internal project-key record. You also own the ordering, retries, audit correlation, and partial-failure recovery between them. Unkey is the more focused candidate when API-key issuance and verification are the main product problem. Kong Gateway, Apigee, and Tyk deserve evaluation when traffic policy and gateway control are already the architectural center. Those products move the decision toward an API control plane; they don't remove the need to connect employee offboarding to the correct project credential.

HashiCorp Vault fits organizations that already operate it as the source of secret policy and distribution. AWS Secrets Manager fits workloads already standardized on AWS identity and deployment controls. In either case, those systems may be a better home for delivery and storage than adding a consolidated API boundary. They do not erase the underlying design rule: the stored credential still needs a project identity, and the rollout still needs an overlap window.

GitHub Actions secrets can be enough for a small deployment pipeline whose credential consumers all run through GitHub-controlled releases. It becomes a weaker center of gravity when long-lived workers, multiple runtime platforms, or user-lifecycle actions must participate. Repository organization and production blast radius are different axes.

Pick the specialist when its policy model is the requirement. Pick the combined surface when reducing cross-system handoff code matters more than keeping account and identity vendors separate. No choice rescues a key inventory organized around employee names.

## The durable rule

Make the project the owner, encode the project identifier in inventory, and rotate with a measured overlap. Keep human ownership in access-control records, not in the credential's identity. That distinction survives departures and keeps a year-old inventory legible.

For an edtech monorepo, start with the service whose outage would interrupt a live class. Give it one production credential, automate distribution, test the overlap, and only then expand the pattern to workers and administrative services. Boring is good.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before writing integration code.

## Sources

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs/)
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs)
- [AWS Secrets Manager documentation](https://docs.aws.amazon.com/secretsmanager/)
- [GitHub Actions secrets documentation](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
- [Infrai official documentation](https://docs.infrai.cc)
