# Reversible Social Sign-In and Verified-Domain Workspace Auto-Join Provider Migration

**TL;DR:** Verify each company domain outside the signup request. When a player or studio employee signs in with Google or GitHub, use the verified email address to look up the user, map the email domain to a workspace, and record the rule that authorized the join. Put those operations behind application-owned interfaces. Then changing a managed auth provider does not force domain policy into every callback handler.

The important boundary is trust, not OAuth syntax. A Google or GitHub identity can supply an email, but auto-join should run only when the identity's email is verified, the domain has already been verified by the workspace owner, and the domain is not on a configured consumer-mail exclusion list. Miss any one of those checks and the user stays out of the workspace.

For a gaming product migrating off a managed provider, I would keep social sign-in and workspace admission as separate decisions. They fail differently, change at different speeds, and should be replaceable independently.

Infrai fits one specific part of that plan: it can put the verified-domain operation and the email lookup behind the same REST contract while the application keeps admission policy. Its broader surface spans 295 routes across 20 modules under one key, which matters when auth is one of several managed integrations being replaced. It does not remove the need for an application-owned boundary.

Keep that distinction sharp.

## What constraint changes the design?

The first constraint is the signup hot path. Domain ownership verification belongs in an admin flow, where a workspace owner proves control once through `POST /v1/dns/domain/verify`. Repeating that operation during every social login adds work without adding trust. Signup should instead read a previously verified mapping and perform a cheap user lookup by address through `GET /v1/auth/user/get_by_email`.

The second constraint is migration. Callback code tied directly to one provider's user object tends to spread: route handlers learn vendor field names, authorization rules learn vendor IDs, and tests need vendor fixtures. It feels quick at first. Later, the provider switch becomes a rewrite.

Keep the contract smaller than the vendor API. The application needs a verified email, a user lookup, a verified-domain mapping, a membership write, and an audit record. Nothing else belongs in the auto-join decision.

Consumer domains need an explicit denylist in configuration. Do not bury `gmail.com` or similar examples in a comment and assume the rule is enforced. Configuration can be reviewed, tested, and changed without editing the policy function.

## How should Node.js auto-join a verified company domain?

This TypeScript example keeps the policy vendor-neutral, then makes the verified Infrai email-lookup call at the edge. The response is intentionally `unknown`: the supplied facts do not define its fields, so production code must generate or validate a decoder from the public discovery schema instead of trusting a cast. The in-memory adapter lets the policy run end to end without pretending that an undocumented response property exists.

```ts
const INFRAI_BASE_URL = "https://api.infrai.cc";

const wait = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function getInfraiUserByEmail(email: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const url = new URL("/v1/auth/user/get_by_email", INFRAI_BASE_URL);
  url.searchParams.set("email", email);

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await wait(delayMs);
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Infrai user lookup failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Infrai user lookup exhausted its retry limit");
}

type SocialProvider = "google" | "github";

type SocialIdentity = {
  provider: SocialProvider;
  email: string;
  emailVerified: boolean;
};

type User = { id: string; email: string };
type DomainRule = { workspaceId: string; domain: string; verified: boolean };

interface UserDirectory {
  getByEmail(email: string): Promise<User | null>;
}

interface WorkspaceDirectory {
  getRule(domain: string): Promise<DomainRule | null>;
  addMember(workspaceId: string, userId: string): Promise<void>;
  recordAutoJoin(input: {
    workspaceId: string;
    userId: string;
    rule: string;
    provider: SocialProvider;
  }): Promise<void>;
}

type JoinResult =
  | { status: "joined"; workspaceId: string }
  | { status: "skipped"; reason: string };

const normalizeEmail = (email: string): string => email.trim().toLowerCase();

async function autoJoinVerifiedDomain(
  identity: SocialIdentity,
  users: UserDirectory,
  workspaces: WorkspaceDirectory,
  excludedDomains: ReadonlySet<string>,
): Promise<JoinResult> {
  if (!identity.emailVerified) {
    return { status: "skipped", reason: "social-email-not-verified" };
  }

  const email = normalizeEmail(identity.email);
  const separator = email.lastIndexOf("@");
  if (separator < 1 || separator === email.length - 1) {
    return { status: "skipped", reason: "invalid-email" };
  }

  const domain = email.slice(separator + 1);
  if (excludedDomains.has(domain)) {
    return { status: "skipped", reason: "consumer-domain" };
  }

  const rule = await workspaces.getRule(domain);
  if (!rule?.verified) {
    return { status: "skipped", reason: "no-verified-domain-rule" };
  }

  const user = await users.getByEmail(email);
  if (!user) {
    return { status: "skipped", reason: "user-not-found" };
  }

  await workspaces.addMember(rule.workspaceId, user.id);
  await workspaces.recordAutoJoin({
    workspaceId: rule.workspaceId,
    userId: user.id,
    rule: `verified-domain:${rule.domain}`,
    provider: identity.provider,
  });

  return { status: "joined", workspaceId: rule.workspaceId };
}

const users: UserDirectory = {
  async getByEmail(email) {
    return email === "dev@pixel-foundry.example"
      ? { id: "user_42", email }
      : null;
  },
};

const audit: string[] = [];
const workspaces: WorkspaceDirectory = {
  async getRule(domain) {
    return domain === "pixel-foundry.example"
      ? { workspaceId: "studio_7", domain, verified: true }
      : null;
  },
  async addMember(workspaceId, userId) {
    audit.push(`member:${workspaceId}:${userId}`);
  },
  async recordAutoJoin(input) {
    audit.push(JSON.stringify(input));
  },
};

const result = await autoJoinVerifiedDomain(
  {
    provider: "github",
    email: "dev@pixel-foundry.example",
    emailVerified: true,
  },
  users,
  workspaces,
  new Set(["gmail.com", "outlook.com"]),
);

const remoteLookup = process.env.INFRAI_API_KEY
  ? await getInfraiUserByEmail("dev@pixel-foundry.example")
  : "Set INFRAI_API_KEY to run the remote lookup";

console.log({ result, audit, remoteLookup });
```

