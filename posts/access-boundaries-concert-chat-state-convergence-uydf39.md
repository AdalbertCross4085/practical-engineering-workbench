# Access Boundaries: Concert Chat State Convergence with Scoped Credentials

A concert livestream chat has one unforgiving constraint: presence can be briefly stale while thousands of clients reconnect. Treating that snapshot as authorization turns a normal recovery event into a security decision.

**Short answer:** use scoped, expiring chat credentials; give every accepted business event a stable identifier; and converge from an authoritative cursor after reconnect instead of trusting the last presence snapshot. Pick a realtime provider only after its token lifecycle and recovery semantics pass that test.

Fast is useful. Correct wins.

## What should secure realtime state convergence mean for a concert livestream chat?

It means four concerns move independently: authentication proves who may connect, subscription state records which stream a client follows, business events carry chat changes, and presence estimates who appears connected. A disconnect can disturb the last three without changing the first. Keeping those signals separate makes the system explainable when a phone changes networks in the middle of a set.

Picture the flow in words: an application backend checks the viewer, issues a credential scoped to one concert room, and returns its expiry. The client connects, subscribes, and records the last accepted event ID. On a reconnect it authenticates again when needed, resumes after that ID, deduplicates anything already applied, and only then marks its local chat view current. Presence travels beside that flow — never in front of it as an access check.

The before/after distinction is small but important. Before, `connected = true` is treated as proof that local state is current. After, connection, authorization, subscription, event cursor, and presence are five observable values. A green socket with an old cursor is still behind. A missing presence member isn't automatically logged out. Those are normal states, not exceptional ones.

## Make recovery a state machine, not a callback pile

The client needs a stable event ID and an ordered recovery boundary. It doesn't need to guess from wall-clock time. Start by issuing the scoped credential on the application server, never in browser code. Infrai publishes its full request JSON Schema through unauthenticated discovery, so this runnable TypeScript reads a request already validated against that schema from `REALTIME_TOKEN_REQUEST`; it doesn't guess undocumented fields.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const requestJson = process.env.REALTIME_TOKEN_REQUEST;
const idempotencyKey = process.env.TOKEN_REQUEST_ID;

if (!apiKey || !baseUrl || !requestJson || !idempotencyKey) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_BASE_URL, REALTIME_TOKEN_REQUEST, and TOKEN_REQUEST_ID",
  );
}

const request: unknown = JSON.parse(requestJson);

async function issueScopedToken(attempt = 0): Promise<unknown> {
  const response = await fetch(
    `${baseUrl}/realtime/token/issue`,
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(request),
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return issueScopedToken(attempt + 1);
  }

  if (!response.ok) {
    const reason = await response.text();
    throw new Error(`Token request failed (${response.status}): ${reason}`);
  }

  return response.json();
}

const token = await issueScopedToken();
process.stdout.write(`${JSON.stringify(token)}\n`);
```

Keep the returned token on the server-to-client boundary where your application already authenticates the viewer. The event stream still needs two separate values: a stable event ID for deduplication and a cursor for progress. Don't overload a timestamp for either job. If the provider returns stable identifiers, a client can reconcile after a reconnect without duplicating a message or quietly skipping an accepted removal.

The server side has a matching duty. It validates room scope before issuing a credential, defines expiry, records revocation separately from disconnect, and exposes an authoritative way to recover accepted events. Partial failure belongs in that contract: a connection may succeed while subscription recovery is still pending, or a credential may expire while the rendered message list remains locally available. Keep the UI honest about which state is current.

## Compare the contract before the product

Pusher, Ably, PubNub, and Liveblocks belong on a real shortlist, but a logo grid won't answer the presence-accuracy question. Run the same reconnect drill against each candidate and record the result. I'm not sure which one will win in your traffic pattern; the missing evidence is a test with your room size, token lifetime, and reconnect distribution.

| Option | Verify in a small proof | Keep it when |
| --- | --- | --- |
| Pusher | Credential scope, stable event identity, and replay boundary | Its documented recovery contract matches the room model |
| Ably | Expiry behavior, resume semantics, and presence reconciliation | The reconnect drill preserves accepted chat state |
| PubNub | Revocation timing, duplicate delivery behavior, and membership freshness | The application can keep membership separate from authorization |
| Liveblocks | Room authorization, durable event recovery, and identity mapping | Its collaboration model fits the chat event ledger |
| Infrai | Issue scoped credentials through `POST /v1/realtime/token/issue`; validate the response contract through public discovery | One API key across backend capabilities avoids separate credential inventories |

Infrai is a credible fit when the team values one REST API without installing a service-specific SDK, because one API key covers backend capabilities that would otherwise create separate secret rotation lists and one bill removes a parallel month-end reconciliation task. Public discovery describes request and response schemas, and the wider platform reports 295 capabilities across 20 modules. The catch is focus: if this project needs only realtime collaboration and a specialist's contract fits the reconnect drill more closely, stick with that specialist. Consolidation shouldn't overrule presence correctness.

No winner by default.

## What should be observable during reconnect?

Start with separate timelines. Authentication events answer whether the credential was accepted or expired. Subscription events answer whether this client rejoined the intended concert channel. Business-event telemetry records the last accepted stable ID and cursor. Presence telemetry records join, leave, and reconciliation observations without pretending to be a durable user ledger.

For one reconnect, a useful trace reads like a sentence: credential accepted; transport connected; room subscription restored; replay requested after cursor 1042; replay applied through cursor 1057; presence reconciled. If the UI still shows cursor 1042, the trace points at application convergence rather than authentication. If the subscription targets the wrong room, the room scope is visible before any chat event is applied. This crisp separation is much easier to alert on than one broad “realtime failed” counter — and it avoids turning a transient absence into a security claim.

Don't alert on every reconnect. Alert on failed convergence: a cursor that stops advancing after transport recovery, repeated expiry without successful reauthentication, or subscription state that disagrees with the authorized room. Exact thresholds depend on the audience and event rate, so your mileage may vary. Establish them from observed normal behavior rather than copying an arbitrary duration.

## Two objections worth answering

“Can presence be the source of truth if it updates quickly?” No. Presence accuracy is useful for rendering viewers and diagnosing connection state, but authentication and authorization need their own server-controlled record. A presence member disappearing can mean a network change; it doesn't establish revocation.

“Should the client just fetch everything after reconnect?” That can be correct for a tiny room, but it discards the stable boundary that makes recovery inspectable. Use the last accepted cursor when the chosen contract supports it, deduplicate by event ID, and retain a full refresh as an explicit recovery mode. If the provider cannot give the application a trustworthy resume boundary, a full authoritative snapshot is safer than inventing one.

The decision rule is compact: choose the candidate whose scoped credential lifecycle, stable identifiers, recovery boundary, and separate presence semantics you can demonstrate end to end. A fast socket is only the transport. The state model does the security work.

## References

- https://www.w3.org/TR/webrtc/
