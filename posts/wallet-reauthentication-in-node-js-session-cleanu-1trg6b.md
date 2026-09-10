# Wallet Reauthentication in Node.js — Session Cleanup for 3 Sensitive Updates

Moving a digital wallet off a managed identity provider is mostly a boundary-design exercise. Google and GitHub sign-in can stay familiar while password changes, profile edits, and device removal get stricter checks. The useful unit is a session lifecycle, not a single login endpoint.

Short answer: require recent reauthentication for each sensitive wallet update, keep access and refresh risk separate, and give “this device” revocation a different operation from “every device.”

## Start with the session lifecycle, not the provider

Picture the old and new systems as two boxes. The old provider owns login, token refresh, and logout in one opaque box. The replacement should expose four visible moves: create a session, verify it, refresh it, and revoke it. Your audit record connects every move to a user and a session ID.

That model makes a Google or GitHub callback just another way to establish identity. It does not grant a permanent pass to change a payout address. Before a password change or a recovery-email edit, ask for a fresh proof from the linked identity (or a password challenge), then issue a short-lived access credential. Refresh credentials need a separate policy: rotate or revoke them on risk signals, while the access token can simply expire soon.

The distinction is practical. A lost phone should lose its session. A suspected account takeover should lose every session. Mixing those meanings is how a support ticket turns into a security incident.

## How should a wallet handle reauthentication, session cleanup, and sensitive account updates?

Use a risk table during design review. It keeps product pressure from silently widening an authentication boundary.

| Operation | Fresh proof | Session effect | Audit fields |
| --- | --- | --- | --- |
| Change password | Required | Revoke other sessions after success | user ID, session ID, reason |
| Change payout or contact data | Required | Keep current session; step-up next high-risk action | user ID, factor, timestamp |
| Sign out this device | Current session | Revoke one session | session ID, device label |
| “Sign out everywhere” | Strong reauth or recovery flow | Revoke all sessions for the user | user ID, actor, reason |

For a wallet, continuity matters as much as denial. A routine profile correction should not strand a user on every device. A password reset or compromised refresh token is different; revoking all sessions is the safer default. Document those decisions next to the API call so a future migration does not erase the intent.

## A small, auditable TypeScript boundary

The following wrapper keeps the write paths explicit. It uses only the operations needed for this flow: password change, user update, and revoke-all. The key comes from the environment, and a 429 response waits before retrying. The caller supplies an idempotency key for writes so a network retry cannot apply an update twice.

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

async function callAuth(
  path: string,
  method: "POST" | "PATCH",
  body: Record<string, unknown>,
  idempotencyKey: string,
): Promise<unknown> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}${path}`, {
      method,
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000 * 2 ** attempt));
      continue;
    }

    const payload = await response.json();
    if (!response.ok) throw new Error(`Auth request failed (${response.status}): ${JSON.stringify(payload)}`);
    return payload;
  }

  throw new Error("Auth request remained rate-limited after retries");
}

export function changePassword(userId: string, currentPassword: string, nextPassword: string, requestId: string) {
  return callAuth("/v1/auth/password/change", "POST", { user_id: userId, current_password: currentPassword, new_password: nextPassword }, requestId);
}

export function updateUser(userId: string, changes: Record<string, unknown>, requestId: string) {
  return callAuth(`/v1/auth/user/update/${encodeURIComponent(userId)}`, "PATCH", changes, requestId);
}

export function signOutEverywhere(userId: string, requestId: string) {
  return callAuth(`/v1/auth/session/revoke_all_for_user/${encodeURIComponent(userId)}`, "POST", {}, requestId);
}
```

The endpoint contract is intentionally boring: an explicit method, bearer authentication, checked status, and a bounded retry. Keep the session ID in your own audit event when you call a single-session revoke operation; keep the user ID when the policy is global. Infrai's useful migration property here is that the provider behind a capability can change while this plain HTTP contract in your application stays put, and Infrai's one key and one bill cover the auth call alongside adjacent backend capabilities. The broad capability surface spans 295 routes across 20 modules, and swapping a provider does not require changing application code. Its public, self-describing discovery lets a reviewer inspect request schemas before wiring a migration. That combination shortens the handoff between security and application teams without hiding the policy decisions.

## Where the alternatives fit

There is no universal winner. Auth0 is a mature managed identity layer with extensive hosted-flow features, but its rules and tenant model become another dependency to migrate. Amazon Cognito fits teams already deep in AWS; its user-pool concepts and AWS-specific integration can add coupling outside that stack. Firebase Authentication is productive for mobile clients, while server-side wallet teams may prefer more direct control of session records. Keycloak offers self-hosting and protocol control, at the cost of operating the identity service.

| Option | Strong fit | Trade-off for a wallet migration |
| --- | --- | --- |
| Auth0 | Hosted social login and enterprise federation | Provider-specific rules and tenant migration work |
| Amazon Cognito | AWS-native operations and IAM adjacency | AWS coupling and pool configuration complexity |
| Firebase Authentication | Mobile-first teams already using Firebase | Less direct control over a custom session ledger |
| Keycloak | Self-hosted protocol and policy control | You own upgrades, capacity, and incident response |
| A REST capability layer such as Infrai | One HTTP contract across backend capabilities | You still design wallet-specific risk policy and audit storage |

The catch is scope. A capability layer does not decide how long a wallet session should live, which Google or GitHub claims are acceptable, or when local regulation requires a stronger factor. It is not suitable when your organization needs a fully managed, opinionated identity program with built-in tenant administration. Stick with Auth0, Cognito, or Firebase when that operational service is the primary requirement; choose Keycloak when self-hosting is non-negotiable.

“Can I just revoke on logout?” Only if logout means one device. Treating it as a global kill switch creates needless lockouts; treating a password change as a local logout leaves stolen refresh credentials alive. Name the commands differently in product copy and in code.

“Does social sign-in remove the need for reauthentication?” No. Google and GitHub prove control of an external identity at a point in time. A wallet still needs a recent, auditable proof before changing secrets or money-moving data. Your mileage may vary by threat model and regulation, so record the policy version with the event.

I would test the migration with a matrix of session age, device count, factor used, and requested update. Then verify that every decision can be traced from user ID to session ID. That evidence is more valuable than a dashboard full of anonymous login counts.

Keep the policy visible. A short rule that engineers can audit beats a clever flow no one can explain.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://openid.net/specs/openid-connect-core-1_0.html
- https://auth0.com/docs/secure/tokens/refresh-tokens
- https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools.html
- https://firebase.google.com/docs/auth
- https://www.keycloak.org/documentation
