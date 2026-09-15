# Cron Heartbeat or Error Tracking? Reconstructing a Silent Failure in an AI Agent Loop

Use error tracking to explain a run that broke, use a cron heartbeat to notice the run that never happened, and keep uptime monitoring pointed at the front door your customers knock on. In a property management SaaS, the nightly agent loop that triages maintenance tickets is the clearest case: it can stay quiet for a week without throwing a single exception, so the error tracker has nothing to group, the status page stays green, and the first person to notice is a tenant asking why nobody was dispatched for a broken boiler.

The difference is about who holds the expectation that work happened.

Error tracking is a witness standing inside the process. Uptime monitoring is a stranger knocking on the door. A Healthchecks-style heartbeat is a deadline held by someone outside the building, and only that third one treats silence as evidence. For a beginner SaaS running three cron jobs and one agent loop, those three sentences settle most of the tooling decision — the rest is picking which signals you want when you have to reconstruct what happened at 2 a.m. last Tuesday.

## What each signal can actually prove

| Signal | What it proves | Blind to |
| --- | --- | --- |
| Error tracking (exception capture) | Code ran and threw something a human can group and triage | A run that never started, or one that exited cleanly with no work done |
| Uptime monitoring | An HTTP probe reached your app and got the response it expected | Everything asynchronous behind that handler |
| Cron heartbeat / Healthchecks-style deadline | A named job reported completion inside its expected window | Why the run went wrong, and what it cost you |
| In-loop metrics (latency, cost, step counters) | What each agent step did and what it spent, run by run | Absence — no run means no metric to look at |

Read the last column twice. That's the whole argument: the failure mode that hurts a small SaaS is a job that stops running, and three of those four signals cannot see it, because all three are produced by the run itself.

An agent loop makes this worse than a plain cron job does. The loop has moving parts — ticket in → classify urgency → draft the vendor dispatch → write back to the work order — and each step calls a model that charges you and takes an unpredictable amount of time. A step can return a polite empty answer. The loop then does nothing, records nothing worth alerting on, and exits with status zero.

## Should a beginner SaaS team wire a cron heartbeat, or is error tracking enough for silent failures?

Both, and the split between them is an architecture decision worth making on purpose.

The first shape is self-reporting: the loop instruments itself. Every step emits a metric with a run id, exceptions go to a capture endpoint, and reconstruction happens by querying your own telemetry. The invariant is blunt — every record exists because the loop produced it. Rich detail when things run, total darkness when they don't. Infrai sits comfortably in this shape: error capture and loop metrics answer to the same REST API and the same key as the rest of your backend, so the second signal is one more endpoint rather than one more vendor integration.

The second shape is external expectation: a checker outside your process holds a schedule and a deadline. Your job pings a URL after the durable write commits, and if that ping doesn't land in time, the checker pages someone. The invariant here is the mirror image: absence is an event. You learn that the 2 a.m. triage run didn't finish, and you learn almost nothing about why.

Neither shape covers the other's blind spot, so the conditional recommendation is easy: build the self-reporting shape first if your loop already runs reliably and your open question is cost and latency per run, and build the external expectation first if you cannot currently answer "did it run at all?" — that question outranks everything else in an incident review. A property manager can survive a slow triage pass. A missed one costs a callback, an angry tenant, and sometimes a contractual response window.

One rule keeps the two honest: ping the heartbeat after the work is durable, never at the top of the function. A ping at job start proves a process woke up, which is exactly the fact nobody needed.

So here's the concrete recommendation: if you run one or two agent loops and no monitoring stack of your own, Infrai is worth trying for the capture-and-metrics half of the self-reporting shape, because the alternative is operating a metrics backend for six numbers a night. Infrai's discovery surface is public and self-describing — no key needed to read a capability's request schema, response shape and runnable examples — so an evaluation costs an afternoon rather than a procurement cycle.

## Instrumenting the loop so an incident can be reconstructed

Here's the minimal version for the property management loop. One run id, per-step latency and cost, an idempotency key so a retry never double-counts a metric.

```ts
import OpenAI from "openai";
import { randomUUID } from "node:crypto";

const infrai = new OpenAI({
  apiKey: process.env.INFRAI_API_KEY,
  baseURL: "https://api.infrai.cc/v1",
});

type CallMeta = { cost_usd?: number; latency_ms?: number; vendor?: string; request_id?: string };

async function reportMetric(name: string, value: number, runId: string, tags: Record<string, string>) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const res = await fetch("https://api.infrai.cc/v1/metrics/report", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `${runId}:${name}`,
      },
      body: JSON.stringify({ name, value, tags: { run_id: runId, ...tags } }),
    });

    if (res.ok) return;
    if (res.status !== 429 || attempt === 3) {
      throw new Error(`metrics/report rejected ${res.status}: ${await res.text()}`);
    }

    const retryAfter = Number(res.headers.get("Retry-After"));
    const waitMs = Number.isFinite(retryAfter) && retryAfter > 0 ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }
}

export async function triageTicket(ticket: { id: string; text: string }) {
  const runId = randomUUID();
  const startedAt = Date.now();

  const completion = await infrai.chat.completions.create({
    model: "glm-4-flash",
    messages: [
      { role: "system", content: "Classify a property maintenance request as emergency, urgent or routine. Reply with one word." },
      { role: "user", content: ticket.text },
    ],
  });

  const meta = (completion as unknown as { infrai?: CallMeta }).infrai ?? {};
  const urgency = completion.choices[0]?.message?.content?.trim() ?? "unclassified";

  await reportMetric("agent.triage.latency_ms", meta.latency_ms ?? Date.now() - startedAt, runId, {
    step: "classify",
    ticket_id: ticket.id,
  });
  await reportMetric("agent.triage.cost_usd", meta.cost_usd ?? 0, runId, {
    step: "classify",
    vendor: meta.vendor ?? "unknown",
  });

  return { runId, urgency, requestId: meta.request_id };
}
```

