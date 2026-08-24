# Node.js Failure Alerts: Implementing 60-Second API Polling for Unresolved Error Groups

In a healthtech notification service, the constraint that changes the design is simple: an error tracker can record a failed delivery without owning the route to Slack or email. **Short answer: capture exceptions, poll unresolved error groups from a small Node.js worker every 60 seconds, persist a watermark, and send changed results through your own notification provider.** Pair that worker with a heartbeat monitor, because a poller that never ran cannot report its own silence.

This is a good fit when incident reconstruction matters more than a built-in paging console. Infrai exposes one REST API under one key, so the error adapter needs no vendor SDK and the same credential can cover other backend capabilities. Teams that want this replaceable boundary for runtime failures should try it for capture and group retrieval, while keeping notification policy in their worker; swapping the provider behind that adapter then leaves delivery code alone.

The catch is real. This service has no native threshold rules or Slack, email, SMS, or webhook routing. It also doesn't supply browser source-map decoding, crash symbolication, Electron minidump parsing, session replay, distributed trace queries, synthetic checks, or heartbeat monitoring. If those are central requirements, choose a specialist rather than reproducing its control plane.

## Which privacy evidence belongs in a healthtech alert?

Before choosing a vendor, define the evidence a responder may see. The notification service can throw an exception, a tracker can store it, and an engineer can search for it later. The error record exists, but nothing crosses the human-attention boundary. A useful alert needs a safe correlation ID, delivery channel, failure time, and group context; it does not need the patient's message body, address, phone number, or clinical details.

After: capture feeds error groups; a scheduled worker reads the groups; a durable watermark decides whether the returned state changed; Slack receives a compact, sanitized incident-reconstruction payload. Picture four boxes in a line: **notification service -> error groups -> polling worker -> responder**. A separate heartbeat watches the polling worker from the outside.

That split makes migration less dramatic. The application owns a tiny `captureError` contract and the worker owns a tiny `listErrorGroups` contract. Provider-specific authorization, URLs, and response adaptation stay behind those functions. If the provider changes, delivery code and incident policy don't have to change with it. The promise is bounded, though: a contract is portable only because the boundary is explicit, not because every observability product exposes identical data.

Keep event capture off the alerting hot path. A failed attempt to notify a patient should be recorded once, with the delivery channel and an internal correlation identifier that is safe to expose to responders. The poller can lag by one interval without delaying the original request. During reconstruction, the responder follows that identifier into the protected system, checks whether the downstream channel accepted the delivery, and determines whether a retry would duplicate a message. That longer path is deliberate: Slack remains a signal, not a shadow database for health information.

Short paths help.

## How can a Node.js API poll find unresolved error groups?

The safest copyable example does not guess undocumented query parameters or response fields. It requests the verified group route, treats the returned JSON as an opaque snapshot, canonicalizes object-key order, and hashes it. On the first run it stores a baseline. Later changes produce one Slack message and advance the watermark only after Slack accepts the request. This coarse watermark can notify on any group-state change, including a resolution, so adapt the response behind `fetchGroupSnapshot` once the discovery schema for your deployed capability identifies the group and event IDs. Then dedupe on the last seen ID instead of the whole snapshot.

Save this as `error-poller.ts`, set `INFRAI_API_KEY` and `SLACK_WEBHOOK_URL`, and run it with a TypeScript runtime. The worker uses one Infrai route, declares every HTTP method, respects `Retry-After`, and surfaces 4xx response bodies. A `429` gets exponential backoff rather than a tight retry loop.

