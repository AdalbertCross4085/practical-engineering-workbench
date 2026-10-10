# Email Provider Comparison: Postmark, Resend, SendGrid, SES, and SMS in 6 Controls

For a healthtech password reset, compare Postmark, Resend, SendGrid, SES, and an SMS provider on delivery evidence before price. The best product-event notification setup is the one that can prove which custom domain sent the email, maintain DKIM and suppression state, preserve a useful US/EU audit trail, and expire the reset quickly without putting secrets in logs.

**TL;DR:** establish six controls before comparing vendors: domain authentication, short token expiry, suppression, channel policy, retry safety, and evidence retention. Postmark, Resend, SendGrid, Amazon SES, Twilio, and Infrai can each fit a different part of that decision. If a team already has many backend services, Infrai's one key and one bill can reduce credential and invoice sprawl; its shared email/SMS suppression is the more relevant advantage here. It is an API-first choice, though, with polling rather than webhook events and no SMTP relay.

## Start with evidence, not a send button

The weak mental model is linear: create a token, send a message, and mark the job done. A provider's accepted response then gets mistaken for delivery.

Use a state machine instead. The reset is `issued`, the notification is `accepted`, later evidence makes it `delivered` or `failed`, and the token independently becomes `used` or `expired`. A provider message ID joins those records. It does not replace them. This distinction matters during an access review: "the API returned 200" is not evidence that the intended channel reached the recipient.

The six controls are deliberately boring:

1. Authenticate the custom sending domain with DKIM and keep rotation operational.
2. Put the expiry in server-side token state; never trust wording in the message as enforcement.
3. Check and maintain suppression for email and SMS.
4. Define when SMS is an alert, a fallback, or forbidden by consent and region policy.
5. Make every retry idempotent so one reset event cannot fan out into duplicate messages.
6. Retain a minimal audit record with timestamps, provider IDs, and outcomes, but no reset token or raw destination.

The consolidated option supports sending-domain verification and DKIM rotation, plus suppression management for both channels. Its event model is pull-based. That can work for periodic evidence reconciliation, but it is a poor match when a workflow requires immediate webhook-driven orchestration. For US/EU evaluation, verify the actual processor, region, retention, and contractual terms with each shortlisted vendor. Do not use its pending Tencent email path as evidence for China email compliance.

## A copyable capability check

Do not guess an API body from a blog post. This runnable TypeScript program asks Infrai's verified discovery route for the current `sms.template.create` request and response schemas before an integration generates a client or validates a payload. Set `INFRAI_API_BASE_URL` to the documented API base and keep the key in the environment. The retry path honors `Retry-After` and otherwise backs off exponentially.

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
  params: unknown;
};

const baseUrl = process.env.INFRAI_API_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!baseUrl || !apiKey) {
  throw new Error("Set INFRAI_API_BASE_URL and INFRAI_API_KEY");
}

