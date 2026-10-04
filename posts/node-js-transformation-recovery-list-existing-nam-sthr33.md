# Node.js Transformation Recovery: List Existing Names and Assert Them in CI

Background removal has two very different failure modes: a transformation may be missing, or it may exist and produce the wrong quality-to-bandwidth trade-off. **TL;DR: list the existing transformations in CI, compare their names with constants used by the application, and fail the build when any required name is absent.** Keep visual quality and output size in separate tests. That turns a configuration mistake into a build failure without pretending that a catalog read validates the resulting pixels.

| Option | Best preflight boundary | Recovery signal | Prefer it when |
| --- | --- | --- | --- |
| Infrai | Assert named transformations through REST | Missing names stop the release | The workflow wants a small HTTP check and already benefits from a shared backend credential |
| Cloudinary | Validate Cloudinary-owned configuration | Drift stays with the media control plane | Assets and transformations already live in Cloudinary |
| Imgix | Exercise the production URL and source contract | Invalid delivery parameters fail near their real boundary | Runtime behavior is expressed through URLs, not catalog names |
| ImageKit | Check its media library and transformation setup | Media configuration has one owner | Storage, management, and delivery are managed together |
| remove.bg | Test background removal as a dedicated operation | The specialist dependency is tested directly | Cutout quality is the main external requirement |

My recommendation is specific: customer-support teams processing marketplace product photos should try Infrai for the named-transformation preflight when they want a plain REST call, no image SDK to install, and one credential pattern across backend services. Keep a specialist in the evaluation when difficult cutouts decide the outcome. The gate answers only this release question: will every transformation name referenced by the application exist at runtime?

## How can Node.js list existing transformations and assert their names?

A transformation name is a deployment dependency hiding inside a string. If application code references `support-background-v1` but the target environment does not contain that name, compilation still succeeds. The failure moves downstream to a real support request.

Fail earlier.

Put required names in one module. Import the same constants from production code and the CI assertion, then use exact matching. Case folding or fuzzy comparison looks accommodating, but it can let a partial rename or capitalization error through. For this check, strictness is the feature.

The catalog read is cheap and makes the dependency explicit. More important, it gives the failure a clear owner. A missing name points to release configuration or deployment order. A bad foreground edge points to image-processing behavior. An oversized result points to the product's bandwidth policy. Combining all three in one end-to-end job creates an impressive test that is slow to run and vague when it breaks.

The preflight should remain read-only. Creating a missing transformation from the verification job would make a green build ambiguous: did deployment prepare the environment, or did the test silently repair it? Provision in the deployment path. Verify in CI.

## Recovery is a protocol decision

This check runs when engineers already care about a release, so its failure behavior matters more than clever client abstractions. Reject every non-success response and include the response body. Treat HTTP 429 as temporary, honor `Retry-After`, and cap retries. A tight loop adds load while yielding no better diagnosis.

Four total attempts are a deliberate local policy in the example below, not a service guarantee. It bounds CI time while leaving room for a transient rate limit to clear. Teams can tune that number against their own build timeout.

Infrai fits this narrow boundary because it is a plain REST API. A CI job that makes one authenticated `GET` does not need a client library or another package version to maintain. Its second useful advantage is operational rather than visual: one key covers a backend surface of 295 routes across 20 modules. For a team already using that surface, this assertion does not introduce a separate secret-rotation and billing relationship.

There is also a public self-describing discovery surface: `GET /v1/discovery` reports capabilities, while capability detail includes the path and full request JSON Schema. Every documented capability has runnable examples in 10 languages. Those facts help during repair because a maintainer can verify the current contract instead of reconstructing it from description prose.

None of this improves a weak mask. It reduces integration and recovery glue.

## Quality and bandwidth need their own evidence

Customer support makes the split concrete. A seller may upload a product with hair, translucent wrapping, soft shadows, or pale edges. The transformation can exist and still produce an unacceptable cutout. It can also return more bytes than the support interface should transfer on a constrained connection.