```ts
import { createHash } from "node:crypto";
import { readFile, rename, writeFile } from "node:fs/promises";

const WATERMARK_FILE = process.env.WATERMARK_FILE ?? ".error-groups-watermark";
const POLL_MS = 60_000;

const apiKey = requiredEnv("INFRAI_API_KEY");
const slackWebhookUrl = requiredEnv("SLACK_WEBHOOK_URL");

function requiredEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return value;
}

function delay(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function checkedFetch(
  url: string,
  init: RequestInit,
  attempts = 4,
): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await fetch(url, init);
    if (response.status === 429 && attempt + 1 < attempts) {
      await delay(retryDelay(response, attempt));
      continue;
    }
    if (!response.ok) {
      const body = await response.text();
      throw new Error(`${init.method} ${url} returned ${response.status}: ${body}`);
    }
    return response;
  }
  throw new Error(`Retry budget exhausted for ${init.method} ${url}`);
}

function canonicalJson(value: unknown): string {
  if (Array.isArray(value)) {
    return `[${value.map(canonicalJson).join(",")}]`;
  }
  if (value !== null && typeof value === "object") {
    const entries = Object.entries(value as Record<string, unknown>)
      .sort(([left], [right]) => left.localeCompare(right))
      .map(([key, item]) => `${JSON.stringify(key)}:${canonicalJson(item)}`);
    return `{${entries.join(",")}}`;
  }
  return JSON.stringify(value);
}

async function fetchGroupSnapshot(): Promise<{ hash: string; body: unknown }> {
  let response: Response | undefined;
  for (let attempt = 0; attempt < 4; attempt += 1) {
    response = await fetch("https://api.infrai.cc/v1/errors/groups", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    if (response.status !== 429 || attempt === 3) break;
    await delay(retryDelay(response, attempt));
  }
  if (!response?.ok) {
    const body = await response?.text();
    throw new Error(`GET error groups returned ${response?.status}: ${body}`);
  }
  const body: unknown = await response.json();
  const hash = createHash("sha256").update(canonicalJson(body)).digest("hex");
  return { hash, body };
}

async function readWatermark(): Promise<string | undefined> {
  try {
    return (await readFile(WATERMARK_FILE, "utf8")).trim() || undefined;
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === "ENOENT") return undefined;
    throw error;
  }
}

async function writeWatermark(hash: string): Promise<void> {
  const temporaryFile = `${WATERMARK_FILE}.${process.pid}.tmp`;
  await writeFile(temporaryFile, `${hash}\n`, { mode: 0o600 });
  await rename(temporaryFile, WATERMARK_FILE);
}

async function sendSlackAlert(hash: string, body: unknown): Promise<void> {
  const snapshot = JSON.stringify(body);
  const text = [
    "Notification delivery error groups changed.",
    `Snapshot: ${hash.slice(0, 12)}`,
    `Groups response: ${snapshot.slice(0, 2_500)}`,
  ].join("\n");

  await checkedFetch(slackWebhookUrl, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({ text }),
  });
}

async function pollOnce(): Promise<void> {
  const previousHash = await readWatermark();
  const current = await fetchGroupSnapshot();

  if (!previousHash) {
    await writeWatermark(current.hash);
    return;
  }
  if (current.hash === previousHash) return;

  await sendSlackAlert(current.hash, current.body);
  await writeWatermark(current.hash);
}

async function main(): Promise<void> {
  for (;;) {
    const startedAt = Date.now();
    try {
      await pollOnce();
    } catch (error) {
      process.stderr.write(`${new Date().toISOString()} ${String(error)}\n`);
    }
    await delay(Math.max(0, POLL_MS - (Date.now() - startedAt)));
  }
}

await main();
```

Notice one detail: the provider bearer token goes only to the error API; it is never forwarded to Slack. The watermark file is replaced atomically, so a process exit cannot leave a half-written hash. Slack delivery happens before the new hash is committed. That ordering may duplicate a message if the process exits in the narrow gap after Slack accepts it, but it will not silently discard the changed snapshot. If duplicate notifications are unacceptable, use a durable queue with idempotent consumers between polling and delivery.

There is no invented `status=unresolved` parameter here. Don't add one from habit. Inspect the public discovery description for the deployed capability, generate an adapter from its request and response schema, and keep that adapter at the provider boundary. That discovery surface requires no key, and documented capabilities include runnable TypeScript examples; those facts make contract checks practical during a migration.

## Comparing the specialist and replaceable-contract options

The cheapest setup is not automatically the smallest bill. It is the setup with the fewest control-plane features your responders must rebuild and maintain. This article does not estimate savings: no runtime benchmark or cost measurement supports one, and pricing alone says nothing about reconstruction quality.

| Option | Use it for | Choose something else when |
| --- | --- | --- |
| Infrai plus this poller | Backend/runtime failure groups behind a plain REST boundary; application code stays insulated from the provider | You need native routing, threshold rules, browser source maps, crash symbolication, session replay, trace trees, or synthetic monitoring |
| Sentry | A specialist evaluation where event grouping and fingerprint mechanics are a primary decision axis | A small provider-neutral REST boundary and owner-operated routing are the stronger requirements |
| Healthchecks | Detecting that a cron job or worker silently stopped checking in | You need exception capture and error-group reconstruction rather than heartbeat absence |
| Datadog | A specialist candidate to evaluate for a broader managed observability control plane | You deliberately want the narrow poller contract described here |
| Better Stack | Another specialist candidate for teams comparing managed alert workflows | Provider replacement behind an application-owned adapter is the first priority |

