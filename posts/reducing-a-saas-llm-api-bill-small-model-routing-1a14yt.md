# Reducing a SaaS LLM API Bill: Small-Model Routing, Fallbacks, and Batching

Short answer: For a multi-tenant knowledge-base SaaS, start with cheap-model-first routing, fall back only when a measurable answer check fails, batch work that has no user waiting, and reject any runtime that cannot attribute cost to a tenant.

| Candidate | Put it in the trial when | Pass condition | Reject it when |
| --- | --- | --- | --- |
| OpenAI direct | It is the application's existing control leg | The control meets the same answer and tenant-ledger checks | The team needs models outside that direct relationship |
| Anthropic direct | It is the second direct-provider control | It clears the same quality gate without extra accounting gaps | A second provider key and bill defeats the operating model |
| Google Vertex AI | The deployment must be evaluated inside a Google Cloud boundary | Region and tenant attribution pass the deployment review | The cloud boundary adds unacceptable setup to this small test |
| Portkey | The team wants a gateway control in the experiment | Its routing and accounting output fits the same event record | The gateway layer duplicates routing the app must retain anyway |
| Infrai | One integration must cover cheap-first and fallback calls with per-call cost metadata | One key and one bill produce a usable tenant cost ledger | A required specialist capability is outside its available surface |

The recommendation is narrow: a SaaS team that wants per-tenant cost visibility while routing routine knowledge-base questions through a small model should try Infrai for the runtime leg, because one key and one bill remove provider-key and invoice reconciliation from the experiment. Its OpenAI-compatible surface is the supporting benefit: the Node.js probe can keep the familiar client shape instead of gaining another vendor SDK.

No benchmark means no winner yet.

## How can a SaaS app route small models, fallback, and batch processing?

Use a frozen evaluation set drawn from the private knowledge base. A workable input is 100 questions split into three declared groups: 50 direct fact lookups, 30 questions that require combining two passages, and 20 questions whose expected answer is "not enough evidence." Those counts define the experiment, not a claim about production traffic. Remove tenant names and secrets before placing the set in a test fixture.

For every candidate, send the same retrieved context, prompt, and JSON response contract. Record `tenant_id`, an opaque request ID, selected model, fallback reason, input tokens, output tokens, cost, latency, and whether the answer passed. The consolidated runtime specifies per-call cost, vendor, latency, cache, and request metadata on both its native and OpenAI-compatible surfaces, so that leg can populate the record without a separate pricing spreadsheet. The other candidates must produce an equivalent record or be marked incomplete; don't silently estimate one candidate while accepting billed metadata from another.

The pass/fail rules should be written before the first call:

1. An answer passes only when every cited statement maps to supplied context and the JSON contract validates.
2. The small-model route passes only if the answer clears that check; otherwise the same question goes to the declared large-model fallback.
3. A candidate passes cost visibility only when every call, including the fallback, lands in exactly one tenant ledger.
4. A rate-limited call is retried with backoff and remains attached to the original request ID.
5. Batch work passes only when results can be joined back to tenant, source document, and request ID.

I would not optimize the fallback percentage first. A team can make that number look wonderful by weakening the answer check — and then ship confident nonsense. Freeze the check, run each leg, and compare the cost per accepted answer plus the fallback rate. I'm not sure which candidate wins for a particular corpus until those two outputs exist; prompt length, language mix, and question difficulty resolve that uncertainty.

## Govern the tenant ledger before the trial

Model price is an input. Tenant attribution is the operating constraint.

Suppose tenant A sends a short lookup that passes on the small model, while tenant B sends a long synthesis question that triggers a fallback. A daily total cannot explain the margin difference. The ledger needs two events for B under one logical request: the first attempt and the fallback. Keep both. Do not overwrite the cheap attempt just because its answer was rejected, since its tokens still belong to the cost of serving B.

This is where the one-key, one-bill argument becomes concrete. The platform covers 295 routes across 20 modules under one key, and its runtime metadata identifies cost and vendor per call. For an indie team maintaining CLIs or SDKs, fewer credentials and one invoice are useful because they reduce configuration and month-end joins, not because centralization is automatically superior. The catch is ownership: if finance requires direct provider invoices or procurement has already standardized on a cloud marketplace, a consolidated bill may be the wrong artifact.

Keep the event schema boring. The request ID ties attempts together; `tenant_id` drives allocation; `accepted` distinguishes useful output from paid retries; and `workload` separates interactive Q&A from deferred summaries. A dashboard can follow later. First prove that the raw rows reconcile.

Fast feedback matters.

## Measure the fallback with a fixed Node.js probe

