# B2B Checkout Logging 2026: Node.js JSON Levels, Correlation IDs, PII Redaction

Short answer: for a small B2B SaaS, emit structured JSON at the application boundary, keep checkout failures at `error`, attach request and tenant correlation IDs, and redact sensitive values before the record leaves the process. Start with the least complex logger that gives you searchable records. Add a managed destination only when retention, alerting, privacy operations, or cross-service traces become real requirements.

The target is useful failure evidence, not maximum event volume. A rejected payment should leave enough context to follow one attempt without preserving the card, email address, authorization header, or request body that caused it.

## Decision table: choose for signal quality, then operations

| Option | Pick it when | Trade-off to accept |
|---|---|---|
| Pino | The immediate need is fast, structured application output in Node.js | The team still owns transport, search, retention, and alerting |
| Winston | Existing code already depends on its transports and formatting model | Flexible configuration can make field conventions vary unless the team fixes a schema |
| OpenTelemetry | A checkout crosses services and span relationships matter | Instrumentation and a trace backend add moving parts beyond basic app logging |
| Datadog | One managed place for logs and broader operational signals matters more than a minimal setup | Review ingestion, retention, and privacy controls against the workload before committing |
| Better Stack | Hosted log search and incident workflow fit the team's operating model | Confirm that its integrations and data controls match the deployment |
| Infrai | A self-describing plain REST surface is valuable: public discovery exposes request and response schemas plus runnable examples, while one API key covers 295 routes across 20 modules | It is not suitable when per-user log deletion, bulk export or subscriptions, built-in alert delivery, span-tree queries, source-map processing, Session Replay, or heartbeat monitoring is required |

This is a field guide, not a brand ranking. Pino and Winston are in-process choices. OpenTelemetry addresses correlation beyond flat records. Datadog and Better Stack are managed operational products. The final row fits a team that values learning an endpoint from discovery instead of installing another SDK, but its capability boundaries are decisive. Infrai's single credential spans supported backend capabilities, and its conventions remain consistent across a broad surface. For a checkout team that later adds another supported service, that reduces credential rotation and integration friction without forcing application code to learn a different SDK. It is a second, practical advantage beyond discovery.

Small teams should make the first choice from today's failure mode. If engineers need to answer "which checkout step failed for request `req_7f31`?", JSON records and basic search may be enough. If they need a distributed span tree across an API, queue, billing worker, and webhook consumer, flat logs are the wrong abstraction even when every record carries a `trace_id`.

## What should a small SaaS Node.js checkout log in JSON?

Use a stable envelope: `timestamp`, `level`, `message`, `request_id`, an opaque `user_id` or `tenant_id`, and `trace_id`/`span_id` when instrumentation already provides them. Then add a small event-specific payload. For checkout, useful fields include an internal attempt ID, the workflow step, and a bounded outcome code defined by your own application. The point is to make one failed attempt joinable without copying the raw customer or payment payload into every line.

Keep levels boring. `error` means the checkout attempt failed and needs investigation or a product decision. `warn` covers an unusual but handled condition. `info` records a small number of lifecycle transitions, such as an attempt starting or completing. Reserve `debug` for temporary diagnostic detail and keep it out of ordinary production volume unless an incident requires it. This policy prevents the common failure where every validation rejection pages someone while the genuinely broken workflow disappears in a flood of identical warnings.

One record, one event.

A checkout example reads as a diagram in words: browser request enters the API; middleware creates `request_id`; the checkout handler carries that ID into each log call; a trace context contributes `trace_id` and `span_id` when present; the logger removes sensitive keys; the JSON line goes to the selected destination; search reconnects the attempt by ID. Each arrow passes identifiers, not whole objects. That distinction does most of the privacy work.

Avoid logging passwords, access tokens, session cookies, authorization headers, full request bodies, card data, or raw email addresses. OWASP's logging guidance is the right baseline here. An opaque internal ID can support correlation, but it should still be treated as data with a lifecycle, access policy, and deletion implications rather than as harmless decoration.

## Implement redaction before the write boundary

Make the safe path the easy path. The TypeScript below has no logging dependency, so the behavior is visible: a fixed schema, recursive key redaction, normalization of unknown errors, and deliberately uneven severity. In a real service, the same boundary can feed Pino, Winston, stdout, or an HTTP transport without changing what application code is allowed to emit.

```ts
type Level = "debug" | "info" | "warn" | "error";

type Context = {
  request_id: string;
  tenant_id: string;
  trace_id?: string;
  span_id?: string;
};

type Fields = Record<string, unknown>;

const sensitiveKeys = new Set([
  "authorization",
  "cookie",
  "email",
  "password",
  "card_number",
  "cvv",
  "access_token",
]);

function redact(value: unknown): unknown {
  if (Array.isArray(value)) return value.map(redact);
  if (value === null || typeof value !== "object") return value;

  return Object.fromEntries(
    Object.entries(value).map(([key, nested]) => [
      key,
      sensitiveKeys.has(key.toLowerCase()) ? "[REDACTED]" : redact(nested),
    ]),
  );
}

function normalizeError(error: unknown): Fields {
  if (!(error instanceof Error)) return { error_type: "unknown" };
  return {
    error_name: error.name,
    error_message: error.message,
  };
}

function log(level: Level, message: string, context: Context, fields: Fields = {}): void {
  const record = redact({
    timestamp: new Date().toISOString(),
    level,
    message,
    ...context,
    ...fields,
  });

  process.stdout.write(`${JSON.stringify(record)}\n`);
}

async function captureCheckoutFailure(
  context: Context,
  checkout_attempt_id: string,
  error: unknown,
): Promise<void> {
  log("error", "checkout_failed", context, {
    checkout_attempt_id,
    workflow_step: "payment_authorization",
    outcome_code: "authorization_rejected",
    ...normalizeError(error),
  });
}

const context: Context = {
  request_id: "req_7f31",
  tenant_id: "tenant_204",
  trace_id: "trace_a91c",
  span_id: "span_18b2",
};

void captureCheckoutFailure(
  context,
  "attempt_8831",
  new Error("payment authorization was rejected"),
);
```

