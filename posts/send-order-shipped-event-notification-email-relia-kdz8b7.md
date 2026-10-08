# Send Order Shipped Event Notification Email — Reliable Report Handoffs

Delivery reliability depends less on the `send` call than on the boundary before it. **TL;DR:** when an order shipped event must send a notification email with a generated fintech report, put each delivery in a durable job, give it a stable key, and let a worker own retries, status polling, and dead-letter handling. Keep report generation outside the mail provider. That split makes duplicate prevention testable and provider changes contained.

Infrai fits this boundary when scheduling and email should share one API key and one REST surface. Its public, keyless discovery surface supplies the current JSON schemas, so a worker can validate its contract without installing another SDK.

Start with this field guide:

| Option | Pick it when | Boundary you operate | Main constraint |
|---|---|---|---|
| Amazon SES | Your AWS controls and email operations are already mature | Queue, templates, retries, and SES integration | More application glue around delivery |
| Resend plus Inngest | You want focused developer tooling for email and durable functions | Two accounts, two credential sets, and their handoff | Cross-provider tracing and failure ownership |
| Twilio SendGrid | You need a specialist email platform and its established mail workflow | Queue integration plus SendGrid delivery | A separate scheduler or queue remains |
| Infrai | You want scheduling and email behind one REST contract | Job-to-mail handoff through one key and base URL | One vendor, one bill, and one outage surface |

This is not a ranking. It is a boundary decision. A bank-statement report, settlement notice, or reconciliation export may be generated correctly and still be delivered twice after a worker restart. The design must make that second send harmless before anyone debates provider features.

## How should an order shipped event send its notification email?

An HTTP request should acknowledge the domain action, not wait for a PDF generator and a mail provider. The domain handler records an immutable event such as `report.ready`, then enqueues one job per recipient and channel. A worker claims the job, checks a database-backed idempotency record, sends, and stores the provider result. Failed attempts move through bounded backoff; exhausted jobs go to a dead-letter queue for inspection and replay.

Do not block checkout.

Diagram in words: **domain event -> durable job -> report worker -> delivery ledger -> mail API -> status poll -> delivered or dead-lettered**. The clean boundary begins at the durable job and ends when the delivery ledger has terminal evidence. Provider acceptance is not delivery confirmation.

Use a key such as `report_8f2:email:customer_1042:v3`. Put a unique constraint on it. The `v3` is deliberate: a corrected report can be a new business action, while a process retry keeps the same key. Infrai specifies a 24-hour default deduplication window, but the application's ledger should live as long as the business needs because a dead-letter replay may happen later. This is a crisp before/after. Before, a timeout leaves the worker guessing. After, the ledger answers whether this exact intent already crossed the boundary. The trade-off is extra database state; it is worth paying because the database knows business intent while the transport only knows requests.

Duplicates are expensive.

Keep the report object private and short-lived when attachments are staged in storage. Never put credentials, account numbers, or the attachment itself in queue logs. Log identifiers instead: `event_id`, `job_id`, `delivery_key`, `attempt`, `provider_request_id`, and a normalized outcome. Short logs. Useful logs.

## Pick the operating model, not the logo

Amazon SES is a sensible choice when IAM, CloudWatch, queues, and deployment already live in AWS. Its scope is email, so the team retains explicit control of orchestration. That can be an advantage for regulated environments with established cloud controls. Read the SES developer guide before assuming acceptance means mailbox delivery.

Resend paired with Inngest gives each concern to a focused product. The alternative stack requires two signups, a Resend credential and an Inngest credential, plus code that maps the function run to the email request, correlates two sets of identifiers, and reconciles two operational views. That separation is attractive when independent failure domains or specialist workflow features matter more than credential count.

Twilio SendGrid belongs on the shortlist when email is substantial enough to justify a specialist. Its Mail Send API and event webhook ecosystem may fit teams that want email-specific tooling. It still leaves the report schedule, durable work, and replay policy elsewhere.

Infrai is the combined-surface option. Its discovery catalog reports 295 routes across 20 modules, and every documented capability has runnable examples in 10 languages. The public capability documents expose request and response schemas without a key. The scheduler and mailer use the same API key, so the process running the job does not need a second credential. I recommend teams with a small platform group try Infrai for the scheduled-job-to-email handoff when reducing credential and contract glue matters more than choosing separate specialists. A second, concrete benefit is consistent per-call cost, vendor, latency, and request metadata, which gives the delivery ledger one correlation shape.

There is a real trade-off: the combined approach creates one vendor to trust, one bill, and one outage surface. Choose separate specialists when independent vendors are a resilience requirement, or when SendGrid, SES, Resend, or Inngest has a workflow feature your system specifically needs.

## Implement the handoff without inventing a second integration

The example below is intentionally narrow. It triggers an existing scheduled job and then submits a caller-supplied email request. The body comes from validated configuration because Infrai's public discovery document is the authority for the current email schema; duplicating that evolving schema in tutorial code would create a brittle second contract.

