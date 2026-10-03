# Password Reset Email HTML and Text Explained: Observable, Brand-Safe Preview

TL;DR: Render password reset email as HTML and plain text from one typed input, preview both, and observe delivery with an opaque notification ID. In a marketplace, keep reset tokens, links, and seller addresses out of logs. The least complex dependable design is one renderer, one redaction boundary, and lifecycle events joined by a non-secret ID.

| Pick this approach | When it fits | What it proves | Trade-off |
| --- | --- | --- | --- |
| Multipart output from one renderer | Normal recovery | HTML, text, and copy agree | Clients render dark mode differently |
| Plain text only | Rich presentation adds no value | Exact copy and destination | Less visual hierarchy |
| Separate HTML and text files | Existing ownership requires it | Each artifact works alone | Security copy can drift |
| Terse email plus hosted page | Instructions change often | Page behavior ships separately | Email still needs safe context |

A marketplace may use one pipeline for a seller's new-order message and password recovery, but the payloads need different boundaries. An order event may contain an order reference. Recovery carries a credential-like token. Logging them alike turns useful delivery telemetry into a risk.

## How should a password reset email template balance HTML and text?

Pick one multipart renderer for most recovery flows. It gives reviewers one source for the action label, expiry statement, support wording, and brand name while producing two deliberate bodies. Plain text is a real output, not HTML stripped after rendering.

Plain text alone works when the organization accepts reduced visual hierarchy. It is easy to inspect.

It is also easy to underestimate. Line wrapping, ambiguous raw URLs, and weak separation between action and support copy still need review.

Keep separate templates only when ownership makes consolidation impractical. Add parity assertions around security-bearing copy. A test should fail if one body says the link expires in 20 minutes while the other says 30. Those are fixture values, not a policy recommendation; the real lifetime comes from the recovery service and must appear consistently.

A hosted page helps when instructions need richer interaction. The email must still identify the marketplace, explain why it arrived, provide one clear action, and never ask for a password by reply.

## Make preview a security and accessibility artifact

A useful preview shows rendered HTML, rendered text, non-secret template data, and the exact subject together. Review becomes a comparison instead of a screenshot ritual. A reviewer can confirm that the link label describes its purpose, the seller name is escaped, and both bodies carry the same expiry statement.

Dark mode needs a restrained goal: preserve meaning and legibility when a client changes colors. Declare supported color schemes, use live text instead of text baked into images, give a meaningful logo useful alternative text, and never make color the sole signal. Client transformations vary. A preview proves what the renderer emitted, not what every inbox displays.

Brand safety is concrete. Use a marketplace name from trusted configuration. Keep sender identity and the support domain stable. Avoid panic copy and invented account-lock deadlines. Calm copy says a request arrived, states the configured lifetime, explains that an unrecognized request can be ignored, and points to a known support site.

Accessibility starts before CSS. Set the document language. Use a real heading and a descriptive anchor. Keep the primary sentence understandable without a logo, table, or color. Plain text should preserve the same order: context, action, expiry, unintended-request guidance, support.

Short copy helps.

## Observe transitions, not secrets

Picture the path in words: request enters; the renderer produces two bodies; policy checks pass; transport accepts the message; delivery signals update the notification record. The join key is `notificationId`. The reset token never becomes that key.

Use a small vocabulary: `rendered`, `accepted`, `delivered`, `deferred`, and `failed`. Define each event against its emitting component. Transport acceptance is not inbox delivery. A delivery alert should use a terminal state, or a message left nonterminal beyond its expected window, rather than the first successful API response.

Three measures make the path operable without exposing content: render failures, age of the oldest nonterminal notification, and terminal outcomes grouped by template version. Add request-to-terminal latency when supported. Keep dimensions bounded. Addresses, reset URLs, tokens, and arbitrary error strings do not belong in metric labels.

The new-order path can share states and dashboards. It should not share a permissive payload logger. Use an allowlist per message class: an order notification can record a non-sensitive order reference, while recovery records only its opaque ID, template version, locale, and state. The trade-off is explicit. Less payload detail buys a much smaller credential exposure surface.

