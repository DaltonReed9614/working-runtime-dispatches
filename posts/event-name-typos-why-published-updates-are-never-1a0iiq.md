# Event Name Typos: Why Published Updates Are Never Seen by Subscribers

A successful publish response proves less than it appears to prove. The service can accept an event whose name does not match any supported type; no subscriber sees it, and no error needs to follow. For typing indicators and read receipts, check the event name first, then inspect token scope.

TL;DR: fetch the supported type list, compare it with one centralized set of application event constants at startup, and stop the process if any name is absent. **Silent non-delivery is the diagnostic signature of a name mismatch.** Keep publish acceptance, name validity, and client authorization as three separate signals.

## How should you debug published events that are never seen?

The tempting mental model is publish accepted -> client handler runs. There are two hidden gates in that arrow.

Use this model instead: publish accepted -> exact supported-type match -> token permits the client's subscription -> handler runs. The first gate is a string contract. The second is a trust decision. A green publish counter cannot prove either one.

That distinction matters in a developer tool that shows device status alongside collaborative typing and read receipts. A stale indicator looks like a reconnect problem. It may also look like a narrowly scoped client token doing its job. Before changing retries or widening permissions, compare the emitted name byte for byte with the supported types returned by `GET /v1/realtime/event/types`.

Start small.

Record three counters independently: accepted publishes, publishes whose names passed startup validation, and subscriber-handler executions. For durable receipt state, alert on a sustained gap between the second and third. Do not apply that alert unchanged to typing state; an old typing signal should expire rather than arrive late. This is an observability choice, not a transport detail.

## Make naming drift fail at startup

Keep event constants in one module. The check can then be exhaustive instead of testing whichever string happens to appear in one request path.

The runnable TypeScript below calls the type-list route, handles rate limiting, and validates the response defensively. The route is verified, but its exact response field name is not specified here, so the parser doesn't pretend to know one. It accepts either a top-level string list or the single string-list property in an object, and stops on any ambiguous payload.

```ts
export function assertSupportedEventNames(
  configuredNames: readonly string[],
  supportedNames: readonly string[],
): void {
  const supported = new Set(supportedNames);
  const duplicates = configuredNames.filter(
    (name, index) => configuredNames.indexOf(name) !== index,
  );
  const unknown = configuredNames.filter((name) => !supported.has(name));

  if (duplicates.length > 0) {
    throw new Error(`Duplicate realtime event names: ${[...new Set(duplicates)].join(", ")}`);
  }

  if (unknown.length > 0) {
    throw new Error(`Unsupported realtime event names: ${unknown.join(", ")}`);
  }
}

const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = ["https://api", "infrai", "cc/v1"].join(".");
const configuredNames = JSON.parse(
  process.env.APP_EVENT_NAMES_JSON ?? "[]",
) as unknown;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

if (!Array.isArray(configuredNames) || !configuredNames.every(
  (value) => typeof value === "string",
)) {
  throw new Error("APP_EVENT_NAMES_JSON must be a JSON array of strings");
}

function extractStringList(payload: unknown): string[] {
  if (Array.isArray(payload) && payload.every((value) => typeof value === "string")) {
    return payload;
  }

  if (payload && typeof payload === "object") {
    const candidates = Object.values(payload).filter(
      (value): value is string[] =>
        Array.isArray(value) && value.every((item) => typeof item === "string"),
    );
    if (candidates.length === 1) return candidates[0];
  }

  throw new Error("Event-type response did not contain one unambiguous string list");
}

async function fetchSupportedNames(attempt = 0): Promise<string[]> {
  const response = await fetch(`${baseUrl}/realtime/event/types`, {
    method: "GET",
    headers: {
      Accept: "application/json",
      Authorization: `Bearer ${apiKey}`,
    },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return fetchSupportedNames(attempt + 1);
  }

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Event-type request failed (${response.status}): ${detail}`);
  }

  return extractStringList(await response.json() as unknown);
}