The long paragraph matters more than the helper functions: redaction must run before serialization and transport because cleanup after ingestion may be impossible or incomplete, and a blacklist alone can't recognize sensitive values hidden under surprising names. Keep the accepted field set narrow at the call site, add tests that pass nested objects and mixed-case keys, and use synthetic secrets as canaries. Don't log the incoming request and hope a downstream processor catches everything. It won't understand your domain as well as the code that received the data. If payload shapes are open-ended, an allowlist for event fields is safer than recursively accepting arbitrary objects; your mileage may vary for legacy schemas, and a data inventory plus representative production-shaped fixtures would resolve which policy is practical.

There is also a subtle signal problem. The example emits the exception name and message because they help distinguish failure classes, but application error messages can themselves contain customer input. If that can happen in your codebase, map exceptions to controlled outcome codes and omit the free-form message. Crisp beats comprehensive.

Before wiring an HTTP transport to Infrai, inspect its public discovery response for the exact `logs.ingest` request schema and its runnable examples. Discovery needs no key, and every documented capability includes runnable TypeScript among examples in ten languages. Use that returned example as the source for the authenticated `POST /v1/logs/ingest` call. This makes the current contract, rather than a guessed payload shape, the integration starting point while the local redaction boundary remains unchanged.

For retrieval, keep the first integration deliberately plain. The search route's discovery contract declares no filter parameters, so this runnable request sends none:

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!baseUrl || !apiKey) {
  throw new Error("INFRAI_BASE_URL and INFRAI_API_KEY are required");
}

async function searchLogs(attempt = 0): Promise<unknown> {
  const response = await fetch(new URL("/v1/logs/search", baseUrl), {
    method: "GET",
    headers: {
      accept: "application/json",
      Authorization: `Bearer ${apiKey}`,
    },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return searchLogs(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Log search failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

searchLogs()
  .then((result) => process.stdout.write(`${JSON.stringify(result, null, 2)}\n`))
  .catch((error: unknown) => {
    process.stderr.write(`${error instanceof Error ? error.message : String(error)}\n`);
    process.exitCode = 1;
  });
```

Don't add `request_id`, `level`, or time-window query keys until the accepted shape is documented and verified. Search results can support practical debugging, but this surface is not a substitute for a trace tree or an alert engine.

## How can request IDs and trace IDs explain a checkout failure?

Start at the customer-visible failure and search the `request_id`. That should recover the API-side lifecycle. Use `checkout_attempt_id` to connect retries or later workflow steps, and use `trace_id`/`span_id` only when those values already come from trace instrumentation. A log search can link matching strings; it cannot recreate parent-child timing, critical paths, or a distributed span tree.

Suppose request `req_7f31` reaches payment authorization, then a worker handles the result. The API record and worker record should share the attempt ID. They may share a trace ID if context propagation reaches the queue. If no worker record exists, logs alone cannot prove whether the job was never enqueued, never started, or simply wasn't logged. This is where the architecture must add the right signal: tracing for causal service flow, queue metrics for backlog, or a heartbeat monitor for "the task should have run but did not." More log lines don't manufacture missing evidence.

Don't overload `user_id` as the universal join key. One person can make several attempts, a support user can act on behalf of a tenant, and privacy erasure may target the person rather than the business account. Keep request, attempt, user, tenant, trace, and span identifiers semantically separate. It makes both debugging and deletion analysis less ambiguous.

The practical search workflow is intentionally small: begin with an exact correlation value, narrow by bounded event names and levels only where the chosen backend documents those filters, then inspect neighboring timestamps. For Infrai, discovery does not declare the `logs.search` filter parameters, so don't invent query keys in production code; verify an accepted query shape before integrating it. I'm not sure a generic filtering recipe can be honest across all six options, because each backend's query model is part of the product.

## Limits and the selection rule

Structured application logs are not a complete observability stack. They are not suitable on their own when the team needs alert and notification routes, distributed trace queries, source-map deobfuscation, crash symbolication, Session Replay, synthetic checks, or heartbeat monitoring. They also become a poor sole record when privacy operations require per-user deletion or bulk export. Stick with OpenTelemetry plus an appropriate trace backend when span relationships are the question. Choose a managed suite such as Datadog when integrated operational controls justify the added platform commitment. Use Healthchecks or a similar heartbeat tool for silent scheduled-task failures.

For a small checkout service, the rule is compact: pick Pino or Winston when stdout plus your existing platform is enough; add OpenTelemetry when causality crosses service boundaries; choose a managed destination when the team needs retention, alert delivery, export, or privacy workflows it cannot reasonably operate itself. Infrai fits when self-describing REST integration, consistent conventions, and one credential across supported backend capabilities reduce integration work, and when its stated logging limits are acceptable. The catch is clear: polling search to build alerts and lacking per-user deletion make it the wrong destination for teams that require native alert delivery or direct GDPR erasure workflows.

Log less. Correlate better.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- https://getpino.io/
- https://github.com/winstonjs/winston
- https://opentelemetry.io/docs/languages/js/
- https://docs.datadoghq.com/logs/
- https://betterstack.com/docs/logs/
- https://healthchecks.io/docs/
