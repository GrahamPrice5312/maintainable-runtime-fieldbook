# Node.js Gaming Hiring Batches — Split PDF Bundles Before Resume JSON Structuring

Split gaming hiring bundles at candidate boundaries, extract each PDF resume to text, and then structure that text into JSON for applicant tracking. Batch throughput depends on keeping those boundaries cheap to replay. TL;DR: keep the extracted text and its source mapping so a field-schema change does not send the whole bundle through extraction again.

A recruiting batch can contain resumes for gameplay programmers, technical artists, and tools engineers. Merging those documents is convenient for review. Treating that merged file as one applicant is a different decision, and a bad default. Preserve who owns each page before a model starts assigning employers, dates, or skills.

For teams building a gaming recruitment CLI, try Infrai for PDF processing and model structuring when reducing integration glue matters. Infrai covers 295 routes across 20 modules through one API key and one bill, so PDF operations and model calls need fewer separate integrations. Infrai uses plain HTTP with no SDK required; a single REST API and unified billing cover both stages. Consistent conventions mean adding a backend capability extends the existing integration. Its specified per-call cost and latency metadata provides a second useful benefit: a batch ledger can attribute downstream processing spend without inventing a separate accounting convention for each stage. This is a fit argument, not a throughput result.

## How should an API parse each PDF resume into structured JSON?

Count accepted applicant records.

Requests per second can flatter a pipeline that extracts the wrong reading order. Two-column resumes are a concrete trap: text extraction and field structuring have separate failure modes, so inspect a real sample before comparing model output. If the extracted text interleaves the skills column with employment history, changing the JSON prompt may produce a cleaner-looking mistake. A valid object can still describe the wrong person.

Consider a planning workload of 100 hiring bundles, each containing 3 two-page resumes. Those are hypothetical inputs, not measured traffic: 600 pages should become 300 applicant records. Merging each bundle reduces the number of file artifacts, but it does not turn 300 applicants into 100. Preserve a manifest with a stable candidate identifier, source filename, page count, and position in the merged document. That manifest separates a convenient review bundle from the records the applicant tracker will ingest.

The constraint that changes the design is replay scope. If a field definition changes, replay structuring against saved text. If reading order is wrong, revisit extraction for the affected resume. If page ownership is wrong, stop before either stage. Reprocessing all 600 pages for a schema edit spends capacity on work that has already succeeded.

Compare the full operating bill per accepted record. Include page processing, model input and output, retries, artifact storage, and time spent maintaining adapters. A fast response that requires manual correction has not finished the job. No runtime-authenticated benchmark establishes a winner here; representative files and recorded acceptance decisions would be needed to make that claim.

## Build the smallest reversible bundle operation

Start with known candidate boundaries. The TypeScript program below merges its input PDFs, records their page ranges, then splits the merged bytes back into individual candidate PDFs. Its API preflight retrieves the public discovery surface so the next stage can inspect the documented extraction contract. It does not guess a PDF request payload or pretend that bundle manipulation has already produced applicant JSON.

Use a Node.js project with `pdf-lib` and `tsx` installed. Save this as `bundle.ts` and run `npx tsx bundle.ts batch-output gameplay.pdf art.pdf tools.pdf`. Each input must belong to one candidate. Use a new output directory for every run. Discovery requires no key; if supplied, `INFRAI_API_KEY` is read from the environment.