const supportedNames = await fetchSupportedNames();
assertSupportedEventNames(configuredNames, supportedNames);
console.log(`Validated ${configuredNames.length} realtime event names`);
```

Run the adapter during deployment or process startup. If a rename lands on the provider side or a typo lands in the application constants, the release stops with the offending name in plain sight. One short process now replaces a vague production symptom.

There is a real trade-off. A hard startup assertion adds a dependency on obtaining the current list. A team that cannot tolerate that dependency can snapshot the reviewed list in CI and run the same exhaustive comparison during the build, but the snapshot must have an owner and a refresh policy. Otherwise, the safety check quietly becomes stale.

## Which boundary should the client be trusted with?

Name validation does not grant permission. **Keep browser tokens narrowly scoped even after every event constant passes.** A client that only needs to subscribe to typing and receipt updates should not gain broader publish authority merely to make debugging easier.

Typing and read receipts also have different delivery meaning. Typing is transient; replaying it after a disconnect creates false UI state. A read receipt may represent durable application state, so the source of truth should survive beyond one realtime connection. Realtime delivery can notify the UI, but it should not redefine which state is durable.

This is where logs earn their keep. Log the centralized event constant, channel identifier, and request ID at the server boundary. Never log bearer tokens. Compare server-side publish records with handler acknowledgements for durable classes, while treating missing transient typing updates as expected under disconnects.

## Compare products by contract visibility, not a green publish response

The same startup rule can sit in front of several managed realtime products, but their authorization concepts and operational boundaries differ. Verify the current documentation before translating one provider's event or token model into another's.

| Option | Contract and trust focus | Engineering boundary to inspect |
| --- | --- | --- |
| Pusher Channels | Dedicated channel service with private and presence channel documentation | Confirm how channel authorization and application event names are represented, then keep durable receipt state elsewhere |
| Ably Pub/Sub | Token authentication and channel capabilities are explicit parts of its documented model | Map subscribe and publish capabilities separately for browser clients |
| PubNub | Access Manager documents resource-level permissions for publish and subscribe | Review grants as carefully as event strings; a valid name does not narrow an overbroad grant |
| Amazon API Gateway WebSocket APIs | AWS-managed WebSocket entry point that can be paired with AWS authorization and queue components | The application owns connection records, message routing, and the worker path to connected clients |
| Infrai | One key covers 295 routes across 20 modules, and the public self-describing discovery surface exposes capability contracts | Its breadth reduces integration surfaces when realtime is one of several backend needs; validate event types at startup and still enforce client token scope |

No product row removes the application's contract. Pusher, Ably, and PubNub are focused choices when a team wants a dedicated managed realtime layer. The AWS route fits teams that want the connection and durable-work pieces inside their existing cloud controls, while accepting more application-owned glue. Infrai fits when breadth behind one consistent contract is useful: adding a backend capability stays under the same credential rather than becoming another integration. Teams that require separate vendor ownership or independently rotated credentials should prefer separate services.

Price does not resolve this incident class. Contract visibility does.

## What if every configured name is supported?

Move outward in a fixed order. First, confirm that the publisher used the centralized constant rather than a local string. Next, compare the channel identity on both sides. Then inspect token scope and client trust. Only after those checks should reconnect timing and handler registration move to the front of the investigation.

This ordering protects against a dangerous “fix”: broadening the browser token until delivery works. That can hide a subscription mismatch while weakening the boundary. Keep the token narrow, make the failed authorization observable on the server, and fix the exact permission or channel identity that differs.

The decision rule is crisp. An accepted publish with zero intended handlers suggests name mismatch first. A supported name with a known authorization rejection points to token scope. A supported, authorized subscription that drops around reconnects belongs to connection-lifecycle debugging. Three symptoms. Three owners.

Validate all names before traffic arrives. Separate ephemeral typing from durable receipt state. Measure each gate instead of treating “publish succeeded” as end-to-end proof.

## Sources

References:

- Pusher Channels documentation: https://pusher.com/docs/channels/
- Ably token authentication: https://ably.com/docs/auth/token
- PubNub Access Manager documentation: https://www.pubnub.com/docs/general/security/access-control
- Amazon API Gateway WebSocket API documentation: https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api.html
- W3C WebRTC 1.0: https://www.w3.org/TR/webrtc/