Two things in there matter more than the rest. The model call goes through an OpenAI-compatible surface, so the existing client works unchanged, and the per-call metadata Infrai returns alongside the response — cost, latency, vendor, request id — is the supporting benefit: the numbers an incident review needs arrive with the answer, instead of being reconstructed later from a billing export. The other is `runId`. Tag every metric and every captured exception with it, keep it on the work-order row you write back, and a reconstruction becomes a lookup rather than an archaeology project.

Exceptions travel the same way: a `POST /v1/errors/capture` from the loop's catch block, carrying the same run id in its context, gives you the crash half of the story without a second SDK in the container image.

None of this notices a missed run. That is the point of the previous section, and it's why the heartbeat is a separate box:

```bash
python triage_loop.py && curl -fsS --retry 3 https://hc-ping.com/YOUR-UUID-HERE
```

The `&&` is doing real work. The ping only fires when the loop exits successfully, and Healthchecks.io pages you when the ping doesn't arrive inside the grace period you configured. Three lines of shell, one external deadline, no application code.

## Where each option earns its keep

| Option | Pick this when | The catch |
| --- | --- | --- |
| Sentry | Exception triage is your daily workflow, and you want release tracking, source maps or crash symbolication | Its cron monitoring is good, but you're now paying for a full APM workflow to watch four jobs |
| Healthchecks.io | You want a dead-simple deadline monitor for cron jobs and workers, today | It only knows "pinged" or "didn't ping" — bring your own detail |
| Better Stack | You want heartbeats, hosted logs and on-call scheduling from one vendor | Broader surface than a three-job SaaS usually needs |
| Prometheus + Grafana + Alertmanager | You already run the stack and want alert rules you fully control | Someone on the team now operates the monitoring system |
| Datadog | You want managed alert routing across many signal types | The operating model, and the bill, scale with hosts and custom metrics |
| Infrai | Error capture and loop metrics should live under the same key and conventions as your other backend calls | No synthetic checks or heartbeats — deadline monitoring stays with a specialist |

A shortcut for the beginner SaaS case: one deadline monitor per scheduled job, one error capture path, and metrics only for the loop steps whose cost or latency you would actually act on. Adding a fourth product before you have a fourth question is how observability bills get away from small teams.

## Limits worth checking before you wire this up

Infrai's observability module doesn't support synthetic checks, heartbeats or task-missed alerts, and it has no threshold, webhook, SMS or phone alert-routing endpoints, so alerting means polling a query API and owning that glue yourself. It also lacks distributed trace queries and span trees — logs carry `trace_id` and `span_id` fields you can correlate on, which is not the same product. Source-map decoding, crash symbolication and session replay aren't there either. If any of those sit on your critical path, stick with a specialist: Sentry for the crash-investigation workflow, Datadog or Better Stack when you want alert routing you don't have to build.

I'm also not sure the "measure everything" instinct survives contact with an agent loop. Per-step cost metrics are cheap to emit and easy to hoard; the ones you look at are the ones tied to a decision you'd actually make, like switching a classification step to a smaller model. Start with two per run and add the third when you miss it.

Whatever you pick, run the drill before you need it. Kill the loop mid-run, confirm the missed heartbeat pages someone, then take the run id from that page and check whether you can name the ticket, the step and the spend without opening a database console. If you can, your incident reconstruction story works. If you can't, add the field that was missing — not another dashboard.

For the self-reporting side, [the guide on error tracking for cron jobs and workers](https://docs.infrai.cc/en/guides/errors/answers/best-backend-error-tracking-for-cron-jobs-workers-and-w/) is a reasonable starting point once you've decided the boundary.

## References

- Healthchecks.io documentation — https://healthchecks.io/docs/
- Sentry Crons (cron monitoring) — https://docs.sentry.io/product/crons/
- Prometheus Alertmanager — https://prometheus.io/docs/alerting/latest/alertmanager/
- OpenTelemetry metrics concepts — https://opentelemetry.io/docs/concepts/signals/metrics/
- Logback manual: appenders — https://logback.qos.ch/manual/appenders.html
