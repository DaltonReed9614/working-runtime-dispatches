# Pricing-Flag Incidents in a Node.js Game Backend: Error Capture API, Stack Trace Search

Pick the error tracking API you can reconstruct an incident with, and treat the dashboard as a by-product. For a small SaaS game backend on Node.js, the deciding constraint is narrow: when a new pricing rule ships behind a flag and store purchases start failing for one cohort, can you capture every exception together with the flag variant that produced it, keep a stack trace worth reading, and search that set by variant before the rollout window closes? Most candidates survive the demo. Far fewer survive that question, and the alternatives you shortlist should be judged on it rather than on feature counts.

The reason is unglamorous. A pricing change behind a toggle produces two populations of traffic at the same time, and an exception that is not tagged with which population it came from can't tell you whether to widen the rollout or kill it.

## What should a small SaaS backend capture to search exceptions from a pricing flag?

Five things, and the fifth is the one teams forget.

- The exception itself: class name, message, and an unminified stack trace with your own frames on top.
- A stable grouping key, so 4,000 repeats of one defect become one row instead of 4,000.
- Correlation identifiers — a trace id and a span id — so the event can be joined to the surrounding log stream.
- The release or build identifier the process was running.
- The toggle decision: flag name, the variant this request actually got, and a version or hash of the toggle configuration that produced it.

Correlation is the part with an actual standard behind it. The OpenTelemetry logs model treats a log record as a structured event carrying trace context, which is what lets an error record and a log line be recognized as the same unit of work. Carrying those identifiers costs you two string fields per event. Recovering them after an incident costs an afternoon of guessing.

Grouping deserves a moment too, because it decides whether your dashboard is a work queue or a wall of noise. Most services fingerprint on the exception type plus the top application frames. That works well for a `TypeError` thrown deep in a pricing calculator, and badly for errors whose message embeds a unique id — `order 91f3c2 rejected` becomes a fresh group every single time, and your one defect shows up as several hundred. If a candidate API lets you supply the fingerprint yourself, that's worth more to a small team than a nicer chart.

## Before and after: one log stream versus a reconstructed rollout timeline

Picture the pipeline as four stops. Capture boundary, normalized event, group, timeline query.

Before you have that, an incident looks like this: a support ticket says checkout is broken for some players, you grep the last hour of logs for `Error`, you find 63 lines that look similar but not identical, and you spend twenty minutes deciding whether they're one bug or three. Nobody knows which of those requests saw the new pricing rule, because the flag decision lived in memory and was never written down. So the rollout call becomes a vote instead of a measurement.

After, the same incident is a filter. Group by fingerprint, split by `flag.variant`, compare the error rate of the cohort that got the new rule against the cohort that didn't, and pull one representative stack trace from each side. If only the treated cohort throws, the rule is the suspect and you flip the toggle back. If both throw at the same rate, you've just cleared the release and saved yourself a rollback that would have fixed nothing.

That comparison is the whole point. A dashboard that can show you a total count but can't slice it by variant will happily tell you that errors went up, which you already knew from the ticket.

## A capture boundary you can copy

Put one function between your handlers and whichever backend API you choose. It normalizes the exception, attaches the toggle decision, and refuses to let reporting swallow the original failure.

```ts
type FlagDecision = {
  flag: string;           // "pricing_rule_v3"
  variant: string;        // "control" | "treatment"
  configVersion: string;  // hash of the toggle config that produced this decision
};

type CaptureContext = {
  operation: string;      // "purchase.checkout"
  release: string;        // build id of the running process
  flag?: FlagDecision;
  traceId?: string;
  spanId?: string;
};

type ErrorEvent = CaptureContext & {
  name: string;
  message: string;
  stack?: string;
  fingerprint: string;
  occurredAt: string;
};

function toEvent(thrown: unknown, ctx: CaptureContext): ErrorEvent {
  const err = thrown instanceof Error ? thrown : new Error(String(thrown));
  const topFrame = (err.stack ?? "").split("\n")[1]?.trim() ?? "unknown";

  return {
    ...ctx,
    name: err.name,
    message: err.message,
    stack: err.stack,
    // Stable across repeats: ids inside the message would otherwise split one defect
    // into hundreds of groups.
    fingerprint: `${ctx.operation}:${err.name}:${topFrame}`,
    occurredAt: new Date().toISOString(),
  };
}

async function report(event: ErrorEvent): Promise<void> {
  const res = await fetch(process.env.ERROR_INGEST_URL as string, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      authorization: `Bearer ${process.env.ERROR_INGEST_TOKEN}`,
    },
    body: JSON.stringify(event),
  });

  if (res.status === 429) {
    const wait = Number(res.headers.get("retry-after") ?? "2") * 1000;
    await new Promise((r) => setTimeout(r, wait));
    await report(event);
  }
}

export async function withCapture<T>(
  ctx: CaptureContext,
  run: () => Promise<T>,
): Promise<T> {
  try {
    return await run();
  } catch (thrown) {
    // Reporting is best-effort. It must never replace the caller's exception.
    await report(toEvent(thrown, ctx)).catch(() => undefined);
    throw thrown;
  }
}
```

