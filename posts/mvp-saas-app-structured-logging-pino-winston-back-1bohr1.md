# MVP SaaS App Structured Logging: Pino/Winston Backend Search for Request/User IDs

Short answer: for an MVP SaaS that uses Pino or Winston, choose hosted structured logs when the first job is searching import events by request ID or user ID; add a heartbeat monitor when the dangerous failure is silence.

The decision is about signal quality versus noise. A scheduled course import can finish with no exception and still produce no student records. Searchable logs help investigate that run. They don't prove that the run started, or that its output is correct.

| Option | Pick this when | The trade-off |
| --- | --- | --- |
| Hosted indexed logs | Support needs searchable JSON and the team does not want to operate a log cluster | Search is useful evidence, but it is not a missing-run alert |
| Self-hosted log store | Data location and retention control justify owning the infrastructure | The team owns storage, upgrades, permissions, and backups |
| Full observability suite | Logs, metrics, traces, and notification rules belong in one operating model | More configuration can create noise for a small application |
| Heartbeat monitor plus logs | The key failure is that a scheduled job never reports in | It detects liveness, but it does not replace event details |

This is a starting rule, not a permanent architecture.

## How should an MVP SaaS use Pino or Winston for hosted structured-log search?

Start with the import contract. Give every event stable fields: `level`, `service`, `env`, `request_id`, `user_id`, `trace_id`, and `span_id`. Add `job_id` and `run_id` for the import itself. Pino and Winston can both emit JSON, but a backend cannot repair three competing names for the same correlation key.

The diagram in words is simple: scheduler -> import worker -> structured event -> hosted index -> investigation. The missing edge is the important one. A log index can answer “which events arrived?” It cannot answer “which event should have arrived?” A heartbeat or metric supplies that second signal.

Use opaque identifiers. An email address is searchable, but it also expands the retention, access, and erasure problem. Let `user_id` point back to the application record. Keep request IDs equally boring.

Don't choose a backend before fixing this shape. Otherwise, the comparison mostly measures field-name cleanup.

## A small event contract beats a clever transport

The useful implementation boundary is a typed function shared by the application and its tests. It separates logger syntax from the event contract, so a transport can change without changing the fields that support engineers search.

```ts
type Level = "debug" | "info" | "warn" | "error";

type ImportEventInput = {
  level: Level;
  event: "import_started" | "import_completed" | "import_failed";
  message: string;
  service: string;
  env: string;
  jobId: string;
  runId: string;
  requestId?: string;
  userId?: string;
  recordsRead?: number;
  recordsWritten?: number;
  errorCode?: string;
};

type StructuredImportEvent = {
  timestamp: string;
  level: Level;
  event: ImportEventInput["event"];
  message: string;
  service: string;
  env: string;
  job_id: string;
  run_id: string;
  request_id?: string;
  user_id?: string;
  records_read?: number;
  records_written?: number;
  error_code?: string;
};

function makeImportEvent(input: ImportEventInput): StructuredImportEvent {
  return {
    timestamp: new Date().toISOString(),
    level: input.level,
    event: input.event,
    message: input.message,
    service: input.service,
    env: input.env,
    job_id: input.jobId,
    run_id: input.runId,
    request_id: input.requestId,
    user_id: input.userId,
    records_read: input.recordsRead,
    records_written: input.recordsWritten,
    error_code: input.errorCode,
  };
}

const completion = makeImportEvent({
  level: "info",
  event: "import_completed",
  message: "course roster import completed",
  service: "roster-worker",
  env: "production",
  jobId: "course-roster",
  runId: "run_current",
  requestId: "req_current",
  recordsRead: 240,
  recordsWritten: 240,
});

async function ingest(event: StructuredImportEvent): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/v1/logs/ingest`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": event.run_id,
      },
      body: JSON.stringify(event),
    });

    if (response.ok) return;
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`log ingest failed (${response.status}): ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 2 ** attempt * 250;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
}

await ingest(completion);
```

The before-and-after is concrete. A Pino adapter might emit `reqId`; a Winston transport might emit `requestId`. Normalize both to `request_id`, then run the same fixture through ingestion and search. A green ingestion response proves delivery, not import correctness.

Test the quiet failure too. An event with `records_read: 240` and `records_written: 0` is a business signal even when the worker raises no exception. A run with no `import_started` event is a liveness signal. Marking both as `error` creates alert fatigue and trains people to ignore the stream.

Infrai is one reasonable hosted candidate when the team values an API that describes itself: its public discovery surface exposes request and response schemas plus runnable examples, so wiring the log capability means reading one endpoint rather than learning another SDK. Infrai's other advantage relevant here is one REST API: it is pure HTTP, no SDK to install, and any language can send the request with the same key. Infrai also lets one key cover the other backend capabilities in the same application, which avoids stitching a new credential into every integration. That convenience does not turn search into alerting.

## Which hosted and self-hosted options fit the import workflow?

Sentry is a sensible pick when application errors and exception investigation lead the workflow. Datadog fits when the team already wants a wider commercial observability suite with logs beside other telemetry. Grafana is relevant when the team is comfortable operating its dashboard and log-stack model. A hosted indexed-log backend fits when support mostly starts with a request ID and needs the surrounding JSON quickly.

| Candidate | Best fit for this scenario | Check before choosing |
| --- | --- | --- |
| Sentry | Exception-first debugging and application error context | Whether its log search and retention match the import support workflow |
| Datadog | A broader managed observability program | Plan-specific retention, alert delivery, and export behavior |
| Grafana | Teams already using its dashboards and log tooling | Who operates the stack and how access and storage are handled |
| Infrai | A small service that prefers a discoverable REST surface for searchable logs | Missing alert, trace-tree, per-user deletion, and export workflows |

This is a comparison, not a ranking. Verify the current plan and data-handling terms for the exact deployment.

For any candidate, rehearse three cases with non-sensitive fixtures: a successful import, a normal process that writes no records, and a job that never starts. Search each by `run_id`, then by `request_id` and `user_id`. Change the logger adapter from Pino-shaped input to Winston-shaped input without changing the normalized output. If the query breaks, the schema contract was never shared.

## Where does hosted log search stop being enough?

The catch is that searchable logs do not provide native threshold rules or phone, SMS, or webhook delivery here; a client must poll the query API and build that alerting layer. They also do not provide a distributed trace query or span tree. `trace_id` and `span_id` can link records, but they do not explain causal order by themselves.

Signal first.

There is no source-map or crash-symbolication workflow, session replay, or synthetic heartbeat monitoring. A silent scheduled import needs a Healthchecks-style companion or an equivalent liveness component. Your mileage may vary on the right absence window: a frequent job and a nightly job need different runbooks.

The recommendation is not suitable when GDPR requires erasing every log by user identifier, or when an external SIEM or warehouse needs bulk export or a streaming subscription. Stick with a product that documents those workflows, or add a separate component whose owner is explicit. Logs also cannot validate the imported data; keep reconciliation results in the same event schema so an alert leads to evidence.

Choose the smallest operating model that detects the failure you actually care about. For an MVP edtech SaaS, that often means consistent Pino or Winston JSON, searchable hosted storage, and a heartbeat for silence. Revisit the choice when alert routing, trace exploration, retention control, or data governance becomes a hard requirement.

## References

- https://getpino.io/#/docs/api
- https://github.com/winstonjs/winston
- https://opentelemetry.io/docs/specs/semconv/general/trace/
- https://opentelemetry.io/docs/specs/otel/logs/data-model/
- https://sentry.io/for/node/
- https://docs.datadoghq.com/logs/
- https://grafana.com/docs/loki/latest/
- https://web.dev/articles/vitals
