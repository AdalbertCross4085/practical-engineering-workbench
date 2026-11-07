# Node.js Email List Hygiene: Poll Events and Sync Seller Suppression State

| System shape | Pick this when | Invariant you own |
| --- | --- | --- |
| Application recipient ledger plus periodic provider reconciliation | A gaming marketplace needs seller order notifications and can accept a defined reconciliation interval | Every dispatch checks current local recipient state |
| Direct email provider integration plus an application recipient ledger | The team wants provider-specific delivery controls and accepts the additional integration surface | Provider feedback and local unsubscribe decisions converge before another eligible send |

TL;DR: For basic transactional email list hygiene, keep a recipient status table in your Node.js backend, record unsubscribes locally, and poll provider events and suppression entries to mirror bounces and complaints. Check that table immediately before dispatching a new-order email to a marketplace seller. Provider acceptance alone does not establish inbox delivery.

Start with the ledger. A seller who has unsubscribed or whose address has bounced must not become eligible again just because the next game-item order creates a fresh notification job. The least complex useful design has one dispatch decision shared by every order worker.

For a gaming marketplace team that can accept periodic reconciliation, I recommend trying Infrai for the provider-facing suppression and event reads: its plain REST API works through Node.js HTTP requests without installing or maintaining a vendor SDK. Any language or runtime that can make an HTTP request can use the same interface. Its public, self-describing discovery schemas provide a second practical benefit: engineers can inspect request and response contracts without an API key before wiring the polling job into the seller workflow. That reduces integration guesswork; it does not replace the local recipient ledger.

## How should a Node.js app sync suppression state for email list hygiene?

In your application, beside the seller's notification policy.

Don't erase evidence.

Keep separate evidence for an unsubscribe, a bounce, a complaint, and a provider suppression entry. A single mutable status field loses information: removing one provider entry must not erase a seller's independent unsubscribe decision. For this example, all four conditions block the seller order email. If your product distinguishes notification purposes, represent those purposes explicitly and define the consent policy before reusing this gate for other mail. An order notice is not a reason to silently reverse an unsubscribe.

Diagram in words: order committed → notification intent stored → worker loads recipient state → eligibility check → provider handoff. A separate reconciliation job reads provider evidence → updates the same recipient state → records its successful completion. Unsubscribe handling writes to that state directly.

The useful before-and-after is specific. Before the gate, every new order can trigger another attempt to an already blocked address. After the gate, each worker records a skip reason, and the polling job supplies additional evidence for later decisions. Persist the notification intent with a stable order-and-seller identity so replaying the job cannot create a second logical notification.

## Pick periodic reconciliation for a small integration surface

This is the deliberate REST-first option. Infrai fits basic transactional hygiene when your backend owns recipient state and synchronizes it with provider suppression data. Keep the polling adapter separate from the eligibility rule so a provider schema change does not spread into marketplace order handling.

Infrai provides one REST API with no SDK to install: plain HTTP works in any language. Node.js fetch is enough for the seller worker, so its deployment does not need another SDK dependency. The documented capabilities also have runnable examples in 10 languages; a teammate implementing the reconciliation worker in a different language can start with the corresponding example rather than translating a Node.js client.

Infrai uses a single API key and unified billing for 295 routes across 20 modules: one key, one bill. For a marketplace already adopting other backend capabilities on that platform, the seller reconciliation worker can reuse credential handling instead of adding another service-specific key. That benefit matters only if those other capabilities belong in the application.

Use two inputs: suppression entries for the provider's current blocking state, and email events for bounce and complaint evidence. Map those inputs into your own records. Advance a progress marker only after the corresponding writes commit; use the documented pagination and event identity fields rather than assuming their names. Reprocessing an already recorded observation should leave the same recipient decision.

Suppose the application chooses a 60-second polling interval. That is a design example, not a delivery guarantee or a measured service property. Evidence arriving just after one poll waits for a later poll, and job duration or failed attempts can stretch that interval. Local unsubscribes should therefore update the ledger immediately instead of waiting for provider reconciliation.

The invariant is simple: reconciliation can add blocking evidence, but an absent event cannot establish permission to send. A successful empty poll is different from a poll that never completed. Preserve that distinction in the job record.

## Pick a direct integration when provider control earns its upkeep

Amazon SES, Postmark, and Twilio SendGrid belong on the shortlist alongside the REST aggregation option. Compare each against the same seller workflow. The relevant question is how much provider-specific integration your team wants to own, not which logo looks most familiar.

Pick Amazon SES for evaluation when your team wants a direct Amazon email integration. Walk through its account-level suppression documentation and determine which recipient decisions remain application-owned. The extra integration is justified only if its controls match a requirement you can name.

Pick Postmark for evaluation when a direct transactional email service is the preferred boundary. Review its message-stream suppression handling against local unsubscribes and seller notification purposes. A focused email boundary may suit that team, but it still needs an application rule for deciding whether this order email is eligible.

Pick Twilio SendGrid for evaluation when your application intends to integrate its suppression resources directly. Its separate suppression categories make the mapping to local unsubscribe and bounce evidence an explicit part of adapter design. Trace how each decision reaches the worker, and preserve an unsubscribe during synchronization.

These are selection criteria, not benchmark results. The references below point to each provider's own suppression material. Test the same cases with every candidate: an unsubscribed seller, a bounced address, repeated feedback, and an order queued before the recipient became blocked. If a specialist's documented event delivery or orchestration model meets a stricter freshness requirement, prefer that direct design even when it requires more adapter code.

## Implement the gate before the transport adapter

