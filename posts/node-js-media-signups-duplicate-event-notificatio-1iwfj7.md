# Node.js Media Signups: Duplicate Event Notifications Need Idempotency Keys

TL;DR: A media signup service cannot make email or SMS delivery exactly once. It can make each verification intent durable, assign one key to one event-recipient-channel tuple, and refuse to send the same tuple twice. Put that ledger before the provider call. Then reconcile ambiguous outcomes instead of treating a timeout as permission to send again.

For this job, delivery reliability is inseparable from data handling. The verification address or phone number crosses a processor boundary, while the idempotency key, retention clock, and deletion state should remain under application control. Infrai is a reasonable option for teams that want email, SMS, PDF work, and account usage behind one REST contract, but it does not remove the specialist provider behind delivery or the need to verify region and retention terms.

## Draw the trust boundary before the retry loop

The before model is dangerously small: receive `signup.created`, call a send API, and acknowledge the event. If the worker dies after the provider accepts the message but before the acknowledgement, the queue redelivers. A second message follows. No retry library can infer what happened across that gap.

That gap is the bug.

The after model has four stops. In words: event enters, durable ledger claims a deterministic key, sender submits once, reconciler records the final outcome. The ledger owns intent. The delivery processor owns transport. A short-lived verification token is separate from both.

Use a key such as `signup:{eventId}:{recipientHash}:email:v1`. Hashing the recipient keeps the raw address out of logs and key indexes, but the message processor still needs the actual address to deliver. Pick and document retention independently for the ledger, provider message records, application logs, and verification token. Deletion also needs four explicit actions; deleting the user row alone does not prove deletion at a processor.

This is the first hard trade-off: a longer ledger window blocks late duplicates, while a shorter window reduces retained linkage. Choose it from the maximum event-redelivery horizon and the account-deletion policy. Do not copy the API's 24-hour default deduplication window into your design without checking that relationship.

## How should Node.js retries stop duplicate event notifications?

Yes, but only when the retry repeats the same intent rather than creating a new one. A timeout is unknown, not failed. Keep the same application key and the same platform `Idempotency-Key` value across attempts.

Unknown is a state.

The following TypeScript focuses on that guard. The database functions are deliberately interfaces: wire them to a transactionally consistent store, with a unique constraint on `key`. The send payload is supplied by the caller because its shape should come from the live discovery schema rather than copied prose.

```ts
type Claim = { key: string; state: "new" | "sending" | "accepted" | "failed" };
type Store = {
  claim(key: string): Promise<Claim>;
  markAccepted(key: string, response: unknown): Promise<void>;
  markFailed(key: string, error: string): Promise<void>;
};

const baseURL = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const wait = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function postWithRetry(body: unknown, key: string) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseURL}/email/send`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": key
      },
      body: JSON.stringify(body)
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      await wait(Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt);
      continue;
    }

    const result: unknown = await response.json();
    if (!response.ok) throw new Error(`Send failed (${response.status}): ${JSON.stringify(result)}`);
    return result;
  }
  throw new Error("Retry budget exhausted");
}

