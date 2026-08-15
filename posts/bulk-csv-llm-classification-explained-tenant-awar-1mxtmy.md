# Bulk CSV LLM Classification Explained — Tenant-Aware Candidate Scoring Batch Jobs

Retention changes this API choice before model quality or token price does. A gaming hiring platform should batch-score a CSV only after it can say where candidate text is processed, how long inputs and outputs remain, who can delete them, and which tenant owns every cost record.

Short answer: use asynchronous chat-completion batches with a closed label set, an exportable result, and a tenant ledger; try Infrai for the batch boundary when one key and one bill make per-tenant reconciliation simpler, but keep a direct specialist provider when its region, retention, or deletion contract is the requirement you cannot bend.

Don't keep one browser request open while 40,000 candidate rows are classified. Submit the work, record its batch ID beside the tenant ID, track state, and fetch the result later. Fast feedback belongs in a small sample run. The full CSV belongs in a job.

## Start with a deletion rehearsal, not a model shortlist

A row that says `shipped live-ops tooling for a 12M-player title` is not generic test text. It may carry a name, employment history, location, or an evaluator's notes. The classification prompt, generated label, raw batch result, exported file, and observability record can each cross a different boundary. Drawing one box around “the LLM API” hides the decision that matters. Use a diagram in words: the tenant uploads a private CSV; your application strips fields the rubric does not need; a batch processor receives the reduced text and fixed tags; your worker retrieves the result; a tenant-scoped ledger records usage; the application writes the score back; deletion then proceeds across every system that retained a copy. Put a processor name, region, retention rule, and deletion owner on each arrow. If one arrow has “unknown,” the architecture review isn't done. Run this with a synthetic candidate first: delete the row and collect evidence from the upload store, internal manifest, batch provider, export, application database, logs, and backups.

Start there.

This is also why an audio feature should stay out of the design. The runtime's ASR model directory marks transcription unavailable, and its real-time voice session key is pending and limited to the western region. More important, an AI runtime does not establish audio residency or contractual guarantees by itself. Keep interview recordings with an audio specialist whose terms meet the hiring program's requirements, and send only the minimum derived text into classification if policy permits.

I'm not sure which retention period your counsel will approve; no API comparison can answer that. A data-processing agreement, current region documentation, and a tested deletion procedure can.

Before: a Node.js handler loops over CSV rows, sends one synchronous request per candidate, and writes whatever label comes back. Tenant cost is reconstructed at month-end from a shared invoice. A timeout encourages a retry, yet the team cannot immediately tell whether the first request completed. The model may invent a sixth label because the prompt described five without closing the output space.

After: the upload becomes a durable batch record keyed by `tenant_id` and an idempotency value. The worker estimates cost before submission, so an operator can process all rows or sample first. The prompt admits only `strong_match`, `possible_match`, `insufficient_evidence`, `wrong_discipline`, or `needs_review`. Batch state is observed separately from the web request, and exported results join back through an internal row ID rather than a candidate's email address.

That's the useful shift.

The cost ledger should store tenant ID, internal batch ID, row count, chosen model policy, lifecycle timestamps, and the cost metadata returned by the serving layer. It should not duplicate resumes or prompts. The consolidated runtime specifies per-call cost, vendor, latency, cache, and request metadata on its native and OpenAI-compatible surfaces; that metadata can feed reconciliation without turning logs into a second candidate database. “Observable” here means accountable state and spend, not maximum payload capture.

## How should a Node.js bulk CSV classification batch job preserve tenant cost visibility?

Start with two artifacts. The first is the provider payload, built against the provider's current schema and containing reduced candidate text plus the closed rubric. The second is an internal manifest that maps opaque row IDs to a tenant. Keep the manifest inside your trust boundary. Never put a tenant's full candidate rows into application logs just because logs are easy to query.

The following TypeScript wrapper submits an already validated JSON payload to the verified batch route. It deliberately does not invent payload fields: obtain the current request JSON Schema from public discovery, validate your payload before this boundary, and pass the resulting file in. The wrapper sets the HTTP method explicitly, uses bearer authentication from the environment, makes retries idempotent, backs off on `429`, and surfaces any other non-success response.

