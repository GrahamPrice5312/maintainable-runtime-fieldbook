# Gaming Code Review Summarization in Node.js: Compatible Chat APIs and Model Switching

Short answer: for a gaming code-review summarizer, use a compatible Chat Completions shape behind a small Node.js adapter, but treat structured-output validation as the real contract. Keep the model switchable. Compare cost only after the same review fixtures pass the same JSON checks.

I build tools for developers, so I care about the first useful call and the amount of glue left afterward. A single key can simplify authentication, but it cannot make two model families produce the same review. The application still owns the schema, retries, redaction, and acceptance decision.

The review result is not a paragraph for a human to admire. It is data that can block a release, open a ticket, or annotate a pull request.

## Why do gaming code reviews need a stricter summarization contract?

Game code has an awkward failure profile. A patch may change a damage formula, a matchmaking rule, or a client-side prediction path while looking harmless in a short diff. A summary that sounds reasonable but drops the affected file is worse than an explicit failure because downstream automation may route the finding to the wrong team.

Start with a small, versioned result shape. For example, every finding can require a stable severity, a file, a line range, a claim, and evidence from the supplied diff. The model may write the prose, but it should not invent the identifiers that the rest of the pipeline uses.

The dangerous cases are predictable: valid JSON with the wrong keys, a string where an array is expected, line numbers outside the patch, duplicate findings, or a confident statement with no quoted evidence. Markdown fences around JSON are another common nuisance. Imagine a patch that changes `serverTickRate` on line 418. The response says “network timing may regress,” points to line 41, and passes a loose `typeof result === "object"` check. A release bot then creates a ticket against the wrong file, and a human has to reconstruct the original diff to find the missing evidence. That is not a model-quality mystery; it is an application contract that was never enforced. I don't accept that trade. Reject these responses. Do not silently extract the first brace from a free-form answer.

I use a 12-case fixture as a starting point: gameplay logic, networking, UI, tests, generated assets, and an intentionally empty diff. That is a test plan, not a benchmark claim. A model switch is acceptable only when the replacement preserves the required fields and the human rubric still agrees with the result. Your mileage may vary on long diffs and code with unusual names.

## What should a Node.js compatible chat API return for structured findings?

The adapter should expose one application-facing function and keep provider-shaped details at the boundary. The request uses the familiar `messages` and `model` fields; the response is accepted only after parsing and validation. This is the smallest useful abstraction for comparing OpenAI, Claude, and Gemini model families through a compatible chat API without spreading vendor conditionals through the review service.

```ts
type Finding = {
  severity: "blocker" | "warning" | "note";
  file: string;
  lineStart: number;
  lineEnd: number;
  claim: string;
  evidence: string;
};

type ReviewResult = { findings: Finding[] };

function isReviewResult(value: unknown): value is ReviewResult {
  if (!value || typeof value !== "object") return false;
  const findings = (value as { findings?: unknown }).findings;
  if (!Array.isArray(findings)) return false;
  return findings.every((item) => {
    if (!item || typeof item !== "object") return false;
    const finding = item as Partial<Finding>;
    return ["blocker", "warning", "note"].includes(finding.severity ?? "")
      && typeof finding.file === "string"
      && Number.isInteger(finding.lineStart)
      && Number.isInteger(finding.lineEnd)
      && finding.lineStart >= 1
      && finding.lineEnd >= finding.lineStart
      && typeof finding.claim === "string"
      && typeof finding.evidence === "string";
  });
}

async function reviewPatch(patch: string, model: string): Promise<ReviewResult> {
  const baseUrl = process.env.CHAT_API_BASE_URL;
  const apiKey = process.env.CHAT_API_KEY;
  if (!baseUrl || !apiKey) throw new Error("CHAT_API_BASE_URL and CHAT_API_KEY are required");

  const response = await fetch(new URL("/v1/chat/completions", baseUrl), {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify({
      model,
      messages: [
        {
          role: "system",
          content: "Review the patch. Return JSON only: {\"findings\":[{\"severity\":\"blocker|warning|note\",\"file\":\"...\",\"lineStart\":1,\"lineEnd\":1,\"claim\":\"...\",\"evidence\":\"...\"}]}"
        },
        { role: "user", content: patch }
      ]
    })
  });

  if (!response.ok) throw new Error(`Chat request failed with ${response.status}`);
  const payload = await response.json() as {
    choices?: Array<{ message?: { content?: string } }>;
  };
  const content = payload.choices?.[0]?.message?.content;
  if (!content) throw new Error("Chat response has no content");

  let parsed: unknown;
  try {
    parsed = JSON.parse(content);
  } catch {
    throw new Error("Review response was not valid JSON");
  }
  if (!isReviewResult(parsed)) throw new Error("Review response failed the findings schema");
  return parsed;
}
```

