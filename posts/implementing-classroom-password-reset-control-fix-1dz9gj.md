# Implementing Classroom Password Reset Control — Fix Duplicate Email Sends After Retry Timeout

Short answer: create one durable delivery intent per password-reset request, render the message once, and let a worker claim that intent before sending. If the email provider times out, keep the intent in an `unknown` state and reconcile it; do not create a fresh message just because the caller did not receive an acknowledgment. For an edtech signup flow, apply the same rule to verification links. The application owns the template, token reference, and intent identity. The transport owns delivery.

That boundary is the least complex design that addresses the real failure: a timeout says the result is unknown, not that no email was sent. SMTP and provider APIs cannot give an application end-to-end exactly-once delivery. **Aim for one logical message and controlled recovery, not an impossible exactly-once claim.**

| Template ownership model | Pick this when | Duplicate-control consequence | Main trade-off |
| --- | --- | --- | --- |
| Application-owned render | Security and product teams must review the exact reset copy and link shape | One stored render can be reused across ambiguous retries | Deployments carry template changes |
| Repository-owned template plus transport adapter | Several transports share the same approved content | Intent identity stays independent of the transport | Adapter contracts need careful testing |
| Transport-hosted template | Non-engineers change copy frequently | Store the template version and variables with the intent | A remote edit can change a retry unless versions are pinned |

## How Should Password Reset Email Retry Handle Duplicate Sends?

Pick application-owned rendering when the reset link, legal copy, localization, and security review must move together. This is a strong default for a school platform where a guardian may request a reset while a learner is signing up on another device. The database records one intent; the worker sends one immutable message body. A later template deployment cannot silently alter that in-flight message.

Pick a repository-owned template with a generic transport adapter when email is one channel among several. The domain code should emit `PasswordResetRequested`, not a provider-specific payload. The adapter receives a rendered message and a stable intent ID. This keeps retries from inheriting transport details.

Pick a transport-hosted template when operational copy changes matter more than local reproducibility. Record the remote template version and all non-secret variables at intent creation. Never store a raw reset token in logs or metrics. Store a one-way token fingerprint or an internal token record ID instead.

The diagram in words is short: **request handler -> reset record -> delivery intent -> claiming worker -> transport -> reconciliation**. The handler returns after the durable records exist. It does not wait for the network send.

One row. One render.

## Implement the durable intent

The key must describe the user's action, not an HTTP attempt. A client retry with the same reset request ID maps to the same intent. A genuinely new reset request gets a new ID and can revoke the prior token according to the account-security policy.

The following TypeScript uses a generic transactional store and transport. The compare-and-set claim prevents two workers from sending the same ready intent concurrently. The rendered subject and HTML remain fixed.

```ts
type DeliveryState = "ready" | "sending" | "unknown" | "sent" | "failed";

type ResetIntent = {
  id: string;
  requestId: string;
  recipient: string;
  templateVersion: string;
  subject: string;
  html: string;
  state: DeliveryState;
  attempts: number;
  providerReceipt?: string;
};

interface IntentStore {
  insertOnce(intent: ResetIntent): Promise<ResetIntent>;
  claim(id: string, from: DeliveryState[]): Promise<ResetIntent | null>;
  markSent(id: string, receipt: string): Promise<void>;
  markUnknown(id: string): Promise<void>;
}

interface MailTransport {
  send(input: {
    intentId: string;
    to: string;
    subject: string;
    html: string;
  }): Promise<{ receipt: string }>;
}

export async function deliverReset(
  intentId: string,
  store: IntentStore,
  transport: MailTransport,
): Promise<void> {
  const intent = await store.claim(intentId, ["ready"]);
  if (!intent) return;

  try {
    const result = await transport.send({
      intentId: intent.id,
      to: intent.recipient,
      subject: intent.subject,
      html: intent.html,
    });
    await store.markSent(intent.id, result.receipt);
  } catch (error) {
    // A network timeout is ambiguous: the transport may have accepted the message.
    await store.markUnknown(intent.id);
    throw error;
  }
}
```

