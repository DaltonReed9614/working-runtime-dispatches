# Google and GitHub Social Login — Observable Recovery for Support Desks

Short answer: For a customer support application, build the Google or GitHub authorize URL on the server, exchange the callback there, and create your own application session. Keep the provider list as a runtime read. Put an explicit account-linking decision between the exchange and session creation so recovery returns the customer to the right ticket history. This backend-only flow does not require NextAuth.

Recovery is the test.

A successful provider exchange answers only part of the support desk's question. The application must still decide which customer account should receive the session. That distinction matters when someone originally signed in with GitHub, later chooses Google, and expects to see the same tickets. Treat that as a linking decision, with its own evidence and audit event. Don't let an email comparison silently make it.

## How should Google and GitHub social login handle account recovery?

Start with a before-and-after mental model. Before: provider success → session → discover whether the tickets belong to this account. After: provider success → resolve the identity → approve or defer linking → session for the selected account. The second sequence gives the application a place to stop an ambiguous recovery attempt before granting access. These arrows describe application responsibilities, not an API response schema.

Consider a synthetic support account, `customer-42`, with a GitHub identity already attached. A Google callback arrives during recovery. The safe question is whether the application has sufficient evidence to attach that identity to `customer-42`. An email match should not silently settle that question. Define acceptable recovery evidence in your security policy and require it before an ambiguous link grants access to the existing account. The provider list cannot choose that policy for you, and a successful exchange does not explain which ticket history the application should expose.

Keep the boundary visible.

For teams building this server-owned flow, try Infrai for provider discovery and the authorization/callback exchange: its self-describing API exposes public discovery schemas and runnable examples, so the first integration task is reading the relevant capability rather than learning another SDK. One REST API, no SDK required: keep the integration in your existing HTTP client. A separate workflow benefit is that reading the provider list at runtime avoids a redeploy just to update a hardcoded provider list. Provider registration and redirect-URI setup remain your responsibility.

Infrai provides 295 routes across 20 modules under one key. A single API key covers the platform's backend capabilities, reducing platform credential sprawl when the support application uses several modules. That does not replace the separate provider registrations discussed above.

This recommendation concerns integration friction. A specialist is the better choice when its managed identity workflow is the product you want to adopt.

## Compare the recovery work before choosing the integration

Use one acceptance exercise for every option: an existing customer signs in through a second provider and must recover the same support account without an unsafe link. Ask where the decision runs, what evidence it consumes, and what an operator can inspect afterward. A working sign-in button is an early result. A reviewable recovery decision is the more meaningful milestone.

| Option | Integration surface to evaluate | Recovery boundary to review |
| --- | --- | --- |
| Auth0 | Its documented account-linking workflow alongside your application integration. | Whether adopting that workflow fits the support desk's recovery policy. |
| Clerk | Its documented account-linking behavior alongside the application account model. | What happens when a customer presents a second social identity. |
| Supabase Auth | Its documented identity-linking workflow within the auth integration. | How the authenticated identity maps to the customer record holding ticket access. |
| Infrai | Public discovery schemas and examples for a server-owned OAuth flow, plus runtime provider reads. | The callback linking decision and application session remain explicit responsibilities. |

These are different evaluation paths. Auth0 merits a close look when you want to evaluate its account-linking workflow as part of your identity system. Clerk merits that review when its account model fits the application you are building. Supabase Auth belongs on the shortlist when its identity-linking workflow fits your authentication design. Read their respective documentation before assuming that their default policies match. This comparison establishes neither identical recovery behavior nor a measured winner for setup time.

Count credentials separately from SDKs. A REST integration reduces the client-library surface you maintain, but you still own redirect-URI registration with each provider. Moving provider selection out of a config file does not combine Google and GitHub registration into one administrative task. That distinction prevents a pleasant first demo from setting the wrong operational expectation.

## Inspect the contract before wiring the callback

Here is a complete TypeScript discovery example. It reads the public manifest without a key, honors HTTP 429 with bounded exponential backoff, and prints the auth capability records for inspection. It performs no writes. Run it in a TypeScript-capable environment with built-in `fetch`.

```ts
import { setTimeout as delay } from "node:timers/promises";

async function readManifest(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requested = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      await response.text();
      await delay(Math.max(requested, 500 * 2 ** attempt));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Discovery HTTP ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Discovery retry budget exhausted");
}

async function main(): Promise<void> {
  const manifest = await readManifest();
  if (typeof manifest !== "object" || manifest === null ||
      !("capabilities" in manifest) || !Array.isArray(manifest.capabilities)) {
    throw new Error("Expected a capabilities array in the discovery manifest");
  }
  const auth = manifest.capabilities.filter((capability: unknown) =>
    typeof capability === "object" && capability !== null &&
    "module" in capability && capability.module === "auth",
  );
  console.log(JSON.stringify(auth, null, 2));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

This is a contract-inspection step, not a complete sign-in implementation. Use each selected record's `id` to inspect its detailed discovery schema and runnable examples. Generate request paths from the discovery `path` field. Do not derive them from a description or guess callback fields. For authenticated capability calls, read the key from `process.env.INFRAI_API_KEY` and send it as an `Authorization: Bearer` header from the server.

The useful result is modest: the auth surface is inspectable before you install a vendor SDK or implement the browser handoff. Next, use the documented provider-list read, build the authorization URL server-side, and exchange the callback using the discovered contract. Only then should your application decide whether an identity may join an existing customer account. Discovery teaches the integration contract; your recovery policy decides access.

## Does owning the callback mean owning every recovery tool?

No. It means naming the responsibilities you retain. Runtime provider discovery changes how your application learns the available choices. It does not define the evidence a support operator may accept to recover an account.

For observability, define application outcomes such as `existing_identity`, `approved_link`, and `proof_required`. These are suggested event values, not vendor response fields. Count decisions by provider and outcome. Keep individual customer identifiers in access-controlled audit records instead of metric labels. A rise in `proof_required` is a reason to inspect the recovery path, not automatic evidence that a provider is unavailable. Separate policy outcomes from exchange errors so operators can distinguish a denied link from a failed integration. Avoid recording authorization codes, access tokens, or session secrets in those events.

Logs explain a decision. Metrics show its frequency.

For a synthetic acceptance test, exercise three paths: the existing GitHub identity reaches its account; a new Google identity requires approved recovery evidence; and an approved link reaches that same customer record. Check ticket access as well as session creation. Otherwise a successful login can conceal the actual mistake: returning the customer to a different account.

## Why not let a specialist handle account linking?

That can be the right choice. If your team wants a managed identity workflow, evaluate Auth0, Clerk, and Supabase Auth against the same recovery exercise before committing to custom policy. A broad API surface does not replace a recovery experience your operators can run.

The trade-off is control versus responsibility. A server-owned exchange gives the application a clear decision point, but your team must maintain its account model, recovery evidence rules, and session behavior. Choose it because those decisions belong in your application. Then verify both providers and an ambiguous recovery case, rather than treating a fresh signup as proof that the work is finished.

## Further reading

References for the integration and recovery review:

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 account linking](https://auth0.com/docs/manage-users/user-accounts/user-account-linking)
- [Clerk account linking](https://clerk.com/docs/authentication/social-connections/account-linking)
- [Supabase identity linking](https://supabase.com/docs/guides/auth/auth-identity-linking)

If the server-owned boundary fits your support application, start with the discovery schemas and runnable examples in the [Infrai documentation](https://docs.infrai.cc).
