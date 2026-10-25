# Dispatch Team Audio API: 4 Room Trust Boundaries for Ad Hoc Huddles

TL;DR: Treat each dispatch huddle as a server-owned lease, not a room that a browser may create or prolong. Issue separate, short-lived capabilities for audio and cursor presence; let the server decide membership; and close an empty room after a bounded grace period. That rule keeps a compromised client from turning a five-minute logistics standup into durable access.

| Approach | Pick this when | Trust boundary | Lifecycle burden |
|---|---|---|---|
| Browser mesh | The group is deliberately small and every participant may connect directly to every other participant | The control plane still authorizes membership; peers exchange media directly | The application owns signaling, peer churn, and cleanup |
| Hosted media router | Fast delivery and an external media processor are acceptable | The application grants a narrow room capability; the router handles media forwarding | The application still owns business state and expiry |
| Self-operated media router | Network placement or operational control outweighs the extra work | The application and media tier share an explicitly defined authorization contract | The team owns capacity, upgrades, telemetry, and incident response |

The transport choice matters, but the sharper decision is who may assert that a room exists. For a logistics team editing a dispatch plan, the answer should be the service that already knows the shift, depot, document, and roster. Audio and cursor presence are projections of that authority. They are not proof of it.

## What should a team huddle audio room API control?

Use a small application control plane for room state and an independent realtime data plane for packets. In words, the path is: authenticated dispatcher to session service; session service to a signed capability; browser to signaling and media; browser to cursor channel; every component back to metrics and structured logs.

This separation pays off during reconnects. A browser can lose its media connection while the huddle lease remains valid. It can also keep a network connection after its authorization has expired. Those are different states, so one `connected` flag cannot represent both.

That distinction is easy to miss.

Define four boundaries:

1. **Identity:** the ordinary application session establishes the person and organization.
2. **Membership:** the server checks that the person may join this depot, shift, and dispatch document.
3. **Capability:** a signed token grants a specific subject a specific action on one huddle until an explicit expiry.
4. **Presence:** heartbeats and disconnect events are hints used to maintain a view, never authorization evidence.

Keep audio publish, audio subscribe, and cursor publish distinct. A supervisor who listens should not automatically gain permission to move a cursor. A wall display may receive cursor updates without receiving a microphone grant. This is a little more policy work up front. It makes review and revocation legible later.

## Pick the media shape by failure domain

Choose browser mesh when the maximum participant count is intentionally low and a peer-to-peer topology fits the network. WebRTC defines peer connections and media behavior, but the application must still supply signaling; the specification does not prescribe a signaling transport. In mesh, each participant creates a connection to each other participant, so connection count and outbound media work rise as the group grows.

Choose a hosted router when the team wants the media plane operated outside its application boundary. The contract should remain narrow: opaque room identifier, participant identifier, allowed actions, and expiry. Do not put a depot's editable manifest or broad application role into a media token. The media service needs permission facts, not the whole business object.

Choose a self-operated router when data-path placement, inspection, or operational control justifies owning the runtime. This adds real duties: admission limits, regional capacity, key rotation, upgrades, packet-loss telemetry, and load tests that include mass reconnects. Control is useful. Pager ownership comes with it.

The trade-off is concrete. Mesh is not suitable for a huddle whose participant ceiling is unknown or routinely large, because every added peer changes the connection topology. A hosted router is a poor fit when policy forbids the required media path or the team cannot accept that external failure domain. A self-operated router is the wrong choice when nobody can own capacity planning and media incidents. In that case, operational control is a liability, not an advantage. These limitations should decide the media shape before API ergonomics do.

None of these choices should create the canonical huddle record from an unverified client message. The server creates the record first, then grants access to its data planes.

## How should a scoped session capability work?

Make creation idempotent for one dispatch document and one active shift window. The response should carry the server's room identifier and a capability with a short, explicit expiry. A retry after a timeout returns the same active lease rather than opening a second audio room.

Here is a focused TypeScript boundary. The signing and storage interfaces are deliberately generic. The important part is what gets checked and what gets omitted.