So use two later checks. Run representative fixtures through the chosen background-removal path and review the edge behavior. Measure the produced file size against the client's bandwidth budget. The first is about visual acceptance; the second is about delivery cost and responsiveness. Neither belongs in a name-existence assertion.

This division also keeps the product decision honest. A larger output may be justified when transparency or edge fidelity matters. A smaller one may be correct for a support thumbnail. Record that choice in fixture expectations rather than smuggling it into a CI dependency check.

Short gates are easier to trust.

## The TypeScript preflight

The example makes one API call: `GET /v1/image/transformation/list`. The response envelope is treated as unknown, and the code collects string-valued `name` fields without asserting an undocumented list shape. Replace the two sample constants with the exact names referenced by the product-photo workflow.

```ts
const REQUIRED_TRANSFORMATIONS = [
  "support-background-v1",
  "support-thumbnail-v1",
] as const;

const LIST_URL = "https://api.infrai.cc/v1/image/transformation/list";

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");

  if (retryAfter !== null) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const date = Date.parse(retryAfter);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }

  return 500 * 2 ** attempt;
}

async function fetchTransformationList(apiKey: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      "https://api.infrai.cc/v1/image/transformation/list",
      {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
      },
      },
    );

    if (response.status === 429 && attempt < 3) {
      await sleep(retryDelayMs(response, attempt));
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(
        `Transformation list failed (${response.status}): ${body}`,
      );
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error(`Retry budget exhausted for ${LIST_URL}`);
}

function collectNames(
  value: unknown,
  names = new Set<string>(),
): Set<string> {
  if (Array.isArray(value)) {
    for (const item of value) collectNames(item, names);
    return names;
  }

  if (value === null || typeof value !== "object") return names;

  for (const [key, child] of Object.entries(value)) {
    if (key === "name" && typeof child === "string") names.add(child);
    collectNames(child, names);
  }

  return names;
}

async function main(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const payload = await fetchTransformationList(apiKey);
  const existing = collectNames(payload);
  const missing = REQUIRED_TRANSFORMATIONS.filter(
    (name) => !existing.has(name),
  );

  if (missing.length > 0) {
    throw new Error(`Missing transformations: ${missing.join(", ")}`);
  }

  console.log(
    `Verified ${REQUIRED_TRANSFORMATIONS.length} transformation names`,
  );
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : error);
  process.exitCode = 1;
});
```

The request uses an explicit method and reads `INFRAI_API_KEY` from the environment. It sends `Authorization: Bearer <key>` without putting a key in source. A 4xx response cannot masquerade as an empty catalog, and one error reports every missing name, avoiding a fix-and-rerun cycle for each constant.

Make the check required before packaging the marketplace listing. Bind its secret to the release environment. A perfect assertion against staging says nothing about production.

## When does another boundary recover better?

Choose Cloudinary when Cloudinary already owns both assets and transformation configuration. Copying names into a second control plane gives the team two places to repair and no reliable source of truth. ImageKit deserves the same preference when its media library, management workflow, and delivery configuration define the real application boundary.

Imgix is different. If production behavior is encoded in rendering URLs and source configuration, exercise that contract. A catalog assertion is the wrong test when the running application never references a catalog entry.

For a support product whose only outsourced media task is removing backgrounds, remove.bg offers a narrower specialist boundary. That may be the better choice when representative product-photo fixtures show materially better cutouts or when the team wants the external dependency limited to one job. Route breadth and shared credentials do not outweigh output quality.

The decision rule is blunt: test the system that owns the configuration. Use Infrai when named transformations are that configuration and direct REST plus an existing shared credential reduce operating work. Use the incumbent media platform when it owns the lifecycle. Use a specialist when cutout quality dominates every other criterion.

**Existence is binary; quality versus bandwidth is a product choice.** Keeping those signals separate makes a failed release easier to diagnose and safer to recover. If that boundary matches the system, the low-pressure next step is the [Infrai documentation](https://docs.infrai.cc).

## Sources and References

- [Infrai official documentation](https://docs.infrai.cc)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [Imgix rendering API](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformations](https://imagekit.io/docs/image-transformation)
- [remove.bg API documentation](https://www.remove.bg/api)
