# Logistics Photo Sizes API: Consistent Named Crops With Moderation Before Delivery

A logistics marketplace needs consistent listing photo sizes from its API, but a tidy grid is a weak success criterion if a crop hides the detail a reviewer needed to see. For equipment and freight listings, make moderation coverage the constraint that decides when an image can become public.

TL;DR: use an API contract with named aspect ratios, keep uploads private, and approve the exact renditions that will be served. In a Node.js service, resolve names to fixed dimensions on the server, generate and compress candidates, then publish only candidates whose source and output both pass the application's review policy. Treat this as a proposed workflow, not a claim that a particular classifier can detect every prohibited image.

## How should a listing API keep photo sizes consistent?

Consider a proposed fixture: one equipment listing has a wide trailer photo, a portrait loading-bay photo, and a close-up of a shipping label. The card needs a square; the detail view needs a landscape image. Those are different views of the upload. A moderation decision about one view should not silently become a decision about all of them. A crop can omit context. Reviewing only the card can also leave the larger detail rendition outside the review boundary.

Define coverage first.

For this build, coverage means that each public rendition has an approved source revision, an approved output, and a recorded policy revision. It does not mean a model has perfect recall. The review policy must say what counts as a pass, what needs human review, and what remains private when the reviewer cannot decide. A low-confidence result is a state to handle, not a Boolean to coerce.

That constraint changes the implementation choice: build candidates in private storage, review them, and publish afterward. Compression still happens before serving, but successful compression does not authorize publication. This adds work and latency. The reason to accept that trade-off is specific: a listing page should never acquire a new public crop merely because an image worker finished its job.

## Keep the rendition contract small

Start with two names: `card`, at 640 × 640 pixels, and `detail`, at 1200 × 900 pixels. These are illustrative application choices, not standard marketplace sizes. They express 1:1 and 4:3 aspect ratios while also fixing actual output dimensions. A ratio alone cannot promise consistent pixel sizes.

Keep the mapping in one server-owned module. Client callers submit a source revision and a rendition name; they don't submit an arbitrary width, crop recipe, quality setting, or moderation override. For a CLI or SDK, that is a short path to a useful call. It also keeps the transformation space finite enough to review.

Configuration has to earn its place.

There is a catch: a named ratio does not specify what remains in the frame. Decide whether each slot uses cropping, padding, or rejection of unsuitable source dimensions. For this example, choose a crop and require a preview before publication. Reject sources that would require enlargement under the chosen sizing policy. If the full item must stay visible, a padded slot may be the better contract; it produces a different visual result and deserves its own fixtures. Do not let the worker choose opportunistically.

## The smallest implementation makes publication explicit

The TypeScript below is orchestration for a Node.js service. Its ports are application interfaces to implement, not calls to an existing image API. The processing adapter owns orientation handling, crop placement, dimension checks, encoding, and private storage. Its output is an immutable candidate with a revision that identifies the exact bytes reviewed. The publication adapter must enforce the same immutability.

```ts
const slots = {
  card: { width: 640, height: 640 },
  detail: { width: 1200, height: 900 },
} as const;

type Slot = keyof typeof slots;
type Decision = "approved" | "rejected" | "review";
type PrivateImage = Readonly<{ revision: string }>;

interface Ports {
  review(image: PrivateImage, policy: string): Promise<Decision>;
  renderPrivate(
    source: PrivateImage,
    size: { width: number; height: number },
  ): Promise<PrivateImage>;
  publish(input: {
    source: PrivateImage;
    candidate: PrivateImage;
    slot: Slot;
    policy: string;
  }): Promise<{ url: string }>;
}

async function prepareListingPhoto(
  source: PrivateImage,
  slotName: string,
  policy: string,
  ports: Ports,
) {
  if (!Object.prototype.hasOwnProperty.call(slots, slotName)) {
    throw new Error("Unknown rendition name");
  }
  const slot = slotName as Slot;
  const sourceDecision = await ports.review(source, policy);
  if (sourceDecision !== "approved") {
    return { state: sourceDecision } as const;
  }

  const candidate = await ports.renderPrivate(source, slots[slot]);
  const outputDecision = await ports.review(candidate, policy);
  if (outputDecision !== "approved") {
    return { state: outputDecision } as const;
  }

  const published = await ports.publish({
    source, candidate, slot, policy,
  });
  return { state: "published", url: published.url } as const;
}
```

The ordering matters. A rejection or review decision returns before publication. An adapter failure rejects the operation; the caller must retain a private processing state and expose a useful retry status. Do not translate a review timeout into approval. The example deliberately leaves authentication, upload validation, durable jobs, and storage permissions to the service boundary. Without those adapters and controls, it is not a deployable server.

The sharp edge is byte identity. If a worker recompresses an approved candidate during publication, the delivered object is no longer the exact candidate recorded by this flow. Similarly, bind the source revision to immutable content so an upload replacement cannot inherit the previous photo's approval. A URL by itself is too weak an identity for this contract. Consider a proposed replacement test: approve a trailer photo, replace the upload with a label close-up under the same listing identifier, and request the card again. The expected outcome is a new private candidate with a new review decision. Returning the old approval for the replacement would break the source binding, even if both resulting files happened to be 640 pixels wide. Test that transition at the storage boundary, where the revision is resolved to bytes, rather than trusting a matching listing identifier in an API response.

Publish the reviewed bytes.

Format selection belongs inside the processing adapter. MDN's image format guide documents differences in compression, transparency, animation, and browser support. Use those documented capabilities to choose an output policy for the clients you actually serve. The guide does not establish a safe quality value for trailer photos, and it does not substantiate moderation accuracy. Those require separate acceptance evidence.

## Benchmark coverage before tuning compression

Use the proposed three-photo fixture as a starting point, then add a source that fails review, an uncertain result, a crop that removes context, a replaced upload, and a processing failure. Record expected publication decisions before running the pipeline. Otherwise, a quick result can look successful even when it exposes the wrong object.

No measured results are claimed here.

Measure time to first published rendition from upload acceptance, including review and queue time. Record source-review latency, render latency, output-review latency, encoded bytes, and whether the required views passed their checks. Compare encodings on the same sources and crop rules. A smaller output that makes a label unreadable fails the listing's usability requirement, even if its dimensions are correct.

Also test the negative path: rejected, uncertain, and failed jobs should produce zero public candidates. Inspect storage access as well as API responses. A response saying “pending” is meaningless if the underlying object is already readable. Keep image content and label text out of routine logs; use opaque revisions and decision states for correlation.

Ship the contract with the SDK types and an unknown-name test. One small assertion catches accidental API expansion: asking for an unrecognized slot must never trigger an arbitrary transformation. That is a more useful developer experience than a configuration file full of knobs whose interactions nobody has tested.

## What I would change at scale

Move the same transitions into durable jobs. Make retries idempotent for a source revision, slot definition revision, encoder policy, and moderation policy. A changed crop definition creates a new candidate requiring review. Separate private candidate storage from public delivery, and make publication a controlled state transition with recorded evidence.

The trade-off gets sharper with volume: reviewing each candidate consumes processing capacity and delays availability. Measure that delay before changing the coverage policy. Reusing a source decision can be reasonable when its exact revision and policy still match; reusing it as proof that a different crop passed output review changes the guarantee. Make that decision explicit.

Keep the initial contract small. Consistent listing sizes come from named dimensions and deterministic processing; controlled publication comes from reviewing the source and the delivered rendition. Benchmark both against the logistics fixtures before adding another slot.

## Sources

References:

- MDN, Image file type and format guide — format capabilities, compression characteristics, and browser support: https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