```ts
type JoinIntent = {
  actorId: string;
  organizationId: string;
  depotId: string;
  shiftId: string;
  dispatchDocumentId: string;
  requestId: string;
  wantsMicrophone: boolean;
};

type Capability = {
  subject: string;
  audience: "huddle-data-plane";
  roomId: string;
  actions: Array<"audio:publish" | "audio:subscribe" | "cursor:publish">;
  expiresAt: number;
  tokenId: string;
};

interface MembershipStore {
  mayJoin(intent: JoinIntent): Promise<boolean>;
}

interface RoomStore {
  getOrCreateActive(input: {
    organizationId: string;
    depotId: string;
    shiftId: string;
    dispatchDocumentId: string;
    idempotencyKey: string;
  }): Promise<{ roomId: string; leaseEndsAt: number }>;
}

interface Signer {
  sign(capability: Capability): Promise<string>;
}

async function issueJoinCapability(
  intent: JoinIntent,
  membership: MembershipStore,
  rooms: RoomStore,
  signer: Signer,
  now: () => number,
): Promise<{ roomId: string; token: string; expiresAt: number }> {
  if (!(await membership.mayJoin(intent))) {
    throw new Error("membership_denied");
  }

  const room = await rooms.getOrCreateActive({
    organizationId: intent.organizationId,
    depotId: intent.depotId,
    shiftId: intent.shiftId,
    dispatchDocumentId: intent.dispatchDocumentId,
    idempotencyKey: intent.requestId,
  });

  const issuedAt = now();
  const expiresAt = Math.min(issuedAt + 5 * 60_000, room.leaseEndsAt);
  const actions: Capability["actions"] = ["audio:subscribe", "cursor:publish"];
  if (intent.wantsMicrophone) actions.push("audio:publish");

  const token = await signer.sign({
    subject: intent.actorId,
    audience: "huddle-data-plane",
    roomId: room.roomId,
    actions,
    expiresAt,
    tokenId: crypto.randomUUID(),
  });

  return { roomId: room.roomId, token, expiresAt };
}
```

Five minutes is an example policy choice, not a universal optimum. Set the duration from the damage window you can accept and the refresh load you can operate. The hard rule is that refresh repeats membership evaluation. Possession of an old token must not be enough to mint a new one.

On the data plane, validate signature, audience, expiry, room, subject, and requested action. JSON Web Token defines registered claims including `aud`, `exp`, and `jti`; using those names does not remove the need to validate them. Keep server time synchronized, allow only a documented clock-skew tolerance, and emit the reason for every denial without logging the token itself.

Cursor messages deserve their own guardrails. Include a monotonically increasing client sequence number and document identifier, then discard stale or cross-document updates. Coalesce movement before broadcast. Cursor presence is ephemeral, so do not rebuild it from a durable event history after reconnect; send the current snapshot and accept new movement. The dispatch document itself needs a separate consistency model.

Do not blur them.

## Observe the lifecycle, not just the socket

A green connection counter can hide a broken huddle. Instrument state transitions: requested, authorized, joined, refreshed, disconnected, empty, and closed. Each event should include a correlation ID, room ID, organization ID, hashed or internal actor ID, transition reason, and timestamps. Exclude raw tokens and microphone payloads.

The useful metrics are ratios and distributions tied to user-visible stages. Track join authorization denials by reason, time from authorized to media established, refresh success rate, reconnect attempts per participant, active participants per room, rooms left empty past the grace interval, and close latency after lease expiry. WebRTC exposes connection statistics through `getStats()`; selected candidates, round-trip measurements, jitter, and packet loss can help separate access-control failures from media-path trouble.

Alert on sustained symptoms, not one disconnect. Mobile and warehouse networks move. A burst of reconnects across many rooms in one region is different from a single handset changing access points, and the alert dimensions should preserve that distinction.

Test the state machine with a fake clock. Cover duplicate create requests, token expiry during an active call, membership removal before refresh, simultaneous last-participant disconnects, process restarts, and a reconnect arriving during the empty-room grace period. Then load-test synchronized shift changes. Average traffic misses that edge.

## Limits and decision rule

This design does not choose codecs, solve NAT traversal, or make browser permission prompts predictable. `getUserMedia()` requires a secure context and user permission, while ICE may require STUN or TURN infrastructure to establish a path. Those concerns belong in the media plan and its operational budget.

The concise rule is: let business membership create the lease, let narrow capabilities open the data planes, and let server time end both. Pick mesh or a media router according to participant topology and the failure domain your team can own. Never let cursor presence or an open socket vote a room back into existence.

## References

- https://www.w3.org/TR/webrtc/
- https://www.rfc-editor.org/rfc/rfc7519
- https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia
- https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/getStats
- https://www.rfc-editor.org/rfc/rfc8445
