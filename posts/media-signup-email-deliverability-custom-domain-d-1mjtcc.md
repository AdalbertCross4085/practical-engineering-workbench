# Media Signup Email Deliverability — Custom Domain DKIM and Suppression Tests

A media signup flow has one unforgiving constraint: the verification link is useful only while the reader is still trying to create an account. **Treat deliverability as a tested control loop, not a DNS checklist.** Authenticate the custom sending domain with SPF, DKIM, and DMARC; record each send attempt; poll bounce and complaint evidence into a suppression decision; then test that a known failure prevents the next attempt.

TL;DR: prove four observable transitions before launch: `requested -> submitted`, `submitted -> delivered_or_failed`, `failed -> suppressed`, and `suppressed -> blocked_before_send`. DNS authentication establishes identity. The event loop establishes whether the application learns from delivery outcomes.

## Start with the failure rehearsal

The useful before-and-after model is short. Before, the signup service calls a mail transport and treats an accepted submission as success. After, it creates a traceable delivery attempt, lets the transport report what happened, and changes later behavior when a permanent failure or complaint arrives.

Accepted is not delivered.

That distinction changes the launch test. A `250` SMTP reply or an API acceptance means the next system accepted responsibility for the message; it does not prove inbox placement. Likewise, SPF, DKIM, and DMARC passing does not prove that a mailbox exists or that a recipient wants the message. The test plan must observe both layers without merging them into one green checkmark.

Use a small matrix built around the media signup journey:

| Rehearsal | Expected evidence | Required application behavior |
| --- | --- | --- |
| Valid controlled mailbox | Submission plus eventual delivery disposition | Keep the address eligible |
| Controlled permanent failure | Bounce with a permanent status | Suppress before another verification send |
| Controlled transient failure | Temporary-failure evidence | Retain evidence and follow a bounded retry policy |
| Controlled complaint path | Complaint evidence | Suppress later messages |
| Same event fetched twice | Duplicate source event ID | Apply one state transition |
| Poller stopped, then restarted | Growing then recovering event age | Alert on lag and resume from the saved cursor |

Run these cases with addresses and reporting paths you control. Do not probe arbitrary recipients. The point is to demonstrate state transitions, not to generate traffic.

Authentication gets its own preflight. SPF publishes which hosts may use a domain in the SMTP envelope. DKIM signs selected message content and headers with a domain-associated key. DMARC evaluates alignment between the visible `From` domain and an authenticated SPF or DKIM domain, then supplies policy and reporting mechanisms. A practical rollout inspects DMARC aggregate reports before moving to a stricter policy because forwarding can disrupt SPF and message transformations can disrupt DKIM.

The crisp launch artifact is a table of evidence, not a screenshot of three DNS records. Record the test time, message correlation ID, authentication result, terminal delivery category, suppression transition, and the latency from event occurrence to local application. That gives an operator something they can compare after a DNS change or mail-stream migration.

## How should a Node.js custom domain email setup handle deliverability evidence?

It should own correlation, normalization, and the final eligibility check. It should not pretend to own the receiving server's decision.

The transport boundary returns a submission identifier. The poller later translates transport-specific event labels into a deliberately small application vocabulary while retaining the raw label for investigation. The signup worker checks current suppression immediately before submission, because a check performed when the job was queued may already be stale.

Here is one copyable TypeScript sketch. It shows the control loop rather than any commercial SDK. The repository is expected to enforce uniqueness for `(source, sourceEventId)` and commit an inserted event, its projection change, and its cursor progress atomically.

```ts
type DeliveryKind =
  | "delivered"
  | "temporary_failure"
  | "permanent_failure"
  | "complaint";

type DeliveryEvent = {
  source: string;
  sourceEventId: string;
  attemptId: string;
  recipient: string;
  kind: DeliveryKind;
  occurredAt: Date;
  rawKind: string;
};

type SignupMessage = {
  attemptId: string;
  recipient: string;
  verificationUrl: URL;
};

interface DeliveryStore {
  isSuppressed(recipient: string): Promise<boolean>;
  insertEventOnce(event: DeliveryEvent): Promise<boolean>;
  suppress(recipient: string, reason: "permanent_failure" | "complaint"): Promise<void>;
  transaction<T>(work: () => Promise<T>): Promise<T>;
}

interface MailTransport {
  submit(message: SignupMessage): Promise<{ submissionId: string }>;
}

function normalizeAddress(address: string): string {
  const trimmed = address.trim();
  const separator = trimmed.lastIndexOf("@");
  if (separator < 1 || separator === trimmed.length - 1) {
    throw new Error("invalid email address shape");
  }

  const local = trimmed.slice(0, separator);
  const domain = trimmed.slice(separator + 1).toLowerCase();
  return `${local}@${domain}`;
}

async function applyDeliveryEvent(
  store: DeliveryStore,
  event: DeliveryEvent,
): Promise<void> {
  await store.transaction(async () => {
    const inserted = await store.insertEventOnce(event);
    if (!inserted) return;

    if (event.kind === "permanent_failure" || event.kind === "complaint") {
      await store.suppress(normalizeAddress(event.recipient), event.kind);
    }
  });
}

async function sendVerification(
  store: DeliveryStore,
  transport: MailTransport,
  message: SignupMessage,
): Promise<string> {
  const recipient = normalizeAddress(message.recipient);
  if (await store.isSuppressed(recipient)) {
    throw new Error("verification email blocked by delivery evidence");
  }

  const result = await transport.submit({ ...message, recipient });
  return result.submissionId;
}
```