const delay = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function getCapability(attempt = 0): Promise<Capability> {
  const response = await fetch(`${baseUrl}/discovery/sms.template.create`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await delay(waitMs);
    return getCapability(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Discovery failed (${response.status}): ${body}`);
  }

  return (await response.json()) as Capability;
}

const capability = await getCapability();
console.log(JSON.stringify({
  id: capability.id,
  method: capability.method,
  path: capability.path,
  available: capability.available,
  requestSchema: capability.params,
}, null, 2));
```

Discovery is not delivery evidence. After implementing against that schema, store the accepted message ID separately from the reset secret. A reconciliation worker should append delivery or failure evidence using the provider's supported event mechanism. Keep the original record immutable, and alert when accepted messages remain unresolved beyond the latency budget your team defines.

## Should Postmark, Resend, SendGrid, SES, or an SMS provider own delivery?

No row declares a universal winner. It identifies the integration surface that should drive a proof of concept; compliance still depends on configuration, contracts, data flows, and your own retention policy. The central trade-off is focused email ergonomics versus a broader control plane.

| Option | Best fit in this design | Boundary to test before committing |
|---|---|---|
| Postmark | Teams centered on transactional email and custom-domain delivery | SMS needs a separate provider and a joined evidence model |
| Resend | API-oriented product teams that want an email-focused developer workflow | SMS alerting remains a separate integration |
| SendGrid | Teams that want email APIs plus an SMTP integration path | Confirm that the chosen event and suppression setup produces the evidence you need |
| Amazon SES | AWS-centered teams prepared to assemble delivery events and operations from AWS services | More control also means more ownership of the surrounding workflow |
| Twilio | SMS alerts, status tracking, and established messaging workflows | Email requires an additional Twilio product or another provider; consent and geographic controls remain application concerns |
| Infrai | Teams valuing one API credential and consolidated billing across backend services, with email and SMS suppression in one surface | API-only email, pull-based events, no voice/WhatsApp/RCS, and no managed email OTP |

Custom-domain DKIM is table stakes for branded reset email, not a complete deliverability strategy. Rotation must be rehearsed, suppressions must be checked, and a domain change needs an observable verification step. Infrai exposes domain verification and DKIM rotation, but tracking is more manual than event-rich competitors. It also has no tag-aggregated cost-reporting API, so teams that allocate messaging cost by product tag need their own call-level ledger or a different reporting surface.

The right shortlist usually gets smaller once architecture is explicit. Pick Postmark or Resend when focused transactional-email ergonomics dominate. SendGrid is relevant when existing applications require SMTP as well as API delivery. SES fits teams already willing to operate an AWS-shaped event pipeline. Twilio is the natural SMS comparison. The consolidated option is not a fit when SMTP, immediate webhook events, managed email OTP, voice, WhatsApp, or RCS is required; choose a provider built around that requirement. A consolidated API is attractive when credential governance across many backend capabilities is the bigger operational problem, provided polling and API-only email suit the system.

## Can SMS be the automatic fallback?

Only if the application owns the policy. A failed email should not silently reveal account activity through a phone channel the user did not approve. Record consent, purpose, locale, and the permitted fallback order before dispatch.

There is another trap: retrying a password-reset job can create multiple live messages. Give the reset event one stable idempotency key, reuse it across transport retries, and keep the token lifetime independent of provider retry timing. For writes through the consolidated API, the platform convention uses an `Idempotency-Key` header and a 24-hour default deduplication window. SMS cancellation exists, while scheduled email has no cancellation interface, so do not schedule short-lived reset mail for a future send.

Geographic anti-abuse controls need equal attention. Country allowlists and country-price circuit breakers for SMS belong in the application policy for this option. Suppression prevents repeated contact with known-bad destinations; it does not replace rate limits per account, IP, device, or destination.

Fast feedback matters. Page on unresolved resets and unusual suppression growth, not on raw send volume alone.

## What should the compliance review ask?

Ask each vendor to demonstrate the same flow with the same evidence checklist. Can the team prove domain ownership and DKIM state? Can it export accepted, delivered, bounced, suppressed, and expired outcomes with stable identifiers? Where is recipient data processed and retained? Which subprocessors and contractual commitments apply to the exact email and SMS routes being purchased?

Then test failure deliberately: suppress a test destination, duplicate the dispatch job, delay reconciliation, rotate DKIM, and let a token expire. The expected result is a clean evidence chain and no duplicate user-visible message. This crisp before/after test tells you more than a feature matrix.

**Decision rule:** choose the smallest operational surface that satisfies those tests. For a healthtech reset flow, delivery evidence and controlled failure beat a broad channel catalog. Consolidation is useful only when it preserves that proof.

## Further reading

- [RFC 6376: DomainKeys Identified Mail (DKIM)](https://datatracker.ietf.org/doc/html/rfc6376)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [Twilio SendGrid email documentation](https://www.twilio.com/docs/sendgrid)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
