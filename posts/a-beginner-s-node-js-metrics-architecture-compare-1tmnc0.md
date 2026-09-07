# A Beginner's Node.js Metrics Architecture: Compare API Push and Pull for SaaS

Short answer: in a small SaaS custom metrics dashboard, comparing a Prometheus alternative means comparing failure ownership: API push makes the Node.js producer own delivery timing and buffering, while the pull model makes a reachable collector own collection timing and target checks.

| Decision signal | Push to an API | Pull from an endpoint | Collector between app and backend |
| --- | --- | --- | --- |
| Process lifetime | Fits short-lived or externally unreachable work | Fits stable, reachable services | Fits a mixed fleet |
| Who schedules collection? | The application or local exporter | The scraper | The collector, exporter, and backend share the work |
| Failure evidence | Requires an explicit delivery metric | A failed scrape can expose target reachability | Requires health checks for the collector path |
| Application responsibility | Batch, timeout, retry policy, and bounded buffering | Maintain instruments and serve a snapshot | Export through a standard telemetry path |
| Best first test | Stop the receiver and observe bounded behavior | Block the endpoint and observe target health | Stop the collector and inspect queue behavior |

This is a transport decision, not a dashboard decision. Grafana sits at the query and presentation end of the pipeline. It doesn't require a Node.js process to push or to be scraped; it requires a compatible data source containing useful time series. Keep those layers separate and the first choice stays reversible.

## How should a small SaaS compare push API and pull collection?

Start with a diagram in words: **instrument -> aggregation -> transport -> storage -> query -> dashboard**. A counter increment belongs at the first step. Push versus pull changes the transport step and some failure handling around it. It should not change what `checkout_attempts` means, which labels it carries, or which question a panel answers. In a pull design, the application retains metric state and exposes a snapshot. A scraper owns the collection interval and requests that snapshot. This makes the collection clock centrally configurable and gives the monitoring system a direct view of target reachability. The trade-off is topology: the scraper needs an addressable target, discovery information, and permission to connect. Autoscaled processes can still be scraped, but discovery has to follow them accurately. In a push design, an exporter sends batches to an ingest API. That works naturally across outbound-only networks and for processes that may disappear before the next scrape. The application side now has more policy to define: how large a batch may become, how long an export may take, whether a failed batch is dropped or buffered, and how shutdown flushes behave. Don't hide those decisions inside a helper named `sendMetrics`. Make them visible configuration. OpenTelemetry provides a useful boundary here. Its metrics model separates instruments such as counters and histograms from readers and exporters, and it describes both delta and cumulative temporality. That separation is the durable part. A beginner-friendly design can adopt the same shape even before a team standardizes its full telemetry pipeline. The model also changes how absence is interpreted. Under pull, a target that cannot be reached is observable to the scraper, while an empty but successful response means something different. Under push, silence is ambiguous unless each producer emits freshness data or the receiver tracks the last accepted batch. Make a dashboard panel or alert distinguish “zero work” from “no recent observation.” Otherwise the calmest graph can represent the least observable failure.

Start there.

## Pick API delivery when producers are brief or unreachable

API delivery is a reasonable small-system choice when a task runs briefly, lives behind a network boundary that permits outbound traffic only, or cannot expose a stable endpoint. Keep the batch bounded in memory. Give each export a timeout. Record export attempts, accepted batches, rejected batches, and the age of the oldest buffered observation through a separate operational path where practical.

The hard part isn't the POST request.

It is preserving meaning across retries. A cumulative counter reports its value since a defined start point; a delta reports change over an interval. Those are different contracts, and the receiver must know which one it got. A blind retry of a delta can count the same interval twice unless the protocol supplies identity or deduplication semantics. A cumulative stream has its own requirements, including recognizing resets when a process restarts. Use the temporality supported by the chosen metrics pipeline and test restart behavior rather than assuming either form is automatically safe.

Push also moves load control toward the producers. Consider an illustrative 15-second export schedule across 20 instances: without jitter, all 20 timers can become ready together after a coordinated deployment, even though nothing about the metric requires synchronized arrival. Add jitter. Bound concurrent requests. Decide what happens when the buffer is full; blocking product traffic indefinitely is rarely an acceptable telemetry policy. This is where a collector can help: applications export locally or nearby, while the collector handles batching and forwarding. The catch is operational ownership. A collector needs deployment, capacity, upgrades, and monitoring of its own.

## Pick scraping when targets are stable and discoverable

Scraping is attractive when services are long-running, reachable from a collector, and already represented in reliable service discovery. The application maintains current metric state and serves it on request. Collection cadence stays outside the business process, so changing an interval doesn't require changing the application timer.

There is still application work. Snapshot generation must not stall request handling, metric names and label sets need stable contracts, and the endpoint needs an intentional network boundary. A public metrics endpoint can disclose operational details. Put it on an internal listener or protect it through the same network controls used for other administrative endpoints.

Pull is not automatically simpler for every small SaaS. A single Node.js process on a stable host can be easy to scrape; a set of brief jobs with no inbound route cannot. Likewise, push is not automatically easier merely because `fetch` is familiar. Once buffering, retry policy, freshness, and shutdown enter the design, the code has become a small delivery system.

For either model, cardinality deserves attention before the first dashboard. A time series is identified by its metric name plus attributes. Values such as raw URLs, request IDs, email addresses, or tenant IDs can create a continually expanding set of series. Prefer bounded dimensions such as templated route, operation, outcome, and region. Put per-request identity in logs or traces, where searching for a specific event is the intended operation. Metrics should answer aggregate questions quickly.

