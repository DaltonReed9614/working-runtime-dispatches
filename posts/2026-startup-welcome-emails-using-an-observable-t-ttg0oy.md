# 2026 Startup Welcome Emails Using an Observable Transactional Email Provider Contract

**TL;DR:** For a marketplace sending a signup verification link, choose the candidate that reaches a small, observable delivery contract with the least application-specific code. Put Postmark, Resend, Brevo, and Mailgun through the same acceptance test. Do not choose from a headline price or a successful API response. The useful comparison is the engineering distance from `signup.created` to a verified account, including retries, event correlation, suppression handling, and an exit path.

| Option | Pick this when | Integration work to prove before launch | Main limit |
|---|---|---|---|
| One managed email API | The team wants one narrow path from signup to verification | Send adapter, signed event ingestion, idempotency, delivery metrics, suppression flow | The application can quietly absorb provider-specific fields |
| Existing cloud email service | Identity, access, and telemetry already live in that cloud | Permissions, domain setup, event routing, dashboards, runbook | A short send call can hide a longer cloud integration |
| Email with SMS recovery | Email alone cannot meet the account-recovery policy | Channel consent, separate templates, state transitions, channel-level metrics | SMS adds policy and operational work; it is not a transparent fallback |
| Two email providers | A tested second route is required by the risk model | Two adapters, normalized events, routing rules, duplicate prevention | Every delivery behavior now needs two integration tests |

The four named API candidates belong in the first row until testing demonstrates otherwise. Amazon SES belongs in the cloud-service row. This is a field guide for measuring the work, not a ranking. **The winning integration is the smallest one whose failures your team can see and safely replay.**

## How should a startup compare a transactional email provider for welcome emails?

Very little about the user's outcome. A send request crossing an API boundary is one transition in a longer system: marketplace signup, verification token creation, message acceptance, mailbox delivery, link click, token validation, and account verification. Diagrammed in words: **signup event -> outbox -> provider adapter -> delivery event -> verification endpoint -> account state**. Each arrow needs a correlation key.

Acceptance is not delivery.

That distinction changes the comparison. A polished SDK may save an hour at the first arrow while an awkward event model consumes days at the fourth. Evaluate the whole path. Fast setup is useful, but only if the result is diagnosable.

Start with one invariant: one signup intent may create several delivery attempts, but it must produce one logical verification challenge. A timeout must not trick the worker into minting another active token and sending a second, conflicting link. Keep token lifecycle in the marketplace service. Let the messaging adapter transport a rendered message and report what happened to the attempt.

Consider the timeout path in full. The worker submits message `m-1042`, loses the response, and retries. The second request may represent another provider attempt even though the marketplace still has one logical message. A later delivery event can then arrive before the response to the retry. If the database has only a Boolean `sent` field, every observation fights for the same bit and the operator cannot tell delay from duplication. With separate message, attempt, and event records, the late evidence has somewhere honest to land. The exact provider vocabulary may vary; the marketplace invariant does not. This is the integration test that catches false simplicity.

One message. Several attempts.

For each of Postmark, Resend, Brevo, and Mailgun, run the same proof. Can the adapter attach your `messageId`? Can an event be mapped back to that ID without an address search? Can duplicate event delivery leave state unchanged? Can operators distinguish accepted, delivered, temporarily delayed, permanently failed, and user-verified? Record the answers from the candidate's current documentation and a sandbox run. A blank cell is a test result: the integration is not yet understood.

## Pick this when integration speed is the governing constraint

Pick one managed email API when the team owns a small service and wants a tight boundary. The application should see a generic `sendVerification` operation, never a provider SDK object. This keeps the initial change compact and makes later evaluation possible without rewriting signup logic.

Pick the existing cloud email service when the company has already solved the surrounding cloud work. Amazon SES documentation describes the service in terms of sending email and directs implementers through setup and sending concepts. That makes it a candidate, not an automatic low-effort answer. Count the identity, permission, event, and operational setup that your particular account still requires.

Pick email plus SMS recovery only when the product requirement justifies a second channel. CTIA publishes messaging interoperability and compliance best practices for SMS and MMS. Treat that material as part of the integration surface. Consent, message purpose, and channel state deserve explicit design; copying an email retry into an SMS send is not a sound recovery policy.

