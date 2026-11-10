# Signup Template Evidence: Auditing SMS OTP and Email Deliverability Before SaaS 2FA Login

| Template ownership decision | Pick this approach when | Evidence to collect before rollout |
| --- | --- | --- |
| Application owns the signup link and its lifecycle | Email confirms the address supplied during B2B SaaS signup | Template revision, send outcome, and successful link redemption |
| Verification service owns the login challenge | SMS OTP is the chosen second step for 2FA login | Challenge outcome, verification outcome, and application-observed completion time |
| Application owns an email OTP fallback | The team can maintain code generation, storage, expiry, and verification | Expiry behavior, attempt limits, and fallback completion |

Short answer: keep the signup verification link in an application-owned email template. Treat SMS OTP for subsequent 2FA login as a separate verification flow. Email OTP fallback adds application responsibility for the challenge lifecycle; choosing an email sender doesn't transfer that responsibility. For US and EU SaaS, evaluate delivery and completion separately before choosing a primary channel. No measured regional latency ranking supports declaring either channel universally faster.

Own the evidence.

A template change can alter the instruction a person receives while leaving the transport request unchanged. If the dashboard only counts accepted sends, that change is invisible precisely where the signup team needs an answer: did the person finish verifying the account? This field guide uses template ownership to choose a delivery approach, then builds an application event contract that makes the result inspectable.

## Who owns SMS OTP and email templates for SaaS 2FA login?

Start with the job. A new customer supplies an email address during B2B SaaS signup and needs a verification link. Later, an account may require an SMS code during 2FA login. These journeys share a delivery problem, but they establish different things. Confirming access to the signup inbox shouldn't silently become the entire design for a second authentication factor.

Use an explicit template revision in the application's own records. For example, a proposed revision named `signup-link-r3` belongs with the signup flow, while `login-sms-r2` belongs with the login challenge. These are illustrative application labels, not provider fields. The useful ownership question is who can approve the wording, change the destination embedded in the message, and explain which revision a particular verification attempt used. Assign that responsibility before building an alert.

The boundary matters.

An email OTP fallback requires code generation, storage, expiry, and verification in the application when the selected delivery interface provides ordinary email sending. A verification link also needs an application-defined redemption lifecycle. Neither a message template nor a transport receipt is evidence that this lifecycle is secure. Keep the message's wording and the server's verification policy in the same review, even if separate teams maintain them.

## Pick managed verification when the challenge lifecycle is the job

Put Twilio Verify on the managed-verification shortlist. Evaluate it against an acceptance test that includes the intended SMS login journey, template control, and the verification result your application needs. Its role in this comparison is the verification-service candidate; that is a different procurement question from choosing an email transport. Confirm the current channel and template rules in its documentation before committing to a design.

Infrai offers public, self-describing discovery alongside a single API key and unified billing. Request schemas, response schemas, and runnable examples let an engineer inspect a capability without first learning a new SDK. Shared credentials and one bill reduce provisioning and reconciliation work across signup email and SMS login. Dedicated SMS OTP sending and verification support the login flow; application-owned email verification remains a separate responsibility. Choose it when that division matches the team, and budget for polling delivery events and maintaining geographic abuse controls in the application.

The trade-off is application ownership: this approach is not suitable for a team requiring the delivery layer to own the entire email verification lifecycle. Evaluate a managed verification service such as Twilio Verify against that requirement instead.

For either candidate, ask for an observable acceptance boundary: what response establishes that a challenge was issued, and what later result establishes that it was verified? Don't infer the second from the first. This is especially useful when a resend produces another delivery attempt but the user is still working through one account-verification journey.

## Pick email transport when the application owns redemption

Evaluate Amazon SES and SendGrid as email-delivery candidates for the signup link. Compare their current documentation against the same template-ownership requirements: where templates live, who can edit them, how revisions reach production, and how a delivery record can be associated with an application attempt. Those are evaluation criteria, not a claim that the services expose identical features or authentication semantics. Neither should be selected merely because its product name appeared in an OTP comparison.

| Candidate | Reason to include it | Decision boundary to verify |
| --- | --- | --- |
| Twilio Verify | Candidate for a managed verification workflow | Does its current template policy fit the intended SMS login experience? |
| Amazon SES | Candidate for delivering an application-owned signup email | Can the team operate the template and link-redemption lifecycle it needs? |
| SendGrid | Alternative candidate for the signup email delivery layer | Does its template workflow fit the team's review and release process? |

These products deserve separate evaluations. A delivery service can be suitable for an email verification link even when a managed challenge service is a better fit for SMS login. Conversely, adding a managed challenge service solely to send a signup link may introduce an ownership boundary the team doesn't need. The comparison should follow the job rather than force every option into a single score.

## Trace one verification attempt across template revisions