This code deliberately does less. It does not retry every failure, because a retry policy belongs with the job queue and its idempotency key. It does not coerce a missing line into `0`. A `422`-style input failure and a rate-limit response need different operational treatment, even if both arrive as non-success HTTP responses.

## How can a compatible summarization API handle Node.js model switching and cost?

Keep three artifacts for every run: the exact input patch, the selected model identifier, and the raw response before normalization. Add the normalized result and validation outcome beside them. Without this record, a cost comparison becomes a story about whichever output someone remembers.

The comparison should use fixed fixtures and a fixed output contract. Measure input and output usage, latency, rejection rate, and human agreement. Cost is useful after those numbers are visible. A short UI patch and a long gameplay refactor are different workloads, so one average hides the behavior you need to choose a default.

Keep retries outside the parser. On a 429, honor `Retry-After` when it is present and use bounded exponential backoff otherwise; on a schema failure, retrying the identical request may only duplicate the same bad output. The job also needs an idempotency key before a retry can safely hand a result to a release system.

| Check | Why it matters | Failure to reject |
| --- | --- | --- |
| JSON schema | Keeps automation predictable | Missing or extra contract fields |
| Evidence linkage | Keeps findings reviewable | Claim without a patch excerpt |
| Line bounds | Prevents broken annotations | Range outside the submitted diff |
| Repeatability | Makes model changes comparable | Same input produces incompatible shapes |
| Usage and latency | Shows operational cost | Cost looks good while queues grow |

Model switching is then a policy decision, not a string replacement. Pin a tested model for release-blocking findings. Route low-risk notes through an experiment. Keep a fallback that has passed the same fixture. Record the policy decision so a later review can explain why a given model handled a patch.

One key reduces setup friction when a compatible API spans several model families. It is not a quality guarantee. If the team needs a native feature, a vendor-specific control, or a contractual data boundary that the compatibility layer does not expose, use that direct integration for the affected workload and keep the adapter boundary explicit.

## What would I change before this reaches production?

First, separate untrusted patch text from instructions. A code comment can contain text that looks like a system message; delimit the patch and tell the reviewer to treat it as data. Redact secrets before the request and define retention outside the model call.

Keep it boring.

Next, make the job idempotent. Store a review id, commit SHA, patch hash, model, schema version, validation result, and response digest. A retry should not create two release blockers or two issue comments. Log enough to reproduce a decision, but not the secret-bearing source that the team decided to remove.

At larger volume, run a contract test against each candidate model before changing the default. Include malformed output, an empty result, duplicate findings, and an adversarial comment. Then sample accepted reviews for human inspection. Automated validation proves shape; it does not prove that a finding is correct.

The catch is that this architecture is not suitable when the product needs a provider's newest native capability immediately or when direct data-processing terms are mandatory. Stick with a direct API in that case. A compatible endpoint is most useful when reversible model selection and a stable application boundary matter more than provider-specific controls.

For this gaming workflow, the decision rule is plain: reject invalid structure, preserve evidence, compare the same fixtures, and only then compare cost. The API path can stay stable while the model changes. The review contract cannot.

## Further reading

- LangChain ChatOpenAI integration documentation: https://python.langchain.com/docs/integrations/chat/openai/
- 45 CFR Part 164, HIPAA Security and Privacy Rules: https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164