Pick two email providers only after writing the routing rule. "Use the backup" is not a rule. Define which observed state permits another route, how long the first attempt may remain unresolved, and how the system prevents two valid links from racing into the inbox. More routes can expand options, but they also expand test cases.

## Build one narrow contract and instrument it deeply

The application contract can stay small. The event vocabulary cannot. This TypeScript example separates a logical verification message from provider attempts, preserves a correlation identifier, and makes duplicate callback processing an expected case.

```ts
interface VerificationMessage {
  messageId: string;
  accountId: string;
  recipient: string;
  verificationUrl: string;
  expiresAt: string;
}

type AttemptState =
  | "accepted"
  | "delivered"
  | "delayed"
  | "failed";

interface DeliveryEvent {
  eventId: string;
  messageId: string;
  providerAttemptId: string;
  state: AttemptState;
  occurredAt: string;
  reasonCode?: string;
}

interface TransactionalEmailPort {
  sendVerification(message: VerificationMessage): Promise<{
    providerAttemptId: string;
    state: "accepted";
  }>;
}

interface DeliveryEventStore {
  has(eventId: string): Promise<boolean>;
  append(event: DeliveryEvent): Promise<void>;
}

async function recordDeliveryEvent(
  event: DeliveryEvent,
  store: DeliveryEventStore,
): Promise<void> {
  if (await store.has(event.eventId)) return;
  await store.append(event);
}
```

Keep the provider-to-domain translation at the edge. Verify an incoming event before normalization according to the chosen integration's documented mechanism. Then store the original receipt separately from the normalized event so an operator can investigate a mapping error without making provider payloads part of the business model.

The logs need 3 identifiers: logical `messageId`, provider attempt ID, and `accountId`. Do not log the verification URL or token. A useful structured log marks the transition, event time, normalized reason code, adapter name, and deployment version. The metric layer can then answer concrete questions: How many signup intents were created? How many attempts were accepted? How many reached a terminal delivery state? How many accounts were verified?

No token. Ever.

Watch the denominators. "Delivery rate" based only on accepted sends hides outbox failures. "Verification rate" based only on delivered events hides users whose event never arrived. Publish a small funnel with counts at every boundary, plus age buckets for signups that are still unverified. Alert on stuck state, not raw traffic. A quiet queue during a quiet hour is normal; a growing oldest-message age is actionable.

Here is the crisp before and after. Before: the signup handler calls an SDK, records `sent: true`, and support searches an address in a dashboard. After: the handler commits an outbox record, a worker calls the port, event ingestion advances an idempotent attempt, and support searches one correlation ID. The second design contains more pieces. It reduces mystery.

Test it as a state machine. Cover a send timeout followed by retry, the same event arriving twice, a delayed event arriving after a failure event, a click on an expired token, and an account already verified through another request. Then disconnect event ingestion in staging and confirm the oldest-unresolved-attempt alert fires. This exercise measures integration effort better than counting SDK lines.

For the candidate comparison, score the evidence your team produces rather than brochure claims. Use 4 columns: application code changed, operational resources created, failure cases passed, and provider-specific concepts exposed outside the adapter. Give each row a link to the documentation version and the date tested. Keep exact prices in a separate, time-stamped procurement sheet; volatile numbers should not decide the architecture. The trade-off is direct: a thinner first integration can expose more operational concepts later, while a larger initial boundary can reduce leakage into signup code.

## Limits worth accepting explicitly

The main limitation of this method is that a neutral adapter does not make providers identical. Event meanings, authentication procedures, account controls, and delivery feedback can differ. Normalize only the states the marketplace uses, retain the original evidence, and allow an `unknown` mapping to fail visibly during integration rather than guessing. It is not suitable as a substitute for candidate-specific documentation review or mailbox testing.

This method also cannot predict inbox placement from an API contract. It can prove that the marketplace owns its state transitions and can observe delivery evidence. Domain reputation, message content, recipient behavior, and mailbox handling still need ongoing measurement.

The practical stopping rule is concise: ship when one candidate passes the state-machine tests, every signup can be traced without exposing its token, unresolved attempts alert before they become support archaeology, and replacement is confined to the adapter plus event translator. Anything beyond that should answer a named risk.

## Sources

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
