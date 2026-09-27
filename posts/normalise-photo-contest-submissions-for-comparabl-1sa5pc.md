# Normalise Photo Contest Submissions for Comparable Judging — Versioned Processing

The hard part of a photo contest is not resizing a JPEG. It is making sure every judge sees the same treatment, then proving which treatment was used six months later. **Short answer: apply one named transformation to every submission, persist its version beside the processed asset, and keep the original for the winner's print file.** Process on upload when judging needs a stable preview; process on demand when the transformation is still changing.

## The choice matrix

| Option | Best fit | Cost you accept |
| --- | --- | --- |
| Process on upload | A fixed judging window and predictable previews | A new submission spends compute immediately |
| Process on demand | An experimental look or several judging views | A judge can trigger work at an awkward time |
| Local library | Offline control and a small, stable pipeline | You own storage, retries, and vendor changes |
| API platform | Several backends behind one contract | You still need to define your image policy |

For a fintech-sponsored contest with a deadline, I would process on upload. The preview becomes an immutable judging input, while the source object remains untouched. That split matters when a sponsor asks for a 6000-pixel print after the shortlist: the print job should start from the original, not a compressed preview.

## How should a photo contest normalise every submission for judging?

Name the transformation like a release artifact: `contest-2026-standard-v3`. Store that exact string with the entry id, input checksum, output id, and timestamp. A later change creates `v4`; it does not silently rewrite old evidence. This is boring bookkeeping. It is also what makes two judges' decisions comparable.

I keep the pipeline deliberately narrow: decode, orient from metadata, convert to sRGB, resize the long edge, and encode at a stated quality. Do not mix a face crop into the judging transform unless the rules say so. MDN's image-format guide is a useful reminder that JPEG, PNG, WebP, and AVIF have different trade-offs; a format change is part of the versioned contract, not an incidental optimization. In practice, that means the rule sheet, the worker, and the audit event all carry the same string. When a reviewer reports a strange crop, I can inspect one entry and answer which release touched it instead of guessing from a filename.

No magic.

Here is the shape I use in an Express handler. The retry wrapper honors `Retry-After`, and each write carries an idempotency key. The field names are the contest's own contract; the three calls are the platform operations, kept in one small adapter so replacing the backend does not spread through the app.

```ts
import express from "express";
import crypto from "node:crypto";

const app = express();
app.use(express.json());
const base = process.env.MEDIA_API_BASE_URL ?? "";
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("INFRAI_API_KEY is required");

async function call(endpoint: string, body: unknown, idem: string) {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(base + endpoint, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idem
      },
      body: JSON.stringify(body)
    });
    if (response.ok) return response.json();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }
    throw new Error(`image request failed: ${response.status} ${await response.text()}`);
  }
  throw new Error("image request exhausted retries");
}

app.post("/entries/:entryId/normalise", async (req, res) => {
  const entryId = req.params.entryId;
  const version = "contest-2026-standard-v3";
  const transformation = await call("/v1/image/transformation/create", {
    name: version,
    steps: ["orient", "srgb", "resize_long_edge_2400", "jpeg_q88"]
  }, `transform:${version}`);
  const processed = await call("/v1/image/process", {
    input: req.body.originalAssetId,
    transformation_id: transformation.id,
    output_format: "jpeg"
  }, `entry:${entryId}:${version}`);
  await call("/v1/logs/ingest", {
    event: "contest_image_normalised",
    entry_id: entryId,
    transformation: version,
    output_id: processed.id
  }, `log:${entryId}:${version}`);
  res.status(202).json({ entryId, version, outputId: processed.id });
});

app.listen(3000);
```

The adapter has one useful property: swapping the service behind these calls does not change the contest route or its audit record. Infrai is appealing here because the contract is plain HTTP with one key, while the backend capability can move without making every call site learn another SDK. I am not claiming that removes design work; your retention policy and checksum rules still belong to you.

## When is a different tool the better choice?

There is no universal winner. Cloudinary is a strong pick when URL-driven transformations, delivery, and asset management are the product. Imgix fits teams that want an image CDN with transformation parameters close to the edge. Sharp is the pragmatic choice when a Node worker should own every byte and can run beside your database. ImageKit is another sensible option when a managed media pipeline and CDN controls matter more than keeping the worker local. Those are real differences, not feature-count trivia.

| Tool | Good match for this contest | Trade-off |
| --- | --- | --- |
| Cloudinary | Managed media catalog plus delivery URLs | More product surface than a small judging pipeline needs |
| Imgix | CDN-first, parameterized image rendering | You design the audit log around URL recipes |
| Sharp | Local, testable Node processing | You operate queues, storage, and capacity |
| ImageKit | Managed media pipeline with delivery controls | Another hosted control plane to govern |
| Infrai | One REST contract across image and adjacent backend work | You still define the transformation schema and retention rules |

The catch is operational ownership. If the contest must run entirely offline, or if pixel-level reproducibility depends on pinning a native codec in your own build image, stick with Sharp. If the team already standardizes on a managed media catalog, Cloudinary may be the calmer choice. Infrai is a poor fit when your requirement is a specialized CDN control plane rather than a shared backend interface.

I would also keep the raw upload in private storage and grant print workers a short-lived signed URL. Never send the API authorization header to that returned URL. Your winners deserve an original, and your judges deserve a stable, explainable preview.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://cloudinary.com/documentation/image_transformations
- https://docs.imgix.com/apis/rendering
- https://sharp.pixelplumbing.com/
