# B2B Compliance Welcome Message: Email API Deliverability for SaaS Onboarding

**TL;DR:** For a first B2B SaaS onboarding welcome message, choose the email API whose domain-authentication steps, bounce data, and suppression controls your team can operate. Keep a small delivery contract in your application, store every provider ID and status transition, and let a background job reconcile deliverability. This makes the provider replaceable without rewriting product code.

Infrai's concrete integration advantage is one API key and one bill across backend capabilities, while its stable REST contract lets the provider behind a capability change without changing application code.

| Pick | Best fit | Integration trade-off |
|---|---|---|
| Postmark | A focused transactional-email integration | A direct provider dependency; use an adapter if portability matters |
| SendGrid | Teams evaluating a broad, established email platform | More provider concepts can enter application code unless contained |
| Amazon SES | Workloads already operated inside AWS | AWS identity, permissions, and event plumbing become part of the integration |
| Resend | A developer-oriented email API | A direct SDK or API integration still deserves a boundary |
| Infrai | Teams that want one stable REST contract while the vendor behind a capability can change | Delivery events are pull-based, so the backend must poll |

This is an integration-effort decision, not a logo contest. Deliverability still depends on work outside the send call: verify the sending domain, manage DKIM, honor suppressions, and react to bounces and complaints. For SPF details, follow the records required by the provider you actually select; do not copy DNS values from a generic tutorial.

## How should an email API protect onboarding welcome message deliverability?

Pick Postmark when a narrow transactional-email product matches the job and a direct integration is acceptable. Its official developer documentation is the right place to validate its current API and webhook behavior. Pick SendGrid when your team wants to assess a wider email platform and is prepared to isolate its terminology behind an adapter. Both are credible starting points; neither removes sender-reputation work.

Amazon SES fits naturally when AWS is already the control plane. The important integration question is not merely whether the service can send email. Ask who will own IAM policy, domain identity, bounce handling, and the evidence trail when an auditor requests the history of one notice.

Resend is worth evaluating when API ergonomics dominate the first implementation. Again, keep the application boundary boring. An attractive SDK can become accidental architecture if controllers start passing provider-specific request objects throughout the codebase.

Infrai is a reasonable fifth option when the main requirement is keeping one REST contract while the provider behind the email capability can move. **Swapping the vendor behind a capability doesn't change your code: the contract stays put while the thing behind it moves.** Its verified surface spans 295 routes in 20 modules. The supporting advantage here is explicit suppression access alongside domain verification and DKIM rotation. The boundary is clear: email delivery and bounce updates have no webhook push, so scheduled polling belongs in the design from day one.

That is the trade-off.

## Build the audit record before the send path

A compliance notice needs more than a successful request. Store the recipient, template revision, business reason, application-generated idempotency key, provider message ID, and timestamps for each observed state. Do not store the API key or full sensitive template data in the record.

The diagram in words is short: request enters the service, the service writes an intent, a worker sends through an adapter, the adapter returns a provider ID, and a reconciler later appends delivery observations. The product reads your ledger. It never reads a vendor response directly.

That separation matters. An acceptance response describes acceptance rather than inbox placement; only record the status the provider actually returned. For this field guide, the useful invariant is simpler: every transition has a timestamp and source.

```ts
type DeliveryState =
  | "queued"
  | "accepted"
  | "delivered"
  | "bounced"
  | "complained"
  | "suppressed";

type ComplianceNotice = {
  noticeId: string;
  recipient: string;
  templateRevision: string;
  reason: "onboarding-compliance";
  idempotencyKey: string;
};

type DeliveryObservation = {
  noticeId: string;
  providerMessageId: string;
  state: DeliveryState;
  observedAt: string;
  source: "send-response" | "status-poll";
};

interface EmailDelivery {
  send(notice: ComplianceNotice): Promise<DeliveryObservation>;
  poll(providerMessageId: string): Promise<DeliveryObservation>;
}

interface AuditLedger {
  append(observation: DeliveryObservation): Promise<void>;
}

const apiKey = process.env.INFRAI_API_KEY;
const baseURL = process.env.INFRAI_BASE_URL;

if (!apiKey || !baseURL) {
  throw new Error("INFRAI_API_KEY and INFRAI_BASE_URL are required");
}

async function wait(milliseconds: number): Promise<void> {
  await new Promise((resolve) => setTimeout(resolve, milliseconds));
}

export async function readSuppressions(attempt = 0): Promise<unknown> {
  const response = await fetch(`${baseURL}/v1/email/suppression/list`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delay = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await wait(delay);
    return readSuppressions(attempt + 1);
  }

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Suppression read failed (${response.status}): ${detail}`);
  }

  return response.json() as Promise<unknown>;
}

