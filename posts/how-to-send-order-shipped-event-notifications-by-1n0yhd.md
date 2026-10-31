# How to Send Order Shipped Event Notifications by Email and SMS (with Routing Evidence)

When a healthtech system must send an order-shipped event notification by email or SMS, treat the contact form as a routing decision before treating it as a messaging feature. My default is to accept the event once, record the routing decision, and deliver each channel through a separate, retryable worker. The durable result is an audit trail that answers three questions: what arrived, which queue was selected, and what happened to each delivery attempt.

**TL;DR:** Give every submission a stable event ID. Store the normalized payload and an outbox record in one transaction. Make workers idempotent, retry only transient failures, and move exhausted work to a dead-letter queue with a reason. Keep PHI out of message bodies and logs unless the channel and policy explicitly allow it.

This is the before-and-after mental model:

- Before: an HTTP handler decides a queue and calls two providers inline. A timeout leaves the sender unsure whether a message was accepted.
- After: the handler commits one event and two delivery intents. Workers own channel attempts, while metrics and structured logs expose the state transition.
- After: the handler commits one event and two delivery intents. Workers own channel attempts, while metrics and structured logs expose the state transition.

The queue is the boundary.

## How should we send an order-shipped event notification by email?

Start with the smallest useful envelope. A healthtech contact form might contain a category such as `billing`, `appointment`, or `technical-support`, a preferred channel, a callback address, and a consent flag. Treat free-text content as sensitive input. Store a reference to it when the support workflow needs the text; do not copy it into every delivery record.

The event ID is the idempotency key. It must remain stable when a client retries the HTTP request. A database uniqueness constraint is stronger than a check followed by an insert, because concurrent requests can pass the check together. For a shipment event, the same key should identify the shipment transition, even if a webhook or internal publisher repeats it.

```ts
type ContactEvent = {
  eventId: string;
  submittedAt: string;
  category: "billing" | "appointment" | "technical-support";
  email?: string;
  phone?: string;
  consent: { email: boolean; sms: boolean };
  messageRef: string;
};

type DeliveryIntent = {
  eventId: string;
  channel: "email" | "sms";
  templateId: string;
  queue: string;
};
```

The template ID belongs in the intent, not in a worker constant. That lets an operator explain why a particular message was rendered later. It also makes a template change a deliberate deployment decision.

## How do you make routing and delivery idempotent?

Use a transaction for the form event and its delivery intents. The exact database API varies, but the invariant does not: either all records become visible, or none do. A background worker then claims an intent and records the provider response separately from the attempt itself.

```ts
async function acceptContact(event: ContactEvent, db: Database) {
  return db.transaction(async (tx) => {
    const inserted = await tx.contactEvents.insertIgnore(event, {
      unique: ["eventId"]
    });

    if (!inserted) {
      return { status: "already-accepted", eventId: event.eventId };
    }

    const queue = routeQueue(event.category);
    const intents: DeliveryIntent[] = [];

    if (event.email && event.consent.email) {
      intents.push({ eventId: event.eventId, channel: "email", templateId: "support-received-email-v2", queue });
    }
    if (event.phone && event.consent.sms) {
      intents.push({ eventId: event.eventId, channel: "sms", templateId: "support-received-sms-v2", queue });
    }

    await tx.deliveryIntents.insertMany(intents, {
      unique: ["eventId", "channel"]
    });
    return { status: "accepted", eventId: event.eventId, queue };
  });
}
```

The important decision is what happens after a worker timeout. It must not blindly create a second intent. It retries the same intent, with the same event ID, and the channel adapter must tolerate a repeated request. If the downstream API offers an idempotency key, pass the intent ID. If it does not, keep a delivery-attempt record and choose a duplicate-prevention policy that the team can explain to support staff.

Use bounded exponential backoff with jitter for transient transport errors. A useful example policy is three retries at roughly 30 seconds, 2 minutes, and 10 minutes; choose values from the channel's limits and the form's service objective. A validation error, revoked consent, or malformed destination should be terminal immediately.

## Where does observability change the support outcome?

A log line saying `send failed` is not an operational signal. Emit a structured event for accepted, queued, attempted, delivered, retried, and dead-lettered states. Include event ID, intent ID, channel, queue, template ID, attempt number, and a redacted error class. Exclude message text, access tokens, and unnecessary identifiers.

Metrics should answer two different questions. A counter shows volume by channel and result. A histogram shows time from acceptance to the first attempt and from acceptance to a terminal state. Alert on a rising retry ratio, queue age, and dead-letter rate. A successful HTTP response from the form endpoint is not proof that an email or SMS arrived.

For a support queue, the useful trace is a short chain:

`contact.received -> routing.decided -> delivery.queued -> delivery.attempted -> delivery.delivered`

That diagram-in-words gives an on-call engineer a place to start. If `routing.decided` is missing, inspect the transaction. If `delivery.queued` is present but queue age grows, inspect workers. If attempts finish but delivery remains unknown, inspect the channel's response semantics.

A dead-letter item is a recoverable investigation record, not a trash can. Store the original intent ID, the last safe payload reference, attempt count, timestamps, and a classified reason. Preserve enough information to replay after correction, while applying the same retention and access controls as the source event.

Do not automatically replay every dead-letter item. First decide whether the failure was a temporary outage, a bad destination, a policy decision, or a template defect. Replaying a policy failure can create an unwanted message; replaying a transient outage may be correct. A manual replay should create a new attempt linked to the original intent, leaving the old history intact.

This design has a boundary. It is a poor fit for a tiny internal tool that can tolerate a missed notification and has no support queue; the transaction, worker, and replay history add operational work. It is a better fit when a healthtech team must explain a routing decision or distinguish a delayed message from a rejected one. That is a reliability trade-off, not a universal prescription.

## Two objections worth resolving

“Why send both email and SMS?” Because they have different delivery properties and different consent requirements. A channel matrix makes that explicit. For example, an appointment-related form can create an email intent when email consent exists, while SMS may remain disabled until the contact has opted in under the organization’s policy. The router should record the decision instead of silently falling back to another channel.

“Can the message include the user’s details?” Minimize the body. A notification can say that a support request was received and provide a case reference without reproducing free text or clinical details. Identity and authenticator guidance from NIST is a useful reminder that authentication and communication are separate controls; a notification channel should not become an accidental identity-proofing mechanism.

The practical finish line is evidence, not a green send button. You should be able to reconstruct the route, consent decision, attempts, and final disposition for one event without reading sensitive content. That standard keeps the implementation useful to support teams and honest about what delivery systems can prove.

## References

- Amazon Simple Email Service documentation: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html

## Sources

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://pages.nist.gov/800-63-3/sp800-63b.html