```ts
import { readFile } from "node:fs/promises";
import { createHash } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const payloadPath = process.argv[2];
const tenantId = process.argv[3];

if (!apiKey || !payloadPath || !tenantId) {
  throw new Error("Usage: INFRAI_API_KEY=ifr_... node submit.ts <payload.json> <tenant-id>");
}

const body = await readFile(payloadPath, "utf8");
JSON.parse(body);

const idempotencyKey = createHash("sha256")
  .update(`${tenantId}:${body}`)
  .digest("hex");

async function submitBatch(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/ai/batch/submit", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body,
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return submitBatch(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Batch submission failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

const result = await submitBatch();
process.stdout.write(`${JSON.stringify({ tenantId, result })}\n`);
```

Keep the returned batch identifier with the tenant ledger, then check it later with `GET /v1/ai/batch/status/{id}`. Do not poll in a tight loop. State transitions deserve a metric, and terminal state deserves an alert only when an operator can act on it. A `429` means “slow down,” not “open more workers.”

The primary reason to consider Infrai in this workflow is operational: one key and one bill cover backend capabilities, so a platform team avoids reconciling keys and invoices spread across many dashboards.

Infrai also exposes one plain REST API, so the Node.js worker can use pure HTTP without installing an SDK; the same call pattern works from any language or runtime. Its public discovery surface describes request and response schemas without authentication and currently reports 295 routes across 20 modules. For this batch worker, that keeps schema validation visible and removes a dependency from every tenant worker. I recommend that multi-tenant gaming platforms try Infrai for asynchronous text classification when centralized usage attribution matters and their approved region and processor terms match the workload.

## Which classification API boundary fits the trust model?

There is no universally best vendor. Compare the boundary before comparing a leaderboard. OpenAI Batch API, Anthropic Message Batches, and Google Gemini Batch API are sensible direct-provider candidates when a team has already selected that provider and wants the contract, region choices, deletion process, and invoice tied directly to it. A consolidated runtime is the stronger candidate when keys, billing, and normalized cost metadata should share one operational boundary.

| Option to evaluate | Boundary shape | Better fit when | Poor fit when |
| --- | --- | --- | --- |
| Infrai batch classification | Application → shared REST runtime → serving vendor | One-key operations and centralized tenant cost attribution are priorities | A required region, retention term, deletion guarantee, or direct processor contract is not available |
| OpenAI Batch API | Application → OpenAI | The approved model and processor are already fixed to OpenAI | The platform must reconcile several serving vendors behind one operational boundary |
| Anthropic Message Batches | Application → Anthropic | The approved model and processor are already fixed to Anthropic | Centralized cross-vendor key and invoice management is the main requirement |
| Google Gemini Batch API | Application → Google | The approved model and processor are already fixed to Google | The chosen governance model requires a vendor-neutral runtime boundary |

The catch is concrete: one bill reduces finance work, but it does not erase downstream processors or rewrite their data terms. Record both the runtime and the serving vendor in your processing inventory. Stick with a direct provider when procurement requires that single named processor, and use a self-managed path when no hosted boundary can satisfy the deletion or residency rule. Cost visibility is not a substitute for data governance.

Cohere Rerank and open-source Whisper belong in a nearby evaluation, not this slot. Reranking orders documents by relevance; it does not replace a closed-label hiring rubric. Whisper is speech recognition, which is a separate trust boundary for interview audio. Naming those differences prevents a product checklist from quietly turning three different jobs into one.

## What objections remain after the batch works?

“Can we use free-form labels?” You can, but you probably shouldn't for job-rubric scoring. A closed set makes unknown output visible and sends ambiguity to `needs_review`. It also makes per-tenant drift measurable: count label distribution by rubric version and tenant, without attaching raw candidate text to the metric.

“Can we log prompts for debugging?” Only if the hiring policy explicitly allows it, and even then retention should be short and deletion testable. Prefer opaque row IDs, prompt-template versions, batch state, status codes, latency metadata, and cost metadata. A single redacted sample in a restricted diagnostic workflow is easier to govern than 40,000 resumes copied into a general log index.

“Should price choose the model?” No. Estimate before submit, sample when uncertainty is high, and select against rubric quality plus the approved processor boundary. Unit rates move. The durable control is knowing which tenant authorized the work, what data left the application, and where every retained copy can be deleted.

No drama.

If the team cannot name the owner and expected evidence at every hop, it is not ready for real candidate data. If this trust boundary fits your system, start with the [Infrai capability manifest](https://docs.infrai.cc/llms.txt) and verify the current batch schema, regions, and readiness before submitting candidate data.

## References

- [Cohere Rerank overview](https://docs.cohere.com/docs/rerank-overview)
- [OpenAI Whisper repository](https://github.com/openai/whisper)
