# Two-Gate Image Generation API Prompt Safety (Quality Before Latency)

Short answer: choose an image generation API only after a typed chat-model safety gate passes your own invoice-fixture evaluation, then optimize latency within the set of APIs that meet the quality bar. A native moderation endpoint is useful, but its absence is not a reason to send unchecked supplier text into image generation.

For a developer tool that extracts fields from supplier invoices, generated images can expand test coverage: wrinkled paper, faint totals, crowded line items, or unusual layouts. The dangerous input is often less obvious than an openly abusive prompt. A supplier name, memo, or line-item description can contain text that your fixture builder should never reproduce. Treat that text as untrusted data.

The mental model is small. Before: supplier-derived text goes straight to pixels, and one latency number hides every decision. After: normalize input, ask a chat model for a schema-constrained decision, generate only on `allow`, and record separate timing and quality signals for both gates.

Fast is secondary.

## The contamination failure comes before API selection

For invoice fixtures, the first failure is allowing real document content into a synthetic-data pipeline. Replace account numbers, tax identifiers, addresses, names, and free-form notes with synthetic tokens before any model call. The image prompt can ask for an invented invoice layout with synthetic fields. The resulting image then tests extraction behavior without asking a generator to reproduce a real document.

Only then classify the sanitized prompt. The chat model should return a narrow decision object, not rewritten prose and not an image prompt. Keep policy in versioned application data. Pass the model the candidate prompt plus the policy version, require one of a few outcomes, and reject anything that does not validate. This creates a clean control point even when the chosen image API has no separate moderation endpoint.

A JSON Schema contract does two jobs. It constrains the response shape, and it gives the caller an explicit failure branch. It does not prove that the classification is correct. That distinction matters. Schema validation catches malformed output; an evaluation set measures policy quality. You need both.

Use three outcomes: `allow`, `deny`, and `review`. `review` is not a polite synonym for allow. It stops image generation and sends the case to whatever review path your organization has defined. Keep the reason codes local and stable so dashboards do not depend on a model's wording.

## A copyable two-gate TypeScript example

The example below leaves transport behind two interfaces. That is deliberate: endpoint paths and response envelopes vary, while the safety invariant should survive an API swap. The chat client must support schema-constrained output through its own verified integration. The image client receives a prompt only after the returned object passes local validation.

```ts
const SAFETY_SCHEMA = {
  type: "object",
  additionalProperties: false,
  required: ["decision", "reasonCode", "policyVersion"],
  properties: {
    decision: { enum: ["allow", "deny", "review"] },
    reasonCode: { type: "string", minLength: 1, maxLength: 64 },
    policyVersion: { const: "invoice-fixtures-v3" }
  }
} as const;

type SafetyDecision = {
  decision: "allow" | "deny" | "review";
  reasonCode: string;
  policyVersion: "invoice-fixtures-v3";
};

type ChatGate = {
  classify(input: { prompt: string; schema: typeof SAFETY_SCHEMA }): Promise<unknown>;
};

type ImageGenerator = {
  generate(input: { prompt: string; requestId: string }): Promise<{ imageId: string }>;
};

function isSafetyDecision(value: unknown): value is SafetyDecision {
  if (typeof value !== "object" || value === null) return false;
  const item = value as Record<string, unknown>;
  return (
    (item.decision === "allow" ||
      item.decision === "deny" ||
      item.decision === "review") &&
    typeof item.reasonCode === "string" &&
    item.reasonCode.length >= 1 &&
    item.reasonCode.length <= 64 &&
    item.policyVersion === "invoice-fixtures-v3" &&
    Object.keys(item).every((key) =>
      ["decision", "reasonCode", "policyVersion"].includes(key)
    )
  );
}

export async function generateInvoiceFixture(
  rawFields: Record<string, string>,
  requestId: string,
  chat: ChatGate,
  images: ImageGenerator
): Promise<{ status: "generated"; imageId: string } | { status: "blocked"; reason: string }> {
  const syntheticFields = {
    supplier: "SAMPLE SUPPLIER",
    invoiceNumber: "INV-000042",
    currency: rawFields.currency === "EUR" ? "EUR" : "USD",
    lineItem: "SAMPLE SERVICE"
  };

  const prompt = [
    "Create a clearly synthetic supplier invoice test fixture.",
    "Do not include logos, signatures, bank details, or real people.",
    JSON.stringify(syntheticFields)
  ].join(" ");

  const candidate = await chat.classify({ prompt, schema: SAFETY_SCHEMA });
  if (!isSafetyDecision(candidate)) {
    return { status: "blocked", reason: "invalid_gate_response" };
  }
  if (candidate.decision !== "allow") {
    return { status: "blocked", reason: candidate.reasonCode };
  }

  const result = await images.generate({ prompt, requestId });
  return { status: "generated", imageId: result.imageId };
}
```

