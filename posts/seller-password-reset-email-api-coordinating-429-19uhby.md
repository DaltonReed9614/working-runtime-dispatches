# Seller Password Reset Email API: Coordinating 429 Rate Pressure

TL;DR: Treat each seller password reset as one durable delivery intent, then retry that intent after an email API returns 429; never create a fresh reset token or a second logical message merely because one HTTP attempt was throttled. Put new-order mail in a separate queue, honor a valid server delay when one is supplied, add bounded jitter when it is not, and make the worker claim the intent atomically. This keeps integration effort focused on one small state machine instead of scattering retry loops across request handlers.

A gaming marketplace has two urgent-looking messages: a seller needs account recovery, and that seller needs to hear about a new order. They are not the same workload. A reset is security-sensitive and user-triggered. An order notice follows a marketplace event and may arrive in bursts. Sharing an HTTP client is fine; sharing one undifferentiated retry lane is where the operational story gets muddy.

Persist first.

## The before-and-after mental model

Before: the web handler creates a token, calls the mail endpoint, sleeps after 429, and calls again. A process restart loses the retry. Two requests can create two tokens. A long sleep occupies request capacity. Meanwhile, an order burst competes in the same path.

After: the handler records one intent and returns. A worker claims it, makes one send attempt, and records one of three outcomes: delivered, retry scheduled, or terminal failure. Picture the flow as words: seller action -> reset intent -> recovery queue -> mail adapter -> attempt record. Separately: order event -> order intent -> order queue -> the same adapter. The lanes meet only at a deliberate outbound rate controller. **The durable row is the source of truth.** The HTTP response is evidence about an attempt, not permission to manufacture another reset. That distinction also sharpens observability: count intents, attempts, throttled attempts, delivery acknowledgments, and exhausted intents separately. If one chart calls all five "emails," it cannot tell an on-call engineer whether users are waiting, the upstream is limiting traffic, or workers are duplicating work. A dashboard must preserve those categories all the way from ingestion to alerting, because summing them early destroys the signal needed to distinguish a capacity problem from a duplicate-send problem.

## How should a password reset email API handle a 429 rate limit?

Schedule. Do not sleep inside the request that accepted the reset. If the response contains a valid retry delay, use it as the earliest next-attempt time. Otherwise calculate exponential backoff with jitter, cap it, and stop after a locally chosen attempt or age budget. Those limits are application policy, so pick them from the reset-link lifetime and the support experience rather than copying a magic number.

A concrete starting policy can still make the design review useful: three total attempts, a 1,000 ms base delay, and a 30,000 ms cap. Those numbers below are examples, not protocol requirements. The important mechanics are durable scheduling and randomness, which prevents every throttled worker from returning at the same instant.

Retries are scheduling.

Log `intentId`, `attempt`, `status`, `nextAttemptAt`, and a stable error category. Never log the reset token, reset URL, authorization header, or rendered body. Account recovery data deserves the same disciplined handling as other authenticator-related secrets.

## A copyable TypeScript boundary

This example keeps provider details behind a tiny adapter. The database operations are interfaces because the required implementation is transactional: `claimDue` must prevent two workers from owning the same due intent, and each state update must be conditional on the claim.

