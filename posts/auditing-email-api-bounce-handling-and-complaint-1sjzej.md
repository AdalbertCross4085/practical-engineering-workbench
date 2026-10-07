# Auditing Email API Bounce Handling and Complaint Suppression (For Contact Routing)

Short answer: choose an email API by the evidence you can retrieve, normalize, and retain after a customer submits a support form. If inbound callbacks are off-limits, a scheduled poller can collect bounce and complaint events, advance a durable cursor, and suppress unsafe destinations before the next send. The useful comparison is not a feature checklist. It is whether the resulting record can prove which message was attempted, what happened to it, when the system learned that fact, and why later mail was allowed or blocked.

For customer support, that evidence changes routing. A contact form may initially select the billing queue, yet a hard bounce from its queue address should produce a different operational action than a delayed message. A complaint should prevent another nonessential message to the same recipient. Delivery monitoring therefore belongs in the routing control loop, not on a dashboard that nobody consults during an incident.

## Replace the send receipt with an evidence timeline

The weak mental model is tidy: form submitted, email accepted, ticket routed. In that model, the API's successful response becomes the end of the story. It is only evidence that an initial operation completed at the submission boundary; it does not describe every later delivery event. Use a timeline instead. In words, it looks like this: contact form accepted, routing decision recorded, message submitted, provider event observed, event normalized, suppression state changed, and next routing decision evaluated. Each arrow needs an identifier and a timestamp. The chain must survive a worker restart. That is the before/after shift. Before, a support team asks, "Did we send it?" After, an operator can ask sharper questions: Which rule selected the queue? Which provider message identifier came back? Which event caused suppression? Did the poller see it before another message was attempted? This is a compliance-evidence angle, so retention and access matter as much as transport. Store the raw event separately from the normalized state. The raw record supports later review; the normalized record drives routing. Restrict sensitive payloads, define a retention period with the people responsible for policy, and log access to evidence. Do not place the contact-form message body in every operational event merely because it is available.

The chain must be inspectable.

One boundary is especially important: an unsubscribe request and a delivery failure are different signals. RFC 8058 specifies a one-click unsubscribe mechanism for list email. A bounce is not consent withdrawal, and a complaint is not proof that an address is syntactically invalid. Keep the reason in the model instead of collapsing every negative outcome into a boolean named `blocked`.

## What must a pollable event feed prove?

Start with replay behavior. Can the consumer request a stable page after a durable cursor, or does the feed expose only a moving time window? If two events share a timestamp, can both be recovered without guessing? Does an event have a stable identifier that makes duplicate delivery harmless? These questions expose more risk than a polished event viewer.

Then inspect semantics. The API should make it possible to distinguish at least the states your policy treats differently: a terminal bounce, a temporary delivery problem, a recipient complaint, and ordinary delivery progress. Exact provider labels may vary. Your adapter should map only meanings you can defend and preserve the original label beside the mapping.

Ask for the uncomfortable evidence too:

- a documented retention or lookback boundary for events;
- deterministic pagination and an explicit ordering contract;
- correlation between the send response and later events;
- a way to recover after credentials rotate or a poller pauses;
- suppression data that can be read or exported for reconciliation;
- authentication and authorization controls suitable for a read-heavy worker; and
- documentation for rate limits and retry signals.

No single answer makes an API "best." A short event window may be acceptable when your recovery objective is shorter and a second evidence store is mandatory. It is a poor fit when a weekend outage could age events out before collection. Write that constraint down. A selection record should connect the product boundary to the system requirement, rather than awarding points for the largest feature count.

There is also an identity trap. Recipient address alone is often too broad for customer-support decisions, while provider message ID alone is too narrow for policy. Correlate the provider ID with an internal attempt ID, form submission ID, route, and policy version. Hashing or encrypting an address may reduce casual exposure, but the design still needs a controlled way to enforce suppression consistently. Decide that with your privacy and security owners; do not improvise it inside the poller.

## A copyable TypeScript polling boundary

The following example is deliberately generic. It assumes the selected API can return an ordered page after an opaque cursor. The storage methods must be implemented transactionally by the application. The numbers are operating choices for this example, not claims about a provider: fetch up to 100 records, wait 15 seconds after an empty page, and retry a failed cycle with capped exponential backoff.