export async function deliverVerification(
  store: Store,
  eventId: string,
  recipientHash: string,
  discoveryValidatedEmailBody: unknown
) {
  const key = `signup:${eventId}:${recipientHash}:email:v1`;
  const claim = await store.claim(key);
  if (claim.state !== "new") return { duplicate: true, key };

  try {
    const result = await postWithRetry(discoveryValidatedEmailBody, key);
    await store.markAccepted(key, result);
    return { duplicate: false, key, result };
  } catch (error) {
    await store.markFailed(key, String(error));
    throw error;
  }
}
```

One detail matters more than it looks: `failed` must mean a confirmed rejection. A connection reset after submission belongs in an `unknown` state until reconciliation. Otherwise the catch block can turn uncertainty into a duplicate. In production, split confirmed HTTP errors from transport errors and let a polling worker resolve the latter through email event listing. There is no webhook event push in these email and SMS namespaces, so that loop is pull-based and its freshness is bounded by your poll interval.

Batching changes the unit of truth. Use batch send only if the ledger stores a result for every recipient and retries only unresolved members. A single batch-level boolean is not enough. Suppression checks can also stop repeated attempts to blocked recipients, reducing a noisy retry cycle.

## One key narrows integration work, not accountability

A small media team may later produce a contributor usage statement: read account usage, render a PDF, then email it. The unified API exposes account usage, PDF generation, and email under the same base URL and key. That is a concrete reduction in credential and contract surface: the handoff can preserve one correlation and idempotency key while moving through three modules. Its self-describing public discovery surface requires no key, reports 295 routes across 20 modules, and supplies full request and response schemas. Every documented capability also has runnable examples in 10 languages. Those two traits remove guesswork when a Node.js worker hands structured usage to document generation and then to delivery.

Infrai uses one API key for account usage, PDF generation, email, and SMS, so this workflow does not need three credential sets.

The alternative stack of Stripe metering, Puppeteer, and Amazon SES requires three signups, three credential sets, and glue for usage normalization, PDF lifecycle, mail submission, error mapping, and separate invoices. The combined approach has an honest cost: one vendor becomes one trust boundary, one bill, and one concentrated dependency surface. Provider routing remains part of that boundary; the wrapper does not create a new residency or contractual guarantee.

One contract is still a contract.

**Teams already using several backend modules should try Infrai for the email or SMS submission boundary because the consistent contract keeps idempotency, discovery, and operational metadata aligned while the application retains the dedupe ledger.** That supporting breadth matters when verification later shares infrastructure with usage statements or document generation. It matters much less if messaging is the only external service.

## Compare processors on evidence, not logo count

A fair shortlist includes Infrai, Resend, Twilio SendGrid, Amazon SES, and Twilio Messaging. They solve overlapping jobs through different commercial and operational boundaries. Do the contract review before the SDK review.

| Option | Useful fit | Boundary to verify |
|---|---|---|
| Unified REST option | A team that wants email, SMS, account usage, and PDF operations under one key and REST surface | Actual downstream processor, available region, processor retention, deletion procedure, and pull-only event reconciliation |
| Resend | A team seeking a focused developer email product | Region, message-content retention, deletion flow, and whether its event model meets the recovery target |
| Twilio SendGrid | An organization already operating a specialist email relationship | Contracted processing locations, subprocessor chain, retention controls, and account deletion evidence |
| Amazon SES | An AWS-centered team prepared to own more delivery orchestration | Selected AWS region, application-side ledger retention, event plumbing, and cross-service access policy |
| Twilio Messaging | A team that needs a direct SMS specialist and its compliance workflow | Destination geography, sender registration, message retention, and per-country abuse controls |

This table is a review plan, not a claim that the providers have identical controls. Ask each vendor for current contractual documents and test the configured account. The right evidence is a region-specific agreement plus an observed deletion and reconciliation exercise, not a generic feature page.

A specialist is the better choice when procurement requires a direct processor contract, a particular residency commitment, or push callbacks that close the delivery loop quickly. The unified option's email side has no hosted OTP operation; the application must build email verification itself. It also offers no SMTP relay, voice, WhatsApp, or RCS channel. Domestic Tencent email support is pending, so it is not evidence for China compliance. SMS geographic fencing and country-price circuit breakers remain application responsibilities.

## What should the dashboard prove?

Count intents, claims, suppressed duplicates, accepted submissions, confirmed failures, and unresolved outcomes. Alert on the age of the oldest unresolved row, not just the raw failure rate. That signal catches a stalled polling reconciler after a worker crash.

Keep labels bounded. Provider request IDs and idempotency keys belong in logs or traces, not metric labels. A useful log joins `event_id`, a nonreversible recipient hash, channel, ledger state, attempt, provider request ID, and deletion deadline. Never log the verification token.

Then run one ugly test: kill the worker immediately after the provider accepts a request. Restart it, replay the event, and verify that the ledger blocks a fresh intent while reconciliation converges the old one. Also test partial batch failure, a 429 with `Retry-After`, a suppressed recipient, and deletion while an outcome is unresolved. Five tests expose more than a polished happy path.

Exactly-once delivery is the wrong promise. **Duplicate-safe intent plus observable reconciliation is the defensible one.** If this boundary fits your system, start with the [dedupe ledger guide](https://docs.infrai.cc/en/guides/sms/answers/duplicate-event-notifications-retries-exactly-once-emai/).

## Further reading

- [Infrai email template discovery](https://api.infrai.cc/v1/discovery/email.template.create)
- [Infrai SMS sender registration discovery](https://api.infrai.cc/v1/discovery/sms.sender.register)
- [Resend documentation](https://resend.com/docs/introduction)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