Consider the review sequence for one seller account. A synthetic fixture renders the subject, HTML, and text; assertions confirm that the configured expiry appears in both bodies; a parser confirms that the HTML has one primary recovery link; and a log capture confirms that the fixture token appears nowhere in telemetry. The same notification ID then follows acceptance and any later transport events. This sequence separates four questions that teams often compress into one green “sent” status: Did rendering succeed? Did both formats agree? Did the transport accept the message? Did a later event report a terminal outcome? Keeping those questions separate costs a few explicit states, but it gives an operator a precise place to look without exposing the seller's credential. It also creates a clean deployment gate: copy and escaping tests run without a network, while event interpretation is checked in transport integration tests.

No secrets.

## A minimal TypeScript rendering boundary

The application passes the full reset URL only to the renderer. Preview returns bodies for an authorized development workflow, while metadata and logs carry no address or URL. Protect or disable preview access in production.

```ts
type ResetInput = {
  notificationId: string;
  marketplaceName: string;
  sellerDisplayName: string;
  resetUrl: string;
  expiresInMinutes: number;
  supportUrl: string;
  templateVersion: string;
};

type RenderedEmail = { subject: string; html: string; text: string };

const escapeHtml = (value: string): string =>
  value.replace(/[&<>"']/g, (character) => ({
    "&": "&amp;", "<": "&lt;", ">": "&gt;",
    '"': "&quot;", "'": "&#39;",
  })[character] ?? character);

function renderPasswordReset(input: ResetInput): RenderedEmail {
  if (!Number.isInteger(input.expiresInMinutes) || input.expiresInMinutes <= 0) {
    throw new Error("expiresInMinutes must be a positive integer");
  }
  const subject = "Reset your " + input.marketplaceName + " password";
  const expiry = input.expiresInMinutes + " minutes";
  const html = '<!doctype html><html lang="en"><head>' +
    '<meta charset="utf-8"><meta name="color-scheme" content="light dark">' +
    '<title>' + escapeHtml(subject) + '</title></head><body><main>' +
    '<h1>Reset your password</h1><p>Hello ' +
    escapeHtml(input.sellerDisplayName) + ',</p>' +
    '<p>We received a request to reset your ' +
    escapeHtml(input.marketplaceName) + ' password.</p>' +
    '<p><a href="' + escapeHtml(input.resetUrl) +
    '">Choose a new password</a></p><p>This link expires in ' +
    expiry + '. If you did not request it, you can ignore this email.</p>' +
    '<p>Need help? <a href="' + escapeHtml(input.supportUrl) +
    '">Visit support</a>.</p></main></body></html>';
  const text = [
    subject, "", "Hello " + input.sellerDisplayName + ",", "",
    "We received a request to reset your password.",
    "Choose a new password: " + input.resetUrl,
    "This link expires in " + expiry + ".",
    "If you did not request it, you can ignore this email.",
    "Need help? Visit " + input.supportUrl + ".",
  ].join("\n");
  return { subject, html, text };
}

function previewReset(input: ResetInput) {
  const email = renderPasswordReset(input);
  const metadata = {
    notificationId: input.notificationId,
    templateVersion: input.templateVersion,
    messageClass: "account-recovery",
  };
  console.info("notification_rendered", metadata);
  return { email, metadata };
}
```

Transport stays outside the renderer. That keeps unit tests fast and prevents provider responses from changing copy. Test exact subject and text output, parse HTML to assert one primary action, inject `<`, `&`, and quotes into display fields, and confirm captured logs contain neither token nor seller address. Then run integration tests against the chosen transport's documented event meanings.

Use reserved example addresses and synthetic tokens in preview fixtures. Never paste a live reset URL into an issue, snapshot, or CI artifact. A useful guard scans logs and snapshots for the token fixture and fails if it appears outside the body. Before the boundary, the renderer sees a secret. After it, telemetry sees only `notificationId`.

## Limits and operating rules

No HTML template can force identical dark-mode rendering across clients, and no accepted-send response proves that a seller saw the message. Keep those claims out of dashboards. Use client testing for presentation and lifecycle events for transport.

Email authentication and sender hygiene live outside the template but affect delivery. Google's sender guidance describes authentication and sending practices for mail delivered to personal Gmail accounts. Treat them as deployment requirements, not template features. NIST's digital identity guidance distinguishes authenticator and recovery risks; an email link should not be described as phishing-resistant.

The rule is compact: one copy source, two accessible bodies, a protected preview, and secret-free lifecycle telemetry. For marketplace recovery, that exposes failures without making observability a second credential store.

## Further reading

- Google, Email sender guidelines: https://support.google.com/a/answer/81126
- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