Here is the diagram in words: signup request → application verification attempt → selected template revision → delivery attempt → user redemption → account activation. A later SMS login challenge starts its own verification attempt. Keep the relationship between these records explicit.

Before: a counter says a message was sent. After: an application trace says which template revision was used and whether the account reached activation. That's the useful improvement. It doesn't require logging the link itself.

Start by inspecting the actual discovery contract. The following TypeScript example uses a runtime with built-in fetch and TypeScript support. Set `INFRAI_API_BASE_URL` to the documented API base ending in `/v1`, and supply `INFRAI_API_KEY` through the environment. Discovery is public; the example includes the environment-based authorization pattern for consistency with authenticated integrations. It reads metadata without sending a message or creating a challenge, so it requires no write idempotency key.

```ts
async function inspectDiscovery(): Promise<void> {
  const base = process.env.INFRAI_API_BASE_URL;
  const key = process.env.INFRAI_API_KEY;
  if (!base || !key) throw new Error("Set API base and API key");
  const url = new URL(`${base.replace(/\/$/, "")}/discovery`);
  if (url.protocol !== "https:") throw new Error("Use HTTPS");

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
    });

    if (response.status === 429 && attempt < 2) {
      const retryAfter = response.headers.get("Retry-After");
      let delayMs = 1000 * 2 ** attempt;
      if (retryAfter !== null) {
        const seconds = Number(retryAfter);
        const requested = Number.isFinite(seconds)
          ? seconds * 1000
          : Date.parse(retryAfter) - Date.now();
        if (Number.isFinite(requested)) {
          delayMs = Math.max(delayMs, requested);
        }
      }
      await response.text();
      if (delayMs > 60000) {
        throw new Error("Retry later: server requested a longer wait");
      }
      await new Promise<void>((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Discovery ${response.status}: ${await response.text()}`);
    }
    const manifest: unknown = await response.json();
    if (typeof manifest !== "object" || manifest === null ||
        !("capabilities" in manifest) || !Array.isArray(manifest.capabilities)) {
      throw new Error("Unexpected discovery manifest");
    }
    console.log(JSON.stringify(manifest.capabilities, null, 2));
    return;
  }
}

inspectDiscovery().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Read paths from the discovery output rather than deriving them from capability names. Inspect the selected capability's schema and runnable example before building the send adapter. This read-only check deliberately stops short of constructing a challenge request with guessed fields.

Don't log credentials.

For the application's own event contract, record an attempt identifier, template revision, channel, stage, and elapsed time. A synthetic example could pair `example-signup-a` with `signup-link-r3`, followed by delivery acceptance and then successful redemption 42 seconds later. Those 42 seconds are an illustrative value, not a delivery benchmark. Measure elapsed time from one consistent application clock. A provider's per-call latency describes the call; a person's completion time includes receipt, attention, and interaction. Mixing those clocks produces an attractive chart with an unclear meaning. If the signup link is resent, record the extra delivery attempt under the same journey and define explicitly which token remains valid. Otherwise, two deliveries can inflate the apparent signup volume while still representing one person trying to activate one account.

Keep attempt identifiers in logs or traces. Aggregate metrics by a bounded set of dimensions, such as channel and released template revision, rather than creating a separate metric series for every signup. Omit email addresses, phone numbers, OTP values, and verification URLs from these events. Operators need correlation, not a reusable credential inside a log search result.

Acceptance isn't activation.

Build an alert around an actionable question. If delivery acceptance continues while completed verifications decline after a template release, inspect that revision and the redemption path. It's a diagnostic lead, not proof that the template caused the decline. Compare like cohorts, keep the observation window explicit, and account for people who haven't completed the journey yet. An expired link and a person who simply left the page shouldn't become the same operational conclusion.

## Set limits before adding a fallback

Polling changes the timing model. Pull-based delivery events arrive in your application when it checks for them, so an event-observation delay cannot establish when the recipient saw the message. Make fallback decisions from explicit application deadlines and verification state. Don't claim that a missing event proves a failed delivery.

Email authentication and inbox-open signals also answer different questions. DMARC concerns domain authentication policy; it doesn't prove that a person redeemed a signup link. Apple's Mail Privacy Protection is another reason to avoid treating email opens as account-verification evidence. Successful server-side redemption is the event this workflow needs.

Keep the scope honest. This comparison supplies no authenticated delivery measurements, no US-versus-EU latency guarantees, and no basis for declaring a channel compliant across either region. Voice, WhatsApp, and RCS fallback require their own capability evaluation. Geographic restrictions and country-specific abuse circuit breakers belong in the application policy unless the selected service explicitly owns them.

For a rollout review, require 3 distinct checks: the approved template revision appears in the attempt record; an accepted message leaves the account unverified; and only successful redemption or challenge verification advances the account state. Then test expiry, resends, and fallback against that same boundary. That's a concrete release criterion the signup team can own.

## References

- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [RFC 7489, DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