The normalization is intentionally conservative. It lowercases the domain but does not strip dots, remove plus tags, or lowercase the local part according to one mailbox provider's conventions. Those rewrites can merge addresses the application has no right to treat as identical.

There is also a race worth naming. An event can arrive after the eligibility check but before submission. A local projection cannot recall that in-flight message. The control loop instead makes the guarantee precise: once decisive evidence is committed, subsequent checks block subsequent attempts. If the business requires a tighter window, shorten ingestion delay and serialize decisions where practical; do not claim that polling eliminates the race.

## Make event age the page-worthy signal

A scheduled poller can run on time and still provide stale evidence. It might repeatedly fetch the same page, fail to decode a new event type, or advance a cursor before the page's writes commit. A heartbeat stays green through all three.

Measure the age of the newest successfully incorporated source event against the newest available event, when the source exposes enough information to do so. Also retain cursor age, events fetched, duplicate events, unknown event kinds, parse failures, and projection failures. The exact alert threshold belongs to the signup link's validity window and the team's response target; inventing a universal number would hide the real decision.

Diagram in words: **receiver outcome -> reporting feed -> poll cursor -> immutable event -> recipient projection -> pre-send gate**. Put a correlation ID through the send attempt and normalized event so an operator can walk that chain. Avoid recipient addresses in metric labels; high-cardinality personal data makes poor telemetry and creates unnecessary exposure. Logs can carry a protected lookup key under the organization's retention and access rules.

Cursor handling deserves one transaction boundary. Fetch a page, normalize its recognized events, insert each event idempotently, update projections, and save the next cursor only after those writes succeed. If the process dies before commit, the page returns. Duplicate IDs make replay boring. Good.

Picture the failure at page 41. The worker inserts a permanent failure, updates the recipient projection, and stops before its cursor commit. On restart, page 41 appears again; the source event ID turns the second insert into a no-op, and the cursor can advance without applying the suppression twice. Now reverse the unsafe order: if the cursor advances first and the database write then fails, that permanent failure may be skipped on the next poll. The signup service could keep sending verification links to an address the mail system has already rejected. This is why cursor progress belongs in the same commit as the local evidence, or must use an equivalent recoverable handoff when the source and database cannot share a transaction.

Unknown event kinds should not disappear. Store the raw event for investigation, increment a visible counter, and leave eligibility unchanged until the mapping is understood. Treating every unknown value as a permanent failure can lock out legitimate readers; treating it as success can keep sending after meaningful negative evidence. Visibility is the safer default.

The main limitation is detection lag, and the trade-off is explicit.

Polling trades operational simplicity for detection delay. It is not appropriate when the maximum acceptable suppression window is shorter than the source's publication delay plus the polling interval; choose pushed event ingestion into the same idempotent boundary in that case. A local projection also cannot prove inbox placement, restore domain reputation, or recover an outcome the reporting source never publishes. Those are explicit boundaries, not reasons to discard the loop.

## Does an open prove the verification message arrived?

No. Apple Mail Privacy Protection can prevent senders from learning whether a recipient opened an email. Open tracking therefore cannot be the success oracle for this control loop, and an apparent open must never clear a suppression created by a complaint or permanent failure.

For the application journey, the strongest positive signal is completion of the verification action tied to the account request. Keep that product event distinct from transport evidence. A clicked verification link says the user received enough of the message to act; it does not rewrite the mail system's historical bounce or complaint record.

This separation also improves debugging. Product metrics answer “did the reader finish signup?” Transport metrics answer “what did the mail path report?” Authentication reports answer “did receivers observe aligned authenticated mail from this domain?” One dashboard can place them side by side, but one number should not impersonate all three.

## Two objections to settle before release

“Can we skip polling because authentication passes?” No. Authentication makes domain identity assessable; it does not report a nonexistent mailbox, a later complaint, or every other reason a receiver can reject mail. The authenticated stream still needs outcome ingestion and an enforced suppression path.

“Can we make suppression a Boolean and periodically reset it?” A Boolean can serve the hot send path, but it is a weak record. Keep the decisive event, occurrence time, source identifier, and reason behind the projection. Automatic expiry is especially hard to justify for complaints and permanent failures because elapsed time is not evidence that consent returned or an address became valid. A correction should create an auditable transition rather than erase the event that produced the block.

Temporary failures need separate treatment. RFC 3463 defines three top-level enhanced-status classes: `2` for success, `4` for persistent transient failure, and `5` for permanent failure. Define a bounded retry policy using that enhanced status, elapsed time, and verification-link usefulness; then test it with controlled inputs. Collapsing every failure into permanent suppression drops recoverable mail. Retrying forever spends reputation on an outcome that may never change.

Ship when the rehearsal proves the whole loop: aligned authentication is visible, every attempt is correlatable, duplicate polling is harmless, lag is alertable, decisive negative evidence changes recipient state, and the next send is blocked locally. That is a narrower claim than “we guarantee inbox placement.” It is also one an engineering team can verify.

## Sources

- https://datatracker.ietf.org/doc/html/rfc7208
- https://datatracker.ietf.org/doc/html/rfc6376
- https://datatracker.ietf.org/doc/html/rfc7489
- https://datatracker.ietf.org/doc/html/rfc3463
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