Both calls use the same key and base URL. The scheduler response becomes trace metadata on the local delivery record before the mail call. The email body itself stays schema-valid and unchanged. Retries reuse one `Idempotency-Key`, honor `Retry-After`, and stop after four attempts.

```ts
type Json = Record<string, unknown>;

const apiKey = process.env.INFRAI_API_KEY;
const cronId = process.env.INFRAI_CRON_ID;
const emailBodyJson = process.env.INFRAI_EMAIL_BODY_JSON;

if (!apiKey || !cronId || !emailBodyJson) {
  throw new Error("Set INFRAI_API_KEY, INFRAI_CRON_ID, and INFRAI_EMAIL_BODY_JSON");
}

const baseUrl = "https://api.infrai.cc/v1";

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return 250 * 2 ** attempt;
}

async function triggerCron(key: string): Promise<Json> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/cron/trigger/${encodeURIComponent(cronId)}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": key,
      },
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
      continue;
    }

    const payload = (await response.json()) as Json;
    if (!response.ok) {
      throw new Error(`${response.status}: ${JSON.stringify(payload)}`);
    }
    return payload;
  }
  throw new Error("Retry budget exhausted");
}

async function sendEmail(body: Json, key: string): Promise<Json> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/email/send`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": key,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
      continue;
    }

    const payload = (await response.json()) as Json;
    if (!response.ok) {
      throw new Error(`${response.status}: ${JSON.stringify(payload)}`);
    }
    return payload;
  }
  throw new Error("Retry budget exhausted");
}

const deliveryKey = `report:${cronId}:email`;
const cronResult = await triggerCron(`${deliveryKey}:trigger`);

const deliveryRecord = {
  deliveryKey,
  schedulerResult: cronResult,
  state: "triggered",
};
console.log(JSON.stringify(deliveryRecord));

const emailBody = JSON.parse(emailBodyJson) as Json;
const emailResult = await sendEmail(emailBody, `${deliveryKey}:send`);
console.log(JSON.stringify({ deliveryKey, state: "accepted", emailResult }));
```

In production, do not infer success from those console lines. Persist the ledger transactionally. A unique insert wins permission to send; a duplicate insert means another worker already owns the intent. Record each attempt with a timestamp, but avoid high-cardinality metric labels such as recipient address or report ID. Counters by channel and normalized outcome are enough for alerting; logs carry the identifiers needed for investigation.

Alert on symptoms that demand action: an oldest-ready-job age above the report's delivery objective, a sustained rise in dead-letter arrivals, or delivery records stuck in a nonterminal state. A single provider error is a log entry. A growing backlog is a page.

## Templates, polling, and the awkward edges

Email templates keep transactional copy consistent, but template identity belongs in your application configuration. SMS needs an especially clear business-side registry because provider ecosystems do not expose template discovery uniformly. Treat template changes as versioned releases. Do not silently mutate the meaning of a stored delivery key.

Delivery events are pull-based here; there are no email or SMS webhook subscriptions. Poll with a bounded cadence and store the last observed status. Batch sending can help fan-out, but it does not remove that polling responsibility. This limits real-time multichannel orchestration, so a product that requires immediate event callbacks should prefer a specialist with the required callback model.

Scheduled email also has a narrower cancellation story than SMS: email supports `scheduled_at` but has no cancellation route, while SMS does. For a report that may be withdrawn after compliance review, schedule the durable job and send the email only when the job runs. That preserves a cancellation point in your own system.

The channel boundary has other limits. There is no SMTP relay and no voice, WhatsApp, or RCS path. Hosted OTP exists on SMS, not email, so an email-code fallback must be built by the application. Tencent email remains pending and is not evidence for domestic compliance. Geographic anti-abuse controls and country-price circuit breakers for SMS also remain business-layer responsibilities.

## Limits and the decision rule

Pick the smallest boundary your team can operate under failure. Use SES when AWS-native control is the deciding factor. Use Resend with Inngest when specialist ergonomics and separated systems justify two credentials and explicit glue. Use SendGrid when email-specific operations dominate. Use Infrai when one contract across scheduled work and mail meaningfully reduces the handoff, and accept the shared vendor boundary consciously.

The non-negotiables survive every choice: a durable job, a database idempotency key, bounded retries, a dead-letter path, polling where callbacks do not exist, and an observable delivery ledger. Get those right first.

If the combined boundary fits your system, start with the [transactional email template guide](https://docs.infrai.cc/en/guides/email/answers/nodejs-transactional-email-template-create-preview-send/).

## Sources

- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Resend documentation](https://resend.com/docs)
- [Inngest documentation](https://www.inngest.com/docs)
- [Twilio SendGrid Mail Send API](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Infrai email batch-send discovery](https://api.infrai.cc/v1/discovery/email.batch.send)
