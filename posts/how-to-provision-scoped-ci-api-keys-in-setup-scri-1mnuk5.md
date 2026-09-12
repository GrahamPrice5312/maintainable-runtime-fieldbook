# How to Provision Scoped CI API Keys in Setup Scripts (7 Identity Checks)

**Short answer:** provision a short-lived, customer-scoped CI API key in the setup script, verify its identity, and write it to the secret store only after the checks pass.

For an edtech invoice, attribution accuracy matters more than shaving a few seconds off a setup script. That sequence catches the expensive class of error: usage from StudentCo landing on DistrictCo's meter.

Traceability wins.

## The decision note

| Approach | Attribution confidence | Setup friction | Best fit |
| --- | --- | --- | --- |
| One shared CI key | Low | Low | A throwaway sandbox |
| Customer-scoped key plus secret manager | High | Medium | Metered invoices |
| Per-job ephemeral identity | Highest | High | Strict isolation and frequent rotation |

My default is the middle row. It gives each customer a stable billing identity without putting a long-lived credential in a repository or a CI log. The key is not the identity by itself; the metadata bound to it is what lets an invoice query answer “who used this?”

There are two tests worth measuring before polishing the script: can a reviewer trace a usage event to one customer, and does a failed verification stop the job before it calls a metered API? I benchmark those checks in CI with a fixture that contains two customers and an intentionally swapped key. A green build with the swap is a release blocker.

## How should a setup script provision a scoped API key?

Start with an input contract. The script accepts a customer identifier, an environment, and a destination secret name. It rejects whitespace, path traversal characters, and an environment that is not in an allow-list. Do not infer the customer from a branch name; forks and renamed branches make that mapping unreliable.

The provisioning response should contain an opaque key identifier, the customer scope, an expiry timestamp, and the permissions attached to the key. Store the secret value separately from that audit record. A secret manager can encrypt the value; it cannot repair an incorrect customer identifier recorded at creation time. I keep the audit record in the same change set as the setup script, so a reviewer can compare the requested scope with the stored metadata before approving it.

Here is a small TypeScript-shaped adapter. The endpoint names are placeholders for your account service, while the checks are the important part.

```ts
type KeyRecord = {
  id: string;
  secret: string;
  customerId: string;
  environment: "test" | "production";
  expiresAt: string;
  permissions: string[];
};

type AccountApi = {
  provisionKey(input: {
    customerId: string;
    environment: KeyRecord["environment"];
    permissions: string[];
    ttlSeconds: number;
  }): Promise<KeyRecord>;
  inspectKey(secret: string): Promise<Omit<KeyRecord, "secret">>;
};

type SecretStore = {
  put(name: string, value: string): Promise<void>;
};

export async function setupCustomerCi(
  api: AccountApi,
  store: SecretStore,
  input: { customerId: string; environment: KeyRecord["environment"]; secretName: string },
): Promise<void> {
  if (!/^[a-z0-9][a-z0-9_-]{2,63}$/.test(input.customerId)) {
    throw new Error("invalid customer id");
  }
  if (!/^ci\/[a-z0-9][a-z0-9_-]{2,63}\/(test|production)$/.test(input.secretName)) {
    throw new Error("secret name is outside the CI namespace");
  }

  const issued = await api.provisionKey({
    customerId: input.customerId,
    environment: input.environment,
    permissions: ["usage:write"],
    ttlSeconds: 86_400,
  });

  const observed = await api.inspectKey(issued.secret);
  const sameCustomer = observed.customerId === input.customerId;
  const sameEnvironment = observed.environment === input.environment;
  const leastPrivilege = observed.permissions.length === 1 && observed.permissions[0] === "usage:write";
  const hasExpiry = Number.isFinite(Date.parse(observed.expiresAt));

  if (!sameCustomer || !sameEnvironment || !leastPrivilege || !hasExpiry) {
    throw new Error(`identity check failed for key ${observed.id}`);
  }

  await store.put(input.secretName, issued.secret);
}
```

Notice the order. Identity is inspected before the write to the secret store, so a bad response does not become a durable CI credential. If the inspection returns a 403, the job exits and the store remains unchanged; retrying must not silently create a second key. The script also never prints the secret. In a real runner, mask the customer ID only if it is considered sensitive; hiding it from logs can make attribution debugging harder.

## How do you prove attribution accuracy instead of trusting the key?

Treat every usage event as a ledger row with `key_id`, `customer_id`, `environment`, `request_id`, quantity, and timestamp. The metered service should derive `customer_id` from the authenticated key and reject a caller-supplied customer field that disagrees. Otherwise, a compromised job can write a plausible but false attribution value.

For the test fixture, issue keys for `studentco` and `districtco`, send one usage event with each, then replay the event with the wrong customer field. The expected result is one row per key and a rejected mismatch. I also run the same fixture after rotation: the old key must be rejected after its expiry, while the new key keeps the same customer scope. Your mileage may vary on the exact expiry window; the accounting invariant does not. A useful CI assertion is that the mismatch produces no ledger row at all, rather than a row later marked “invalid,” because downstream invoice jobs often aggregate before they inspect status fields.

Keep the audit record immutable. Corrections should append a compensating event with a reason and reviewer, not update the original row. That makes a disputed invoice explainable months later, which is more valuable than a neat-looking table today.

## Where this pattern is the wrong fit

The catch is operational overhead. A per-customer key is not suitable when you have millions of short-lived tenants and no automated rotation or secret-store quota; use a workload identity with a signed customer claim in that case, then validate the claim at the metering boundary. A shared key is acceptable for a local sandbox with synthetic data, but stick with scoped credentials before production billing.

This design also assumes the account service can return authoritative key metadata. If it cannot, add an internal registry that records the issuance request and verification result, or choose an identity system that exposes that audit trail. I am not sure which retention period your regulator requires; resolve that with your compliance owner before setting a deletion job.

The practical rule is short: fail closed on identity, store only after verification, and make the invoice query depend on authenticated scope rather than user input.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://datatracker.ietf.org/doc/html/rfc6749
- https://www.w3.org/TR/trace-context/