Notice what is not in there: no request object, no headers, no cart contents. Error payloads are the easiest place in a codebase to accidentally copy authorization tokens and player records into a third-party system, and the demo always looks more impressive right before the review board asks where that data went. Name your context fields explicitly and let everything else stay home.

Two production notes on that snippet. Under a real incident you will be throwing thousands of identical exceptions per minute, so the sender needs a cap — a token bucket, or a per-fingerprint counter that reports the first N and then reports counts — otherwise your error pipeline becomes its own outbound traffic problem. And the retry above recurses once per 429; in a service with 12 workers I'd bound it with an attempt counter rather than trusting the upstream to always send a sane `retry-after`.

## What the flag itself owes the reconstruction

A toggle isn't only a branch in your code. It's a dimension in your telemetry, and the difference shows up on the worst day.

Martin Fowler's taxonomy is useful here because it separates a release toggle — short-lived, meant to be deleted — from a long-lived ops or permissioning toggle. The pricing rule in this scenario is a release toggle wearing an experiment costume: it exists to make a deploy reversible, it earns its keep for maybe two weeks, and it should be removed once the rule is either standard or dead. Toggles that overstay produce the combinatorial mess everyone warns about, where four live flags mean sixteen possible code paths and your grouped errors stop mapping to any single configuration.

The practical rule: emit the decision, not the definition. Log which variant a request received at the moment it was evaluated, along with a hash of the config that decided it. Configuration drifts, someone edits a percentage in a console at 2am, and an event that only records `pricing_rule_v3: on` won't tell you that the cohort boundary moved halfway through your incident.

Pair that with one counter per variant in your metrics pipeline and you can alert on the delta rather than the absolute rate. Errors doubling during a rollout is normal if traffic doubled. Errors doubling in the treated cohort alone is a rollback.

## Two objections, and where a capture API stops being enough

"Doesn't our log pipeline already do this?" Partly. Logs hold the surrounding context — the request that came before, the retry that came after — and a grouped error store holds the work queue. The trade-off is that logs make you rebuild grouping with queries every time, and a small team will not keep those queries current. Run both, keep the trace and span ids consistent between them, and accept the overlap in storage cost.

The second objection is sharper: is a compact capture API enough? Sometimes not, and the boundaries are easy to check ahead of time. A backend-only tracker doesn't support source maps, native crash symbolication, or session replay, so if your game client is a Unity build or a browser front end throwing minified frames, stick with a product that handles client-side symbolication as a first-class feature. Silent scheduled jobs are the other gap — a nightly pricing sync that never ran throws no exception at all, and no error API can capture something that didn't happen, so that failure mode needs a heartbeat check instead. And if you need paging, escalation policies, and on-call rotation, buy an alerting product rather than writing a poller that watches your error store, because the poller is easy and the escalation logic is not.

Honestly, I'd distrust any evaluation that skipped the last step. Throw one deliberate exception behind a flag in staging, confirm the event arrives with its variant and config hash intact, repeat it forty times to see whether grouping holds, search for it by variant, open the stack trace, and then check that a human actually gets told. If any of those five steps only works in the vendor's screenshot, the evaluation isn't finished.

## References

- https://opentelemetry.io/docs/concepts/signals/logs/
- https://martinfowler.com/articles/feature-toggles.html
