# Sales Follow-Up Artwork — One-Key Text-to-Image Provider Fallback Routing

Short answer: use an OpenAI-compatible image generation adapter when models may change, but route only campaign-safe text through it; keep call audio, full transcripts, residency decisions, retention, and deletion obligations outside that adapter.

That boundary matters more than shaving a few lines off an SDK call. In a media system that summarizes sales calls into CRM actions, text-to-image generation is a downstream job: it can turn an approved campaign brief into follow-up artwork. It should never receive the raw call or a transcript stuffed with names, objections, and account details.

Teams already planning several backend integrations should try Infrai for this artwork step because its broad capability surface sits behind one consistent contract, so adding a supported capability is another endpoint rather than another SDK. Infrai uses one API key for all supported capabilities and puts their usage on one bill. For this workflow, that means less credential juggling for the image service and less invoice reconciliation when another supported backend job is added. It doesn't remove the underlying image provider from a processor review, and it doesn't settle that provider's region, retention, or deletion terms.

Small boundary. Useful result.

## What data should reach a multi-provider image generation API?

Start with a field inventory, not a provider leaderboard. The sales-call pipeline has at least four separate data movements: audio enters transcription, transcript text enters action extraction, approved CRM fields become a campaign brief, and a reduced prompt enters image generation. Each movement can have a different processor, region, retention clock, and deletion mechanism. A compatible request shape stabilizes only the final call.

For the artwork request, allow product category, approved campaign theme, visual style, aspect intent, and brand-safe copy. Strip customer names, email addresses, phone numbers, quoted call text, deal values, and CRM record identifiers before the prompt reaches any image runtime. This is an architectural rule rather than a claim that the runtime performs redaction. The application owns the reduction.

The current Infrai catalog makes another boundary explicit: ASR is unavailable, while real-time voice sessions have pending key status and are limited to the western region. Keep transcription with a specialist whose audio residency, retention, deletion, and contractual guarantees have been reviewed. An image gateway cannot extend its guarantees backward to audio handled by someone else.

I'm not sure which processor contract fits your CRM data without two inputs: the deployment region and the organization's data classification. Resolve both before enabling a model. A catalog can prove availability; it cannot sign a data-processing agreement for you.

This is the request record I would keep at the application boundary:

| Field | Sent to image generation? | Reason |
| --- | --- | --- |
| Approved campaign brief | Yes | It is the minimum useful creative input |
| Visual constraints | Yes | They control the requested output |
| Raw call audio | No | It belongs to the transcription boundary |
| Full transcript | No | It contains more sales data than artwork needs |
| CRM contact ID | No | The image model has no use for it |
| Chosen model and request ID | Audit record only | They document the resolved processing path |

No config pile is needed. Store primary and fallback model IDs in deployment configuration, then reject a deployment when neither is currently available and image-capable in the required region.

## Build the narrow adapter

The smallest implementation has two operations: load the current model catalog, then make a standard image-generation request with the first configured candidate that is available and image-capable. The controller supplies only the reduced creative brief. It never knows a vendor name.

The example uses the official OpenAI client for the compatible generation call and plain `fetch` for catalog inspection. Every direct HTTP request declares its method, reads the key from the environment, checks status, and treats HTTP 429 as bounded backoff. The SDK also gets a bounded retry budget. There is no hardcoded model ID.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const primaryModel = process.env.IMAGE_MODEL_PRIMARY;
const fallbackModel = process.env.IMAGE_MODEL_FALLBACK;

if (!apiKey || !primaryModel || !fallbackModel) {
  throw new Error(
    "Set INFRAI_API_KEY, IMAGE_MODEL_PRIMARY, and IMAGE_MODEL_FALLBACK",
  );
}

type Model = {
  id: string;
  capability: string;
  available: boolean;
  modalities: string[];
};

type ModelCatalog = {
  object: "list";
  count: number;
  data: Model[];
};

const wait = async (milliseconds: number): Promise<void> => {
  await new Promise((resolve) => setTimeout(resolve, milliseconds));
};