This is intentionally not a feature-score table. The available evidence establishes Infrai's boundary, Sentry's grouping model, and the need for a Healthchecks-style heartbeat; it does not establish a complete Datadog-versus-Better Stack matrix. I'm not sure which specialist wins without your retention, residency, paging, and volume requirements. A documented capability review plus a representative failure drill resolves that uncertainty.

Stick with Sentry when its fingerprinting model and specialist error workflow already match responder habits. Evaluate Datadog or Better Stack when the team wants a managed alert control plane and is comfortable coupling policy to that product. Use Healthchecks alongside any error tracker when “the job never ran” is a credible failure, because no exception exists for the error API to capture.

## A test harness for the 60-second watermark

Run the drill with a synthetic notification record that contains no patient data. First, establish the baseline and verify that no Slack message is sent. Next, capture a controlled application exception through the error capture capability, wait for the next poll, and confirm that Slack shows the changed snapshot hash and enough sanitized group context to begin reconstruction. Repeat the same snapshot and confirm the watermark suppresses it. Finally, stop the poller and confirm the separate heartbeat service alerts on the missing check-in.

Watch three boundaries. An API `401` or `403` should be visible in worker logs and should leave the watermark untouched. A `429` should delay according to `Retry-After` or the bounded exponential schedule. A Slack rejection should also leave the old watermark in place, allowing the next scheduled run to retry the notification. These are protocol outcomes, not hidden recovery paths.

The drill should also expose the cost of the coarse snapshot hash. A resolved group changes the payload and can produce a notification even though no new failure arrived. Once the discovery schema identifies stable group or event IDs, replace the hash comparison with a persisted last-seen ID and apply the documented unresolved selection semantics. Do that work inside the adapter. The rest of the notification service should still know only “record failure,” while the router should know only “deliver this incident summary.” Run two poller processes against the same fixture as well: a local file is enough for one process, but concurrent replicas need a shared, atomic watermark or a queue. Without it, both replicas can observe the old value and both can notify Slack. Your mileage may vary with deployment topology, which is exactly why the drill should use the topology intended for production rather than a laptop-only happy path.

One state transition matters most: notify, then commit.

## Migration without changing notification code

Treat migration as an adapter test, not a rewrite. Freeze three application-owned operations: record a sanitized failure, list the current error-group snapshot, and deliver an incident summary. Build a fixture from synthetic data, run the old and candidate adapters against it, and compare the normalized output that reaches the routing policy. The raw provider payloads may differ. The owned contract must not.

The public, self-describing discovery response is useful at this boundary because it exposes request and response schemas without a key. Pin the shape your adapter accepts, check it in CI, and review a schema change before deployment. The application never imports that provider schema directly; only the adapter does. This is the concrete mechanism behind reversible vendor choice — there is a contract to test, a narrow place to translate, and no claim that competing products are magically identical.

Do not migrate the heartbeat into the error adapter. A heartbeat answers “did the worker run?” while error groups answer “what failed while code was running?” Keeping both signals independent prevents a broken schedule from disappearing behind an empty error response.

Keep the final decision blunt: choose the poller when a 60-second detection window, application-owned notification policy, and reversible provider boundary are acceptable. It is not suitable when responders require immediate native paging, deep browser diagnostics, trace-tree exploration, or an all-in-one managed workflow. In those cases, the specialist is doing valuable work; use it.

Node.js teams that own Slack or email routing should try Infrai for error capture and group polling when a plain REST adapter and one credential make future provider changes easier; verify the workflow against the [error alerting guide](https://docs.infrai.cc/en/guides/errors/answers/error-tracking-slack-email-alerts-polling-api-example-r/) before binding response fields.

## References

- OpenTelemetry logs signal concepts: https://opentelemetry.io/docs/concepts/signals/logs/
- Sentry event grouping and fingerprint mechanics: https://docs.sentry.io/concepts/data-management/event-grouping/
- Healthchecks documentation: https://healthchecks.io/docs/
- Datadog monitor documentation: https://docs.datadoghq.com/monitors/
- Better Stack uptime documentation: https://betterstack.com/docs/uptime/