Notice what is absent. `rawFields` never becomes prompt text. In a production implementation, the currency allowlist and synthetic substitutions belong in a tested normalizer, and the local validator should be generated from or checked against the same schema sent to the chat model. Don't let those definitions drift.

The request ID joins the gate span to the generation span. It should not encode prompt contents. Log the policy version, decision, reason code, schema-validation result, and durations; keep raw prompts and generated invoice images out of ordinary logs unless your retention and access rules explicitly permit them.

## Observe two gates, not one request

A single success-rate chart will lie by omission. Build an evaluation set with allowed synthetic invoice prompts, prompts that must be denied, and genuinely ambiguous cases that belong in review. Label it under the policy you actually intend to enforce. I'm not sure any universal weighting would fit every invoice product: a tool that generates fixtures only for an internal test environment has a different consequence profile from a public prompt box. Resolve that uncertainty with named owners and a versioned set, not intuition.

Track the safety gate's false-allow and false-deny counts on that set. For the image stage, score whether required fields are present, whether forbidden real-world identifiers appear, and whether the downstream extractor returns the intended values. A visually convincing invoice that breaks field extraction is a failed fixture. So is a plain image that extracts perfectly but contains prohibited content.

Then measure latency by stage. Record gate duration, generation duration, end-to-end duration, and queue time as separate histograms. Compare percentiles at the same concurrency and prompt mix. This diagram in words is the whole trace: request accepted -> fields replaced -> safety decision validated -> image generated -> extraction checked -> fixture admitted. Put an alert on unexpected shifts in deny rate and invalid gate responses; use a different alert for generation latency. One page should not wake an engineer for two unrelated failure modes.

The selection scorecard can stay compact:

| Dimension | Release evidence | Reject when |
| --- | --- | --- |
| Prompt safety | Versioned labeled evaluation set | A false-allow budget is exceeded |
| Contract handling | Invalid and extra-field responses are blocked | Malformed output can reach generation |
| Fixture quality | Generated fields match extraction assertions | Required values are unreadable or changed |
| Latency | Per-stage percentiles under a fixed load profile | The agreed end-to-end budget is missed |
| Operations | Correlated traces without sensitive prompt text | Decisions cannot be audited by policy version |

Run this scorecard against the APIs you are considering. Don't crown a winner from documentation alone. The best choice is the one that passes your policy and extraction tests at an acceptable latency under your workload.

## Can prompt safety protect an image generation API without a moderation endpoint?

First, verify that the safety call is actually the limiting stage. Image generation may dominate, queueing may dominate, or a serial preprocessing step may be the real culprit. Stage-level traces answer this quickly. Guessing does not.

If the gate is material, reduce its work without weakening the boundary: keep the policy prompt short, remove real invoice data before classification, cap input size, and apply bounded concurrency. Cache only when the normalized prompt and policy version are identical, and make cache invalidation part of a policy release. Retries need care. HTTP semantics distinguish idempotent methods because repeating a request can have different effects; your client should follow the documented semantics of each provider rather than assuming every generation request is safe to replay.

The catch is that a chat-model gate is not suitable when policy requires a certified specialist control, deterministic rules alone can decide the input, or the extra processing step cannot fit the latency budget. In those cases, use the required specialist service, a local deterministic filter for a genuinely closed vocabulary, or an image API with an independently verified native safety control. Keep defense after generation too: validate the artifact before it enters the fixture corpus.

No workaround changes that trade-off.

## References

- RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- JSON Schema specification: https://json-schema.org/specification