## How can one metric core support two transport paths?

The following TypeScript sketch keeps business instrumentation unaware of transport. It is deliberately a teaching implementation, not a replacement for a metrics SDK: it supports counters, a bounded set of labels chosen by the caller, an immutable snapshot, and two adapters. The first adapter returns structured samples to an API client. The second makes the same snapshot available to a scrape handler. Storage and dashboard queries remain downstream concerns.

```ts
type Attributes = Readonly<Record<string, string>>;

type CounterSample = Readonly<{
  name: string;
  description: string;
  attributes: Attributes;
  value: number;
  startedAtUnixMs: number;
}>;

function stableKey(attributes: Attributes): string {
  return Object.entries(attributes)
    .sort(([left], [right]) => left.localeCompare(right))
    .map(([key, value]) => `${key}=${value}`)
    .join("\u0000");
}

class Counter {
  private readonly startedAtUnixMs = Date.now();
  private readonly values = new Map<string, { attributes: Attributes; value: number }>();

  constructor(
    readonly name: string,
    readonly description: string,
  ) {}

  add(amount: number, attributes: Attributes = {}): void {
    if (!Number.isFinite(amount) || amount < 0) {
      throw new RangeError("A counter increment must be finite and non-negative");
    }

    const key = stableKey(attributes);
    const current = this.values.get(key);
    this.values.set(key, {
      attributes: { ...attributes },
      value: (current?.value ?? 0) + amount,
    });
  }

  snapshot(): CounterSample[] {
    return [...this.values.values()].map(({ attributes, value }) => ({
      name: this.name,
      description: this.description,
      attributes,
      value,
      startedAtUnixMs: this.startedAtUnixMs,
    }));
  }
}

const checkoutAttempts = new Counter(
  "checkout_attempts",
  "Number of checkout attempts",
);

export function recordCheckout(outcome: "accepted" | "declined"): void {
  checkoutAttempts.add(1, { outcome });
}

export function collectSnapshot(): CounterSample[] {
  return checkoutAttempts.snapshot();
}
```

That core has no URL, token, timer, or scrape syntax. Good. The push adapter can serialize the snapshot to the contract of a chosen receiver, while the pull adapter can pass it to a standards-aware exposition library. Keeping exposition encoding out of hand-written string concatenation matters because names, help text, attribute escaping, and content negotiation are protocol details rather than business logic.

```ts
type PushOptions = Readonly<{
  endpoint: URL;
  authorization: string;
  timeoutMs: number;
}>;

export async function pushSnapshot(options: PushOptions): Promise<void> {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), options.timeoutMs);

  try {
    const response = await fetch(options.endpoint, {
      method: "POST",
      headers: {
        authorization: options.authorization,
        "content-type": "application/json",
      },
      body: JSON.stringify({
        collectedAtUnixMs: Date.now(),
        temporality: "cumulative",
        samples: collectSnapshot(),
      }),
      signal: controller.signal,
    });

    if (!response.ok) {
      throw new Error(`Metric receiver rejected the batch with ${response.status}`);
    }
  } finally {
    clearTimeout(timeout);
  }
}

type ScrapeEncoder = (samples: readonly CounterSample[]) => Promise<{
  contentType: string;
  body: string;
}>;

export async function handleScrape(encode: ScrapeEncoder): Promise<Response> {
  const encoded = await encode(collectSnapshot());
  return new Response(encoded.body, {
    status: 200,
    headers: { "content-type": encoded.contentType },
  });
}
```

The push example intentionally exports once rather than installing a timer. Scheduling belongs at the application boundary, where shutdown behavior and deployment lifecycle are known. The scrape example accepts an encoder rather than pretending a few string templates implement an exposition standard. In production, replace the teaching registry with an established metrics SDK and select its reader or exporter for the transport. The architectural lesson survives that replacement: handlers call instruments; adapters move observations.

That's the boundary.

Test the system as a pipeline. For push, make the receiver unavailable and verify that export work remains bounded and product requests continue according to policy. Restart the process and verify counter reset handling. For pull, deny collector access and verify that target health changes while the application continues serving normal traffic. For both, send an unexpected label value, inspect resulting series, and confirm that dashboard queries use rates or aggregations consistent with the instrument type. A green dashboard screenshot is not an ingestion test.

## Limits that should change the choice

Stick with scraping when centralized collection timing, direct target health, and an already reliable discovery layer matter more than outbound-only connectivity. Choose API delivery when producers are short-lived or cannot accept collector traffic, provided the team is willing to own bounded delivery behavior. Use a collector when the fleet mixes both shapes or when telemetry processing should move out of each application, but account for that collector as production infrastructure.

Neither transport fixes weak metric design. If the main question is “why was this individual request slow?”, traces are a better starting signal. If the requirement is exact per-tenant billing, use a durable event or ledger path rather than treating a lossy observability pipeline as an accounting system. And if a team does not yet have service discovery, retention policy, or an operator for a self-managed metrics backend, adopting a pull protocol does not make those responsibilities disappear.

There is no universal instance-count crossover. It depends on process lifetime, topology churn, export volume, failure policy, and what the team can operate. Measure those constraints, choose the smaller operational contract, and preserve the instrumentation boundary so the answer can change later.

## References

- OpenTelemetry, “Metrics” — https://opentelemetry.io/docs/concepts/signals/metrics/
- Logback Manual, “Appenders” — https://logback.qos.ch/manual/appenders.html