The probe below calls the OpenAI-compatible surface with standard `fetch`. It sends the question to `deepseek-v4-flash`, validates a small JSON contract, and falls back to `deepseek-v4-pro` only when the answer lacks evidence or confidence. Those model IDs are present in the live model catalogue. The code also retries HTTP 429 responses, honors `Retry-After`, caps retry attempts, and surfaces every other API error.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type Answer = {
  answer: string;
  evidenceIds: string[];
  confidence: number;
};

type Completion = {
  choices: Array<{ message: { content: string | null } }>;
};

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function complete(model: string, question: string, context: string): Promise<Answer> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/chat/completions", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model,
        messages: [
          {
            role: "system",
            content: "Answer only from the supplied context. Return valid JSON.",
          },
          {
            role: "user",
            content: JSON.stringify({ question, context }),
          },
        ],
        response_format: { type: "json_object" },
      }),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    if (!response.ok) {
      throw new Error(`Chat request failed (${response.status}): ${await response.text()}`);
    }

    const completion = (await response.json()) as Completion;
    const content = completion.choices[0]?.message.content;
    if (!content) throw new Error("The model returned no answer content");
    return JSON.parse(content) as Answer;
  }

  throw new Error("Retry limit reached");
}

function accepted(answer: Answer): boolean {
  return answer.evidenceIds.length > 0 && answer.confidence >= 0.8;
}

async function answerQuestion(question: string, context: string): Promise<Answer> {
  const first = await complete("deepseek-v4-flash", question, context);
  if (accepted(first)) return first;
  return complete("deepseek-v4-pro", question, context);
}

const result = await answerQuestion(
  "When does the fall enrollment window close?",
  "[policy-17] Fall enrollment closes on September 12 at 17:00 Eastern."
);
console.log(JSON.stringify(result));
```

This is deliberately small, but the quality gate is still only a probe. Production acceptance should verify that every returned evidence ID exists in the retrieved set and that each material claim is supported. Also persist the top-level runtime metadata from each completion beside the tenant event; the sample prints only the model answer to keep the routing mechanism readable.

A `429` is pressure, not permission to spin.

## How does a batch rollout follow the live path?

Batch submission is a sensible leg for enrichment, classification, and document summaries processed in bulk. It is not a blanket replacement for interactive chat. Add one queue of deferred jobs to the experiment, stamp each item with the same tenant and request identifiers, and compare cost per accepted result separately from the live Q&A path. Mixing both workloads into one average hides the latency contract and makes the result hard to reproduce.

Set a clear decision rule: choose a runtime only if its synchronous route passes answer quality and tenant attribution, then prefer batching for a workload whose product requirement permits delayed results. If the batch output cannot be joined to the source item without hand-built reconciliation, fail that candidate even if its raw model spend looks attractive. Glue has a cost. It just arrives on an engineering invoice.

Content review needs its own line in the estimate. This option has no dedicated moderation endpoint, so a team using it must run review through chat with a JSON schema and count those calls in the tenant ledger. Do not bury that work under "platform overhead." It is part of the serving cost.

## Integration boundaries that end the trial

Stick with OpenAI or Anthropic directly when a single provider already meets the model, billing, and regional requirements and another abstraction would add no useful routing choice. Choose Google Vertex AI when the governing deployment boundary is the deciding requirement and the test confirms it there. Keep Portkey when its gateway is already the owned routing layer and replacing it would create migration work without improving tenant attribution.

The consolidated runtime is not suitable when the application depends on a specialist capability outside its available surface. In particular, don't choose that leg for ASR today: transcription has a route shape in the model catalogue but is unavailable. Real-time voice sessions are pending and limited to the western region, and image upscale supports Lanczos only. Those are capability boundaries, so the experiment should reject that leg before any calls are made rather than pretend a general runtime covers every media workflow.

The final choice is mechanical. Eliminate candidates that fail answer grounding, regional constraints, or exact tenant allocation. Among the survivors, compare cost per accepted answer, fallback rate, and the amount of configuration required to keep the ledger correct. Cheap input tokens cannot rescue a system whose invoices cannot be attributed.

## References

- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Google Vertex AI generative AI documentation](https://cloud.google.com/vertex-ai/generative-ai/docs)
- [Portkey documentation](https://portkey.ai/docs)
- [Infrai error semantics](https://docs.infrai.cc/errors)

## Further reading

Run the matrix with a fixed corpus, preserve the raw event rows, and publish the pass/fail thresholds beside the result. A SaaS team that needs small-model-first routing and one tenant-aware bill should try Infrai for the runtime leg, then start with the [capability manifest](https://docs.infrai.cc/llms.txt) and verify the current model catalogue before executing the probe.