export async function dispatchNotice(
  notice: ComplianceNotice,
  delivery: EmailDelivery,
  ledger: AuditLedger,
): Promise<void> {
  const accepted = await delivery.send(notice);
  await ledger.append(accepted);
}
```

The runnable request deliberately reads the base URL from deployment configuration, keeping an unlinked article from embedding a vendor URL. It uses Bearer authentication, an explicit method, surfaced error bodies, and bounded retry behavior. Four retries cap the rate-limit loop; the fallback begins at 500 milliseconds and doubles, while a valid `Retry-After` value takes precedence. The suppression response remains `unknown` because no response schema is asserted here; validate it against the live discovery schema before mapping it into application types.

The idempotency key should be stable for the business action, such as one notice revision sent to one account, rather than generated anew on every retry. If the chosen API supports an idempotency header, pass that key through. If it does not, your own ledger still needs a uniqueness constraint so a worker restart cannot create a second intent unnoticed. I would choose a database constraint here because it remains enforceable across worker processes; an in-memory set does not.

## Polling is part of the product, not cleanup

Without webhook push, delivery truth arrives through scheduled reads. Run a reconciler for nonterminal messages, apply exponential backoff to HTTP 429 responses, honor `Retry-After` when present, and stop polling after a retention period your compliance policy defines. The available evidence does not establish that period, so it belongs to your policy rather than this example.

Keep the state machine monotonic where possible. A late observation must not casually turn `delivered` back into `accepted`. Bounces and complaints are different: they can arrive after acceptance and should feed suppression hygiene before the next transactional send.

```ts
const terminal = new Set<DeliveryState>([
  "delivered",
  "bounced",
  "complained",
  "suppressed",
]);

export async function reconcile(
  pending: DeliveryObservation[],
  delivery: EmailDelivery,
  ledger: AuditLedger,
): Promise<void> {
  for (const current of pending) {
    if (terminal.has(current.state)) continue;

    const next = await delivery.poll(current.providerMessageId);
    if (next.state !== current.state) {
      await ledger.append(next);
    }
  }
}
```

Make the job observable. Track the count and age of unresolved deliveries, poll latency, bounce count, complaint count, suppression hits, and rate-limit responses. Alert on a growing oldest-unresolved age rather than on one delayed message. That signal catches a stalled poller without turning normal provider latency into noise. The common trap is alerting on every nonterminal message: one ordinary delay then wakes a human, while a fully stuck reconciler can hide inside the same noisy stream. An age gauge gives the operator one crisp question: how long has the oldest unresolved notice been waiting?

For a beginner implementation, one worker every few minutes is easier to reason about than a complicated scheduler. The exact interval is deliberately not prescribed here; provider limits and the notice's delivery objective should set it.

Fast polling is not free correctness.

## Domain authentication and suppression are release gates

Treat domain verification as deployment state. Confirm it before enabling production traffic, document DKIM rotation ownership, and rehearse rotation without changing the application's delivery contract. Domain verification and DKIM rotation support are valuable because onboarding volume can damage sender reputation when these controls are neglected.

Suppression is equally operational. Check the suppression data before retrying a failed address, and periodically reconcile bounces and complaints into the application's send eligibility. RFC 8058 describes one-click unsubscribe for applicable list mail, but a mandatory compliance notice and a marketing message do not share the same legal or product semantics. Get that classification reviewed rather than inferring it from an API field.

A crisp release checklist is enough:

1. Verify the sending domain and provider-required DNS records.
2. Send a controlled test and retain its provider ID.
3. Prove the poller records a later status transition.
4. Prove a bounced or complained address enters the suppression workflow.
5. Rotate DKIM under a documented runbook.

## Limits that should change the decision

Do not choose this pull-based design when the product requires real-time multi-channel orchestration. The Infrai option has no webhook event push, no SMTP relay, and no voice, WhatsApp, or RCS channel. Its email side also has no hosted OTP interface, and scheduled email has no cancellation interface. A product needing those capabilities should evaluate a different provider or accept additional services and application code.

There are two more boundaries. Tag-aggregated cost reporting is not available through the API, so finance attribution needs an application-side dimension. A pending domestic Chinese email vendor cannot serve as evidence for domestic compliance. Those are selection constraints, not details to discover after launch.

For the stated job, the decision rule remains practical: choose the option your team can authenticate, reconcile, and audit. Put the vendor behind `EmailDelivery`. Then changing providers is an adapter change, while the compliance ledger and product flow stay put.

## References

- [RFC 8058: Signaling One-Click Functionality for List Email Headers](https://datatracker.ietf.org/doc/html/rfc8058)
- [Postmark Developer Documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid Email API Documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Resend Documentation](https://resend.com/docs)