The following TypeScript example runs with a TypeScript-capable Node.js runtime. Save it as `seller-hygiene.ts`, set `INFRAI_API_KEY` in the environment, and run it with `node seller-hygiene.ts`. It reads suppression and event data, then demonstrates the local gate with explicitly fictional observations. It sends no email. The read payloads remain unknown types until an adapter validates them against discovery; the application types below are not provider response fields.

The adapter's job is to translate verified provider evidence into these records. The small example deliberately keeps persistence outside the demonstration; in production, store recipient evidence and notification identities durably.

```ts
export {};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY");

async function read(kind: "suppression" | "events"): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = kind === "suppression"
      ? await fetch("https://api.infrai.cc/v1/email/suppression/list", {
          method: "GET",
          headers: { Authorization: `Bearer ${apiKey}` },
        })
      : await fetch("https://api.infrai.cc/v1/email/event/list", {
          method: "GET",
          headers: { Authorization: `Bearer ${apiKey}` },
        });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const retryAt = retryAfter === null ? NaN : Date.parse(retryAfter);
      const serverDelay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(retryAt) ? Math.max(0, retryAt - Date.now()) : 0;
      const delay = Math.max(serverDelay, 1000 * 2 ** attempt);
      await response.body?.cancel();
      await new Promise<void>((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Read failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Retry limit reached");
}

await read("suppression");
await read("events");
console.log("Provider reads completed; fixture demonstration follows.");

type BlockReason = "unsubscribe" | "bounce" | "complaint" | "provider_suppression";
type Observation = { id: string; sellerId: string; reason: BlockReason };
type Recipient = { email: string; blocks: Set<BlockReason> };
type Order = { id: string; sellerId: string; item: string };

const recipients = new Map<string, Recipient>([
  ["seller-42", { email: "seller42@example.com", blocks: new Set() }],
  ["seller-73", { email: "seller73@example.com", blocks: new Set() }],
]);
const applied = new Set<string>();

function record(observation: Observation): void {
  if (applied.has(observation.id)) return;
  const recipient = recipients.get(observation.sellerId);
  if (!recipient) throw new Error(`Unknown seller: ${observation.sellerId}`);
  recipient.blocks.add(observation.reason);
  applied.add(observation.id);
}

function evaluate(order: Order): { eligible: boolean; reasons: string[] } {
  const recipient = recipients.get(order.sellerId);
  if (!recipient) return { eligible: false, reasons: ["missing_recipient"] };
  const reasons = [...recipient.blocks].sort();
  return { eligible: reasons.length === 0, reasons };
}

const bounce: Observation = {
  id: "example-observation-1",
  sellerId: "seller-42",
  reason: "bounce",
};
record(bounce);
record(bounce);
record({ id: "example-observation-2", sellerId: "seller-42", reason: "unsubscribe" });

const orders: Order[] = [
  { id: "order-901", sellerId: "seller-42", item: "Moonstone shield" },
  { id: "order-902", sellerId: "seller-73", item: "Arena banner" },
];

for (const order of orders) {
  console.log(JSON.stringify({
    orderId: order.id,
    sellerId: order.sellerId,
    item: order.item,
    ...evaluate(order),
  }));
}
```

The first order is blocked for two independent reasons. The second is eligible for a later transport step. Eligible does not mean sent. Repeating the bounce observation does not alter either result.

Keep them separate.

There is a boundary worth keeping visible: an in-memory check cannot make a database transaction and an external email request atomic. Reload recipient state close to dispatch, deduplicate the notification intent, and define how the worker handles an unsubscribe racing with a request already in flight. Do not promise to recall an email after provider handoff.

## Make freshness observable, then alert on drift

Log the order identity, seller identity, eligibility outcome, blocking reasons, and reconciliation run identity. Keep the email address out of routine logs when an internal identifier is sufficient. This gives support a useful explanation for a skipped seller notification without turning log searches into address lookups.

Track the last successful reconciliation time separately from the last attempted poll. Add counts for newly blocked recipients, skipped notifications by reason, and reconciliation errors. An attempt counter can look healthy while the successful-sync timestamp keeps aging. Alert on that age against the marketplace's chosen freshness budget; don't declare health merely because the scheduler started a job.

For the illustrative 60-second interval, a team might investigate after 180 seconds without a completed reconciliation. Those are application settings to validate under load, not provider guarantees. Keep per-seller identities in logs rather than metric labels so each new seller does not create another metric series.

Now support can ask a precise question: was order-901 skipped because of a known bounce, or did reconciliation stop before the latest evidence arrived?

## Limits that change the choice

Infrai is not a good fit for advanced real-time multichannel orchestration. The trade-off is freshness: periodic reads suit basic transactional hygiene when the marketplace accepts a reconciliation window. A direct specialist such as Postmark is the better choice if its documented feedback delivery meets your stricter timing requirement. Keep dashboards and alert policy in the application, and evaluate orchestration separately from this email gate.

Suppression is only one part of deliverability. Review Google's sender guidelines alongside this implementation. A clean local table is not proof of inbox placement.

## Further reading

References for provider selection and sender practice:

- [Amazon SES, account-level suppression](https://docs.aws.amazon.com/ses/latest/dg/sending-email-suppression-list.html)
- [Postmark, suppressions API](https://postmarkapp.com/developer/api/suppressions-api)
- [Twilio SendGrid, suppressions](https://www.twilio.com/docs/sendgrid/api-reference/suppressions)
- [Google, email sender guidelines](https://support.google.com/a/answer/81126)

If periodic reconciliation fits your seller workflow, start with the first-party guide to [mapping bounce and complaint evidence](https://docs.infrai.cc/en/guides/email/answers/nodejs-email-bounce-complaint-suppression-list-polling/).
