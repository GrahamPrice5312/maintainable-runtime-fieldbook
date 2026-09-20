# 4 Checks When a Deleted User Avatar Still Loads (CDN Cache Debugging)

An old avatar URL can outlive the database row that pointed to it. Short answer: stop issuing the old URL, check whether the edge still holds the bytes, and verify that a request to the origin cannot retrieve them either. A new versioned URL solves stale replacements; it does not revoke an already shared link. For a removal request, treat the old URL as a separate deletion target.

This matters in a media site that reviews user-uploaded images before publication. The review pipeline may keep a source image, a resized avatar, and multiple output formats. A deletion test that checks only the profile page misses direct requests for those derivatives. Image quality and bandwidth matter for approved photos, but neither trade-off excuses a removed avatar remaining reachable.

## 1. Why does a deleted user avatar still load from the CDN cache?

Start with the exact URL that reproduces the report, including its query string. Record the response status, `Cache-Control`, `Age`, `ETag`, and any cache-status header the edge exposes. Repeat the request through the normal public path and directly against the origin if that path is available to your operators. Do not assume that a changed profile page proves deletion: the browser may be showing a local copy, an edge may hold a public response, or the origin may still serve an unreferenced derivative.

Keep the tests separate. A fresh browser session reduces local-cache confusion but does not bypass an edge cache. A cache-busting query parameter may create a new cache key rather than inspect the old object. If a response continues to return bytes after its stored object is removed, the origin and its backing storage need inspection. If only the original URL returns bytes at the edge, inspect the purge target and every cache-key variant.

One URL at a time.

Don't rename a cache miss as a deletion success.

For example, compare the original avatar link embedded in an older feed entry with the current profile link, then repeat each request with the same URL, host, and query string. A new versioned URL on the profile tells you what the application publishes now; it says nothing about the older feed link. If the old link returns a cached response, a request using a new query string is testing a different key. If the identical old link returns an image after a purge, look for another variant of the URL key, a failed purge, or an origin that still has the rendition. This check takes longer than refreshing a profile page, but it isolates the actual layer that serves the bytes.

## 2. Trace every derivative before changing the public pointer

For an avatar, the source file and each rendition need a stable asset ID in the media inventory. Store the moderation state alongside that ID, plus the object keys and public URL keys for the renditions produced from it. A URL version identifies a publication revision; it should not be the only record of what must be removed. Two sizes, two targets. This is especially easy to miss when a feed uses one size and a profile uses another.

The removal sequence is a policy decision with a concrete ordering constraint. Mark the asset unavailable to your application first, so a race with moderation or profile updates cannot publish it again. Then remove the source and derivatives, invalidate cache entries for their previously issued public URLs, and test those URLs through the public edge. Cache invalidation is asynchronous in many deployments; keep the deletion job pending until verification completes, and record failures for retry. If the URL is served by a third-party proxy outside your control, identify that boundary instead of claiming the bytes have vanished everywhere.

## 3. Make the smallest deletion gate explicit

The following TypeScript sketch covers the application-level gate. It assumes the media inventory can enumerate all stored renditions and previously issued URLs. The storage and cache adapters are deliberately abstract: their deletion and invalidation semantics must be checked in the deployed system.

```ts
type Asset = {
  id: string;
  status: "approved" | "removed";
  objectKeys: string[];
  publicUrls: string[];
};

interface Inventory {
  get(id: string): Promise<Asset | null>;
  markRemoved(id: string): Promise<void>;
}
interface ObjectStore {
  remove(key: string): Promise<void>;
}
interface EdgeCache {
  invalidate(url: string): Promise<void>;
}

async function removeAvatar(
  id: string,
  inventory: Inventory,
  objects: ObjectStore,
  edge: EdgeCache,
): Promise<void> {
  const asset = await inventory.get(id);
  if (!asset) return;

  await inventory.markRemoved(id);
  await Promise.all(asset.objectKeys.map((key) => objects.remove(key)));
  await Promise.all(asset.publicUrls.map((url) => edge.invalidate(url)));
}
```

This function is not proof of erasure. Retry failed steps. The caller must do so without resurrecting the asset, and a worker must verify public URLs after invalidation settles. Serving code must check the removed state before generating fresh URLs or reading a stored object. For an edge that serves cached bytes without contacting that code, the application gate alone cannot block an existing cached response.

The useful test is a matrix of previously published URLs: every size, format, host, and version that users could have received. Check each as an unauthenticated client. An `Age` value can help locate a cache hit, but the decisive observation is whether the response still contains image bytes after deletion and invalidation. Log the asset ID, deletion-job state, URL-key identifier, response status, and time of the verification request. Avoid logging a user's full URL if it embeds a sensitive token.

The matrix should include the actual transformations the moderation pipeline published, not hypothetical formats the browser might support. Suppose an approved upload generated a square profile image and a smaller feed image. The source, those two outputs, and any separately cached response paths are distinct deletion candidates even if the UI renders only one of them today. A public URL with a version suffix is another key to check, not an instruction to the edge to erase the previous version. Record which key failed before changing an invalidation rule; otherwise a successful request to the new version may disguise a stale response at the old address. And when a direct-origin probe fails because the origin is deliberately private, don't weaken origin access to make debugging convenient. Use an operator-controlled inspection path for storage keys, then verify the result from outside through the same public hostname that exposed the problem. The final check is observable behavior, not a dashboard count of purge requests.

Run one regression test with a paused worker: remove an approved avatar while a derivative-generation job is in flight, then release the worker and confirm that no new public rendition appears. Run another with a cached old URL. Replacing a picture and removing a picture are different operations; a version bump is useful for replacement because the page points to fresh bytes, while removal has to account for links already distributed.

## 4. What changes when this grows?

At higher volume, make deletion a durable job keyed by asset ID, with idempotent steps for storage removal, invalidation, and public verification. Count pending and failed jobs, and alert on verification failures rather than treating a successful API response as completion. That adds state and operational work. The payoff is an auditable distinction between "removed from the profile," "removed from storage," and "no longer served at the public URL."

For newly published avatars, choose cache policy with the removal requirement in mind. Public, long-lived caching saves bandwidth and can improve delivery of approved variants, but increases the work needed to retract a URL. Private or short-lived responses reduce that exposure at a bandwidth cost. Measure both with the same test corpus of approved images and the same deletion checks; do not use an image-quality score as a substitute for a removal test.

## References

- HTTP caching semantics and cache invalidation: https://www.rfc-editor.org/rfc/rfc9111
- HTTP response `Cache-Control` and `Age` headers: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control and https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Age
- Image formats and browser support: https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types

## Sources

- https://www.rfc-editor.org/rfc/rfc9111
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Age
- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