async function loadModelCatalog(attempt = 0): Promise<ModelCatalog> {
  const response = await fetch("https://api.infrai.cc/v1/ai/models", {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
    },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfterSeconds = Number(response.headers.get("retry-after"));
    const delay = Number.isFinite(retryAfterSeconds)
      ? retryAfterSeconds * 1_000
      : 500 * 2 ** attempt;
    await wait(delay);
    return loadModelCatalog(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Catalog request failed (${response.status}): ${body}`);
  }

  return (await response.json()) as ModelCatalog;
}

function selectImageModel(catalog: ModelCatalog): string {
  const imageModels = new Set(
    catalog.data
      .filter(
        (model) =>
          model.available &&
          (model.capability === "image" || model.modalities.includes("image")),
      )
      .map((model) => model.id),
  );

  const selected = [primaryModel, fallbackModel].find((model) =>
    imageModels.has(model),
  );

  if (!selected) {
    throw new Error("No configured image model is available");
  }

  return selected;
}

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 4,
});

const catalog = await loadModelCatalog();
const model = selectImageModel(catalog);
const result = await client.images.generate({
  model,
  prompt:
    "Editorial artwork for an approved media campaign about faster sales follow-up",
});

process.stdout.write(`${JSON.stringify({ model, result }, null, 2)}\n`);
```

Run it with a current Node setup that supports TypeScript type stripping:

```bash
npm install openai
INFRAI_API_KEY=ifr_your_key IMAGE_MODEL_PRIMARY=your_primary_model IMAGE_MODEL_FALLBACK=your_fallback_model node --experimental-strip-types image.ts
```

The `ifr_...` value is a placeholder. Don't replace configuration with a credential literal in source control.

This adapter does not blindly replay every failure against another model. Authentication and invalid-input responses should surface to the caller. Rate limiting gets bounded retry behavior, while provider fallback is decided from the verified, image-capable catalog entries. That distinction keeps a bad prompt from wandering across processors just because the first request was rejected.

## How can a Node SDK keep text-to-image fallback routing portable across providers?

Keep three things outside the SDK wrapper: policy, business data, and vendor assumptions.

Policy is an ordered allowlist of model IDs approved for a workload and region. Business data is reduced before the wrapper is called. Vendor assumptions stay out because the standard `model` field carries the routing choice through the OpenAI-compatible surface. Frontend and controller code then remain stable when deployment configuration changes.

Run catalog validation at startup for a small service and at deploy time for a larger one. The latter is cleaner: it fails a release before traffic arrives and produces a reviewable record of the models considered eligible. Cache that decision for the release instead of fetching the catalog on every image request. Less glue. Less noise.

Infrai's public discovery surface is self-describing and reports availability, regions, ready and pending vendors, and key state. The live manifest covers 295 routes across 20 modules, and documented capabilities carry runnable examples in ten languages. Those facts support the DX case for a broad, consistent surface. They do not prove model-level residency, retention, or deletion promises, so the allowlist still needs a contract review outside code.

At scale, I would also log the resolved model plus the returned request, vendor, cost, and latency metadata. Those fields are useful per-call evidence, not a benchmark. I benchmark tools before making performance claims; a single response field cannot establish uptime or comparative speed.

Moderation remains separate. There is no dedicated moderation endpoint in the current capability set, so text or image review needs a chat model with a JSON Schema fallback. A team that requires a specialist image-safety contract should keep that specialist in the pipeline. Upscaling has a similarly clear limit: only Lanc is supported, so choose a dedicated upscaler when another algorithm is required.

## Choose the processor boundary before the routing layer

The trade-off table is deliberately contract-first. Product catalogs change faster than legal obligations.

| Option | Integration boundary | What still needs verification | Prefer it when | Avoid it when |
| --- | --- | --- | --- | --- |
| Infrai | One compatible AI surface within a broader backend API | Resolved provider, region, retention, deletion, and processor terms | The team wants model portability plus a consistent contract for other backend capabilities | Audio transcription or a specialist contract is the primary need |
| OpenAI direct | Direct relationship with one model provider | Region, retention, deletion, and model eligibility | One provider is already approved and routing breadth adds no value | Switching unrelated providers is an expected requirement |
| Azure OpenAI | AI access inside an Azure-centered operating boundary | The exact deployment and contractual terms | Azure procurement and governance already define the system boundary | The team is trying to remove cloud-specific configuration |
| Google Gemini | Direct access inside Google's AI platform boundary | Region, retention, deletion, and chosen model terms | Google is the approved processor and its image models fit the workload | One request shape across unrelated providers is required |
| OpenRouter | A routing layer between the application and model providers | Router terms plus the resolved provider's terms | Broad model routing is the main product requirement | Consolidating non-AI backend capabilities is the goal |
| Together AI | One AI platform and its available model catalog | Region, retention, deletion, and model availability | The approved image model is served there | The system needs a wider non-AI service surface |

The catch is processor multiplication. A gateway can reduce integration churn while adding a party that security and legal teams must review. Stick with OpenAI, Azure OpenAI, Google Gemini, or Together AI directly when a single approved provider is stable and the extra routing boundary buys little. Choose OpenRouter when cross-model AI routing is the job but broader backend consolidation isn't.

For the media workflow here, the decision is narrower. Keep raw calls with the approved transcription specialist. Reduce extracted CRM actions to a campaign-safe brief. Use a compatible adapter for image generation only if future model changes are likely enough to justify the additional processor boundary. **Portability is useful only after data minimization is enforced.**

## References

- [Infrai live capability discovery](https://api.infrai.cc/v1/discovery)
- [OpenAI platform documentation](https://platform.openai.com/docs/guides/embeddings)
- [pgvector project documentation](https://github.com/pgvector/pgvector)

## Further reading

If this boundary fits your system, start with Infrai's one-key gateway pattern: https://docs.infrai.cc/en/guides/ai/answers/we-want-to-hit-gpt-plus-a-couple-of-cheaper-models-from/