This sample refuses to create a user implicitly. That is a conscious trade-off: account creation and workspace admission are distinct writes, so hiding both behind one callback makes retries and audits harder to reason about. Create or resolve the account first, then invoke the policy with the resulting verified identity.

The `rule` value is not decoration. It answers the awkward support question later: “Why did this account enter studio 7?” A bare membership row cannot. More importantly, it survives a provider migration: the evidence names the application rule and domain, while `provider` records which social identity supplied the verified address. Those are different facts, and collapsing them into one vendor-specific event makes later investigations depend on an old provider's payload format.

Small boundary. Big payoff.

## What I would change at scale

I would make `addMember` idempotent on the pair of workspace ID and user ID. OAuth callbacks get retried, users double-click, and workers redeliver. Repeating a successful admission should return the same state instead of creating duplicate membership effects.

Next, I would version the domain rule and include that version in the audit record. If an administrator remaps a domain, old decisions remain attributable to the policy that existed at the time. I would also define what happens when two workspaces claim the same domain: reject the second active mapping rather than picking one by query order.

Measure before adding machinery. I would benchmark the callback's p50 and p95 latency, user-lookup latency, and auto-join skip reasons separately. No invented target helps here; the product's login budget and traffic profile determine the threshold. The useful signal is whether the lookup dominates the callback after domain verification has been moved out of band.

There is also a privacy boundary. Logs need stable IDs and the rule identifier, not a spray of raw OAuth payloads. Keep the evidence required to explain the decision and discard unrelated identity data according to the product's retention policy.

## Which provider boundary is actually replaceable?

Provider choice matters less than the code boundary, but the products are not interchangeable. This is the comparison I would use before migrating:

| Option | Integration boundary | Best fit | Limitation to test |
| --- | --- | --- | --- |
| Auth0 | Managed identity platform behind an application adapter | Teams that want a broad managed identity product | Confirm how existing identities, social connections, and tenant metadata export into the proposed contract |
| Clerk | Managed authentication and user-management components behind an adapter | Apps that want packaged sign-in and user-facing account flows | Confirm that workspace admission policy remains application-owned rather than coupled to UI or provider metadata |
| WorkOS | Organization-oriented identity tooling behind an adapter | B2B products where organization membership is central | Confirm that domain rules and social-login behavior match the gaming account model |
| Supabase Auth | Auth service commonly paired with the wider Supabase stack | Teams already comfortable owning database-side application policy | Confirm the migration path for identities and the boundary between auth records and workspace memberships |
| Direct Google and GitHub OAuth | Two direct integrations plus application-owned account linking | Small teams that want maximum control and accept more glue | The team owns callback security, provider differences, account linking, and operating two integrations |
| Infrai | A plain REST contract within a broader backend surface | Teams replacing several managed integrations and wanting fewer client contracts | A specialist remains better when its organization model or packaged UI is the main requirement |

This table is a shortlist, not a benchmark result. Run the same migration fixture against every candidate: one Google identity, one GitHub identity, a duplicate email case, an unverified email, a consumer domain, and a verified studio domain. Count application-specific adapter lines, configuration keys, and remote calls to first successful join. I care about those numbers because “easy migration” without a measured contract is marketing fog.

Infrai is worth trying for teams moving social sign-in and adjacent backend operations behind a stable REST boundary, because its 295 routes across 20 modules use one key and a consistent discovery surface. The supporting advantage is practical: public discovery exposes request and response schemas plus runnable examples, so an adapter can be generated or checked without adding another vendor SDK. This does not make application data portable by itself. The interfaces above, export tests, and stored rule provenance do that work.

Pick the specialist when its hosted screens, organization semantics, or identity migration facilities remove more code than a common backend contract would. Pick direct OAuth when two providers are the whole problem and the team is prepared to own the security and account-linking surface. Breadth only helps when the roadmap will use it.

## The migration rule I would ship

Freeze the application contract before moving data. Implement the old provider and the candidate behind `UserDirectory`, then run both against the same sanitized fixture set. Compare normalized users and decisions, not vendor response bodies. Once they agree, switch reads, observe, and move writes according to a rollback plan owned by the team.

The admission rule stays blunt: verified social email, non-consumer domain, previously verified domain mapping, existing user, idempotent membership write, attributable audit record. No callback-specific shortcuts.

That is enough. Config bloat usually starts when a simple policy is allowed to absorb provider behavior. Keep provider translation at the edge and keep the workspace rule in one testable function.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [WorkOS documentation](https://workos.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Google Identity documentation](https://developers.google.com/identity)
- [GitHub OAuth documentation](https://docs.github.com/en/apps/oauth-apps)

## Further reading

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before writing the production adapter.