Back the `requestId` with a unique constraint. Insert the reset token record and delivery intent in one database transaction. The link should resolve through an opaque, single-use credential with an expiry selected by the application's security policy; do not encode account details into the URL merely to avoid a lookup.

There is a sharp edge here. A worker can send successfully and crash before `markSent`. The row remains `sending`, so a lease is useful for detecting abandoned work, but blindly releasing that lease to `ready` recreates the duplicate. Move expired claims to `unknown` instead. Small distinction. Big effect.

Consider the exact sequence. At 10:00:00, a guardian asks for a password reset and the transaction creates request `R1`, token record `T1`, and intent `I1`. A worker claims `I1`, submits its immutable render, and waits. The connection times out after the remote side may have accepted the bytes. At this point there are still only three durable records; the system changes `I1` to `unknown`, and a browser retry carrying `R1` finds the same intent. Creating `I2`, rendering a new link, or immediately returning `I1` to `ready` would turn missing acknowledgment into a second send. Reconciliation instead asks what became of `I1`. This example uses wall-clock labels to make ordering visible, not to prescribe a timeout duration.

## Treat a timeout as evidence, not permission

Timeouts are ambiguous.

Reconciliation needs a decision tree. First, query the transport by the stable intent ID when that capability exists. If it reports acceptance, store the receipt and stop. If it definitively reports rejection before acceptance, return the same intent to `ready`. If it cannot resolve the outcome, wait for a bounded uncertainty window chosen from observed acknowledgment latency and the reset token lifetime, then make an explicit policy decision.

That final branch is a trade-off. Retrying favors delivery but risks a second email. Suppressing favors a quiet inbox but risks leaving the learner or guardian without a link. Capture the decision as `retry_after_unknown` or `suppress_after_unknown`, not as an invisible exception path. An operator can then explain what happened without reading raw logs.

Make that choice explicit.

Use a stable `Message-ID` when generating RFC 5322 mail, but do not treat it as a universal deduplication lock. It provides message identity for mail handling and threading; downstream systems are not required to suppress another accepted submission carrying the same value. DKIM signs selected headers and the body so receivers can validate responsibility for a message. It does not create exactly-once delivery.

## Make duplicate behavior observable

Three counters tell most of the story: intents created, transport submissions, and intents entering `unknown`. Break them down by template version and transport class, never by email address. A useful invariant is that one request ID creates one intent. Alert when the ratio of submissions to intents rises, and separately when unknown outcomes age beyond the reconciliation window.

Structured events should include `intent_id`, `request_id`, `template_version`, `state_before`, `state_after`, `attempt`, and a normalized outcome. Exclude the reset URL, token, recipient, and rendered body. This gives support enough context to answer “why did I get two emails?” without turning the observability system into a credential store.

Test the crash boundaries deliberately. Inject a timeout before transport acceptance, after acceptance but before the response, and after the response but before the database update. Run two workers against the same ready row. Then assert the state transitions and number of transport submissions. The middle case is the one simplistic retry tests miss.

For deployment, ship the schema and dormant worker logic first. Next enable intent creation while retaining the old sender as a monitored fallback path. Finally switch delivery to claimed intents and watch the submission-to-intent ratio, unknown-state age, and reset completion rate. Crisp before. Crisp after.

## Limits and decision rule

This pattern controls duplicate work inside the application boundary. It cannot prevent a receiving mail system from duplicating, forwarding, delaying, or reordering a message. It also cannot prove that a person saw the email. Delivery receipts and link redemption are different signals.

Choose template ownership by asking who must reproduce the exact message during an ambiguous retry. If engineering and security own that obligation, store an immutable render locally. If another system owns the template, pin its version and preserve the variables. In both cases, one request ID must still map to one durable intent.

Do not promise exactly once. Promise explainable states, stable identity, and a retry policy that treats uncertainty honestly.

## Further reading

- RFC 5322, Internet Message Format: https://www.rfc-editor.org/rfc/rfc5322
- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