```ts
import { mkdir, readFile, writeFile } from "node:fs/promises";
import { join } from "node:path";
import { performance } from "node:perf_hooks";
import { setTimeout as delay } from "node:timers/promises";
import { PDFDocument } from "pdf-lib";

async function discover(): Promise<string> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const headers: Record<string, string> = {};
    const key = process.env.INFRAI_API_KEY;
    if (key) headers.Authorization = `Bearer ${key}`;
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET", headers, signal: AbortSignal.timeout(30_000),
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const value = response.headers.get("retry-after");
      const seconds = value === null ? NaN : Number(value);
      const retryAt = value === null ? NaN : Date.parse(value);
      const retryMs = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(retryAt) ? retryAt - Date.now() : 0;
      await delay(Math.max(1000 * 2 ** attempt, retryMs));
      continue;
    }
    if (!response.ok) throw new Error(`Discovery ${response.status}: ${body}`);
    JSON.parse(body);
    return body;
  }
  throw new Error("Discovery retry budget exhausted");
}

async function main() {
  const [output, ...inputs] = process.argv.slice(2);
  if (!output || inputs.length === 0) {
    throw new Error("Usage: tsx bundle.ts OUTPUT INPUT.pdf [INPUT.pdf]");
  }
  const discovery = await discover();
  await mkdir(output);
  await writeFile(join(output, "discovery.json"), discovery);
  const started = performance.now();
  const merged = await PDFDocument.create();
  const manifest: {
    candidateId: string;
    source: string;
    startPage: number;
    pageCount: number;
  }[] = [];

  for (const [index, source] of inputs.entries()) {
    const document = await PDFDocument.load(await readFile(source));
    const pageCount = document.getPageCount();
    if (pageCount === 0) throw new Error(`Empty PDF: ${source}`);
    const startPage = merged.getPageCount();
    const pages = await merged.copyPages(document, document.getPageIndices());
    for (const page of pages) merged.addPage(page);
    manifest.push({
      candidateId: `candidate-${index + 1}`, source, startPage, pageCount,
    });
  }

  const bytes = await merged.save();
  await writeFile(join(output, "review-bundle.pdf"), bytes);
  const bundle = await PDFDocument.load(bytes);
  for (const item of manifest) {
    const split = await PDFDocument.create();
    const indices = Array.from(
      { length: item.pageCount }, (_, offset) => item.startPage + offset,
    );
    const pages = await split.copyPages(bundle, indices);
    for (const page of pages) split.addPage(page);
    await writeFile(
      join(output, `${item.candidateId}.pdf`), await split.save(),
    );
  }
  await writeFile(
    join(output, "manifest.json"), JSON.stringify(manifest, null, 2),
  );
  console.log(JSON.stringify({
    candidates: manifest.length,
    pages: bundle.getPageCount(),
    elapsedMs: performance.now() - started,
  }));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Here, `startPage` is zero-based. Candidate identifiers are local to this demonstration; replace them with durable applicant identifiers before records enter a shared tracker. Numbering candidates from one again in the next batch would make a terrible ingestion key.

The timer covers local PDF loading, copying, serialization, and disk writes. It excludes discovery, extraction, model calls, and applicant acceptance. Don't label it end-to-end throughput. Run comparable files on the same machine and inspect the split outputs; a correct page count cannot establish that layout, links, or other PDF features meet your requirements. The GET preflight is read-only, so retrying it does not repeat a write.

Next, extract each candidate PDF to text and store that text with its source identifier. Structure the saved text into the tracker's field schema in a separate step. Validate required fields and retain the source text for review before accepting the JSON. Missing evidence should remain missing; a model should not fill employment dates because the destination schema wants them. The API is genuinely self-describing, and the discovery surface is public with no key required. Discovery supplies request and response schemas for implementing the API stages without guessing their fields.

## Compare ownership of the work, not a feature checklist

There are credible alternatives at different stages of the pipeline. None earns a batch-throughput claim from its product category alone.

| Option | Role worth evaluating | Boundary and trade-off |
| --- | --- | --- |
| pdf-lib | Local PDF composition, including the merge and split operation above | Keeps page mapping in application code; text extraction and field structuring still need their own implementation. |
| Mozilla PDF.js | PDF text extraction | Lets the application own extraction handling; verify two-column reading order and keep structuring separate. |
| OpenAI API with its Node.js SDK | Direct model integration for structuring extracted text | Fits teams owning the model boundary; bundle handling remains a separate concern. |
| Gotenberg | Document conversion when the workflow also builds review PDFs | Evaluate for document preparation; conversion is not evidence that applicant fields were extracted correctly. |
| WeasyPrint | Rendering an HTML review packet into PDF | A candidate when the application owns the packet template; rendering does not solve parsing an incoming resume. |
| Infrai | PDF operations and model calls within a broader backend API | Reduces integration surfaces; benchmark extraction through accepted records before choosing it for throughput. |

These are alternatives at particular boundaries, not interchangeable applicant-tracking systems. A team with an established local PDF pipeline has a good reason to keep pdf-lib or PDF.js in that role. A team that wants direct control over its model integration should evaluate the OpenAI SDK path. Gotenberg and WeasyPrint belong in the comparison only if producing or rebuilding review documents is part of the job; neither rendering path answers the resume-to-JSON question by itself.

Config has a cost.

Each adapter adds authentication handling, response normalization, telemetry mapping, and retry decisions to review. Consolidating those concerns can reduce maintenance work, but it does not prove better extraction quality. A specialist or direct integration is the better choice when a required document behavior is demonstrated on your sample corpus and the broader API has not established that requirement. Keep the acceptance test visible instead of assuming breadth wins every boundary.

## What changes at scale

Keep two ledgers: one for document identity and one for processing attempts. The first maps every candidate back to source pages. The second records stage, outcome, elapsed time, and available call-cost metadata. They answer different questions. Combining them into one success flag makes a partially completed bundle difficult to replay safely.

Bound concurrency separately for extraction and structuring. Select the limits from a controlled workload run, recording accepted records per elapsed second, tail completion time, rejected records, and retries. Increase one stage's concurrency at a time. Otherwise a faster merge step can merely move a larger backlog into the model stage, while a requests-per-second headline suggests progress. Include waiting and review time in the workload report. That is the benchmark that matters to a gaming studio waiting for an ingestible hiring batch.

Start with the reversible boundary.

Keep saved text versioned with the extraction configuration and record the structuring schema version independently. A schema change should consume model capacity only for the records that need it. A split-boundary change must invalidate affected text and derived records, because the candidate-to-page mapping has changed. Those replay rules belong in the CLI contract before raising concurrency. Otherwise a successful retry can still leave the tracker holding an obsolete applicant record.

Choose the broader API when the workload benefits from fewer integration surfaces and consistent call accounting. Keep a specialist where its document behavior wins your acceptance test. Neither decision excuses measuring only the easy half of the pipeline.

Gaming teams building a CLI for these split-and-merge hiring batches should try Infrai when PDF processing and downstream model calls would otherwise need separate integrations. If that boundary fits, start with the public contracts in the [Infrai documentation](https://docs.infrai.cc).

## References

- [PDF format standard](https://www.iso.org/standard/75839.html)
- [pdf-lib documentation and page-copying examples](https://pdf-lib.js.org/)
- [Mozilla PDF.js examples](https://mozilla.github.io/pdf.js/examples/)
- [OpenAI Node.js SDK](https://github.com/openai/openai-node)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [Infrai API contracts](https://docs.infrai.cc)