```ts
type MailIntent = {
  id: string;
  kind: "password_reset" | "new_order";
  recipient: string;
  templateData: Readonly<Record<string, string>>;
  attempt: number;
};

type SendResult =
  | { ok: true; receiptId: string }
  | { ok: false; status: number; retryAfterMs?: number };

interface MailAdapter {
  send(intent: MailIntent, idempotencyKey: string): Promise<SendResult>;
}

interface IntentStore {
  claimDue(lane: MailIntent["kind"]): Promise<MailIntent | null>;
  markDelivered(id: string, receiptId: string): Promise<void>;
  reschedule(id: string, nextAttemptAt: Date, attempt: number): Promise<void>;
  markTerminal(id: string, reason: string): Promise<void>;
}

const MAX_ATTEMPTS = 3;
const BASE_DELAY_MS = 1_000;
const MAX_DELAY_MS = 30_000;

function jitteredDelayMs(attempt: number, random = Math.random): number {
  const ceiling = Math.min(MAX_DELAY_MS, BASE_DELAY_MS * 2 ** attempt);
  return Math.floor(random() * ceiling);
}

async function deliverOne(
  lane: MailIntent["kind"],
  store: IntentStore,
  mail: MailAdapter,
  now = () => new Date(),
): Promise<void> {
  const intent = await store.claimDue(lane);
  if (!intent) return;

  const result = await mail.send(intent, intent.id);
  if (result.ok) {
    await store.markDelivered(intent.id, result.receiptId);
    return;
  }

  const nextAttempt = intent.attempt + 1;
  if (result.status !== 429 || nextAttempt >= MAX_ATTEMPTS) {
    await store.markTerminal(intent.id, `send_status_${result.status}`);
    return;
  }

  const delayMs = Math.max(
    result.retryAfterMs ?? 0,
    jitteredDelayMs(intent.attempt),
  );
  await store.reschedule(
    intent.id,
    new Date(now().getTime() + delayMs),
    nextAttempt,
  );
}
```

The adapter may pass `intent.id` to an endpoint that documents idempotency support. Do not assume every endpoint interprets that key. The application still needs its own unique constraint on the reset request's logical identity, plus conditional state transitions, because a timeout can leave the sender uncertain whether the remote side accepted the message.

Keep the token stable for that intent. A resend request can be modeled as a new intent only after an explicit policy decision invalidates or supersedes the old one. That policy belongs beside token issuance, not inside the mail adapter.

## Doesn't a queue add too much integration work?

It adds a table, a worker, and lifecycle metrics. It also removes timers, retries, and provider-shaped errors from the login endpoint. For a small service, the "queue" can be a database outbox polled by one worker; a separate broker is not a prerequisite. The useful boundary is durable state, not infrastructure branding.

Start with four states: pending, claimed, delivered, and terminal. Add `nextAttemptAt`, `attempt`, and a unique intent key. Test the transitions with a fake adapter that returns 429, a timeout, and success in sequence. Then kill the worker after the fake accepts a send but before `markDelivered`; the second claim is the test that exposes whether the idempotency story is real.

There is a trade-off. Database polling can add delivery latency, while a broker can add operational surface area. For password recovery, choose an explicit polling interval and alert on the age of the oldest pending reset. For new-order messages, watch that metric per lane. A global average can hide a starved recovery queue behind a healthy order queue.

## Can order notices and reset messages share anything?

Yes: transport configuration, domain authentication, suppression handling, and the adapter contract can be shared. Capacity and urgency should remain visible by lane. A weighted scheduler, for example, can reserve worker turns for recovery while still draining order notices; the exact weights come from observed arrival rates and service objectives, not from an email library.

Domain-based authentication matters to both streams. DMARC builds policy and reporting on alignment with SPF and DKIM identifiers, so envelope, signing, and visible From-domain decisions need ownership outside an individual template. Treat that configuration as deployment evidence. Verify it when domains change.

The alert set should be small and actionable: oldest pending intent beyond its objective, sustained 429 ratio by lane, exhausted recovery intents, and a gap between accepted intents and delivery acknowledgments. Page on user impact or depleted retry budget. A single 429 is a scheduling event.

**The integration is complete when duplicate suppression survives restarts and uncertainty.** A happy-path 202 response proves much less. With an app-owned intent ledger, bounded scheduling, isolated lanes, and secret-safe telemetry, password resets and seller order notifications can share a delivery boundary without sharing failure fate.

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- NIST SP 800-63B, Digital Identity Guidelines: Authentication and Lifecycle Management: https://pages.nist.gov/800-63-3/sp800-63b.html