```ts
type RawEvent = {
  id: string;
  messageId: string;
  recipient: string;
  kind: "delivered" | "temporary_failure" | "hard_bounce" | "complaint";
  occurredAt: string;
};

type EventPage = {
  events: RawEvent[];
  nextCursor: string;
};

interface EventSource {
  readAfter(cursor: string | null, limit: number): Promise<EventPage>;
}

interface EvidenceStore {
  currentCursor(stream: string): Promise<string | null>;
  commitPage(input: {
    stream: string;
    expectedCursor: string | null;
    nextCursor: string;
    events: RawEvent[];
  }): Promise<"committed" | "cursor_changed">;
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function collectOnce(
  source: EventSource,
  store: EvidenceStore,
): Promise<number> {
  const stream = "support-email-events";
  const cursor = await store.currentCursor(stream);
  const page = await source.readAfter(cursor, 100);

  const result = await store.commitPage({
    stream,
    expectedCursor: cursor,
    nextCursor: page.nextCursor,
    events: page.events,
  });

  if (result === "cursor_changed") return 0;
  return page.events.length;
}

async function runCollector(
  source: EventSource,
  store: EvidenceStore,
  signal: AbortSignal,
): Promise<void> {
  let failures = 0;

  while (!signal.aborted) {
    try {
      const count = await collectOnce(source, store);
      failures = 0;
      if (count === 0) await sleep(15_000);
    } catch (error) {
      failures += 1;
      const delay = Math.min(60_000, 1_000 * 2 ** Math.min(failures, 6));
      console.error("email event collection failed", { failures, error });
      await sleep(delay);
    }
  }
}
```

The transaction behind `commitPage` is the center of the example. It inserts raw events with a uniqueness constraint on `id`, derives any suppression changes, writes an audit record, and advances the cursor together. If the process stops between those operations, the page is read again. Duplicate event IDs become no-ops. Nothing is silently skipped.

Keep suppression policy outside the provider adapter. A hard bounce might block further attempts to one destination, while a temporary failure might schedule review or retry under a separate delivery policy. A complaint can trigger a stronger restriction. Those are organization decisions, so version them and record the version beside each outcome.

Do not log the full recipient or form content in the error path above. In production, emit a structured error category, a correlation identifier, the cursor generation, and the age of the oldest unprocessed event. The snippet leaves that logging adapter out because redaction and field naming belong to the surrounding system.

## Can an email API poll bounce and complaint events without webhooks?

Yes, if the feed has a recoverable position and the polling interval fits the business response target. Polling changes the freshness and load profile; it does not inherently destroy evidence. A durable cursor plus idempotent writes can make replay routine.

The limitation is latency.

It does create a measurable blind spot between provider occurrence and local observation. Track it directly as event lag: local observation time minus provider occurrence time. Also track the age of the last successful poll, pages consumed, duplicates ignored, normalization failures, and suppression decisions by reason. Alert on stalled progress and rising lag. A raw count of HTTP failures misses the nastiest case: successful polls that never move the cursor.

Picture the observability path as two lanes. The data lane reads pages and commits evidence. The control lane watches cursor movement, event age, and policy outcomes. If the data lane returns success while the cursor stays fixed, the control lane makes that visible.

Clock handling deserves care. Parse provider timestamps, retain their original values, and record a local observation timestamp. Ordering should come from the feed's documented cursor contract, not from a local comparison of timestamp strings. If the API offers no durable ordering guarantee, treat that as a selection risk and test overlapping reads rather than claiming exactly-once processing.

## Two objections worth settling before selection

"Why not use webhooks and finish faster?" Webhooks can reduce observation delay, and they are a sound choice when inbound delivery is allowed. They still require authentication, replay protection, idempotency, monitoring, and recovery. Polling is useful when network policy forbids inbound callbacks or when a pull-based evidence process is easier to govern. Its trade-off is repeated read traffic plus detection delay bounded by the polling schedule. It is a poor choice when a response must occur faster than the shortest safe polling interval, or when the event API lacks a recoverable cursor. The selection should follow constraints. It should not become an argument that one transport is universally superior.

Use the boundary you can operate.

"Why store raw events after suppression is updated?" Because mutable state answers only the present-tense question. An evidence record explains how the state came to exist, which mapping logic ran, and what the source actually reported. This separation also lets a team correct a normalization rule by replaying retained inputs, subject to its retention and privacy policy.

Security reviewers may ask how polling credentials are protected. Use the narrowest available read permissions, rotate credentials, and keep secrets out of logs and cursor records. If an administrative action needs time-based one-time passwords, RFC 6238 defines TOTP; it does not replace service authentication for an unattended collector. Keep human step-up authentication and machine credentials as separate controls.

Run failure drills before signing off. Pause the collector long enough to build a backlog, restart it during a page commit, deliver the same event twice, inject an unknown event kind, and rotate its credential. The pass condition is not "the worker came back." It is that the cursor resumes, every retained event is attributable, duplicates do not repeat policy actions, and lag returns within the declared objective.

The final decision can be concise. Pick the API whose documented event semantics and retrieval guarantees let your team produce the required evidence within its recovery window. Reject an option when a critical guarantee is absent or cannot be tested. The support-routing system then has a defensible control loop: observe, record, decide, and verify.

## Further reading

- RFC 8058, "Signaling One-Click Functionality for List Email Headers": https://datatracker.ietf.org/doc/html/rfc8058
- RFC 6238, "TOTP: Time-Based One-Time Password Algorithm": https://datatracker.ietf.org/doc/html/rfc6238
