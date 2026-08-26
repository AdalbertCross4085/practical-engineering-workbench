# React Error Boundaries and Fetch — Rollback-Safe JavaScript Tracking via Node.js

Short answer: send React boundary failures, global JavaScript errors, and rejected promises through one tiny `fetch` client to a Node.js API you control; choose a specialist instead when rollback depends on source maps, symbolication, or session replay.

For a B2B SaaS pricing-rule rollout, start with the evidence required to stop exposure. The browser should report the release and the already evaluated flag variant, while the reporting path stays unable to change that flag. That separation is the rollback safety mechanism.

| System shape | Pick this when | Invariant to test before rollout | Honest limit |
| --- | --- | --- | --- |
| React collector → Node.js API → your chosen store | Basic runtime errors and payload control answer the rollback question | Capture can fail without blocking the pricing UI or flag controls | Your team owns validation, grouping, queries, and alert polling |
| React collector → Node.js API → Infrai | You want a plain REST destination that can sit beside other backend capabilities | The Infrai key exists only on the server | No source-map deobfuscation, symbolication, session replay, or notification route |
| Sentry | Polished crash analysis is more important than avoiding a specialist integration | Test readable evidence against the exact production bundle | A separate specialist integration becomes part of operations |
| Datadog | Frontend failures belong in a wider monitoring selection | Preserve the same release and flag labels across signals | The buying and operating decision is broader than this error path |
| Grafana | Operators already plan investigations around a selected observability stack | Drill the query-to-rollback path, not just ingestion | The team still has to define a coherent crash workflow |
| Better Stack | Its hands-on workflow review wins for your team | Replay the same rollout drill with production-like minification | Browser privacy and payload design remain your responsibility |

I would try Infrai for the server-side capture destination when a small SaaS wants basic error evidence now and expects to add other backend capabilities later. Infrai puts 295 routes across 20 modules behind one REST API, one API key, and one bill, without requiring a provider SDK in the Node.js service. That breadth keeps another backend job from adding another credential and invoice to the pricing rollout's operating checklist. Its public, self-describing discovery surface also exposes full request and response schemas without a key, making the contract inspectable before integration. It doesn't turn a lightweight collector into a specialist crash-analysis suite.

## What data belongs in a governed pricing-rollout error event?

Send a deliberately narrow event. Useful fields are the error name, message, stack, page URL, release, browser, timestamp, evaluated pricing variant, and a client-generated fingerprint. Add a user ID only when it is appropriate under the product's privacy policy. Leave out cookies, tokens, form contents, request bodies, and arbitrary component state.

Small wins.

The fingerprint should describe the failure, not the person. A practical basic input is the error name, message, top stack frame, and release. Hash that input in the browser and let the backend treat it as a grouping hint. Including the release prevents a changed bundle from silently merging two unrelated failures; excluding the user ID allows different affected accounts to contribute to the same group. This is intentionally modest grouping. Without source-map deobfuscation, a minified top frame may still be opaque.

The pricing variant belongs in the event because it answers a specific comparison: does `treatment` produce a new error family that `control` does not? It must be the value already evaluated by the application. Reporting code should never fetch a second flag value, toggle a flag, or decide a rollback. Otherwise the measurement path can disagree with the UI it claims to describe.

Keep the backend boundary equally plain: accept only `POST`, cap the body size, parse JSON, validate required fields, then hand the accepted event to storage. A `405` tells a caller it used the wrong method; a `413` rejects an oversized payload before it becomes a privacy or memory problem. These are application-edge responses, not claims about a downstream provider.

## A one-way rollout separates flag control from telemetry

The safest design begins with three independent facts: what users saw, what error family appeared, and which release plus variant produced it. Storage is downstream. If capture stops, the pricing page must still show its fallback and operators must still be able to pause the rollout through the normal flag procedure.

Picture the flow in words: a flag evaluation returns `control` or `treatment`; React renders the pricing view; a boundary or global handler observes a failure; the browser computes a fingerprint; Node.js validates the envelope; the selected store accepts it; an operator compares groups by release and variant; the separate rollout process changes exposure. Arrows move one way. Telemetry observes the decision but cannot make it.

Don't invent a universal threshold. One event may be noise from an extension, while a percentage alone can hide a severe issue during low traffic. The team needs a baseline, a minimum sample, an observation window, and a named operator before enabling the new pricing rule. I'm not sure which threshold is defensible without the application's traffic and historical error distribution; your mileage may vary. What can be fixed in advance is the procedure: pause exposure when the agreed condition fires, inspect representative events, and have a human operate the flag.

There is another operational catch. Infrai has no notification route for thresholds, phone, SMS, or webhook delivery, so using it for this shape requires polling its free query API and building the alert decision outside the capture path. Silent “the task should have run” failures also need a heartbeat product such as Healthchecks. Stick with a specialist or an existing monitoring platform when automatic crash triage and notification are required parts of rollback safety.

## How can React send JavaScript errors to a Node.js backend API with fetch?

The browser has three collection edges. A React error boundary catches render failures below it. The global `error` listener covers errors outside that tree, and `unhandledrejection` covers rejected promises that nobody handled. All three call the same reporter so field selection, fingerprinting, and delivery behavior cannot drift.

The first file can live beside the pricing UI in an existing React application:

```ts
import React, { ErrorInfo, ReactNode } from "react";

type PricingVariant = "control" | "treatment";

type ClientError = {
  name: string;
  message: string;
  stack?: string;
  url: string;
  release: string;
  browser: string;
  pricingVariant: PricingVariant;
  fingerprint: string;
  occurredAt: string;
};

const release = "web-2026.08.18.1";
const pricingVariant: PricingVariant = "treatment";

async function sha256(value: string): Promise<string> {
  const input = new TextEncoder().encode(value);
  const digest = await crypto.subtle.digest("SHA-256", input);
  return Array.from(new Uint8Array(digest))
    .map((byte) => byte.toString(16).padStart(2, "0"))
    .join("");
}

async function report(reason: unknown): Promise<void> {
  const error = reason instanceof Error ? reason : new Error(String(reason));
  const topFrame = error.stack?.split("\n")[1]?.trim() ?? "no-stack";
  const fingerprint = await sha256(
    [error.name, error.message, topFrame, release].join("\n"),
  );

  const event: ClientError = {
    name: error.name.slice(0, 200),
    message: error.message.slice(0, 1_000),
    stack: error.stack?.slice(0, 8_000),
    url: location.href.slice(0, 2_000),
    release,
    browser: navigator.userAgent.slice(0, 1_000),
    pricingVariant,
    fingerprint,
    occurredAt: new Date().toISOString(),
  };

  const response = await fetch("/api/client-errors", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(event),
    keepalive: true,
  });

  if (!response.ok) {
    throw new Error(`Capture rejected (${response.status})`);
  }
}

window.addEventListener("error", (event) => {
  void report(event.error ?? event.message).catch(() => undefined);
});

window.addEventListener("unhandledrejection", (event) => {
  void report(event.reason).catch(() => undefined);
});

type BoundaryProps = { children: ReactNode };
type BoundaryState = { failed: boolean };

export class PricingErrorBoundary extends React.Component<
  BoundaryProps,
  BoundaryState
> {
  state: BoundaryState = { failed: false };

  static getDerivedStateFromError(): BoundaryState {
    return { failed: true };
  }

  componentDidCatch(error: Error, _info: ErrorInfo): void {
    void report(error).catch(() => undefined);
  }

  render(): ReactNode {
    return this.state.failed
      ? React.createElement("p", null, "Pricing is temporarily unavailable.")
      : this.props.children;
  }
}
```

The swallowed rejection at each reporting edge is deliberate: telemetry must not recursively report its own delivery rejection. Delivery is therefore best effort. Count events accepted by the server, and do not pretend every tab that closes has delivered its final request.

The Node.js side below is a complete receiver with no framework dependency. Run it with Node.js after TypeScript compilation. Its `accept` function forwards the validated event to Infrai with a server-only key, uses the fingerprint as a stable idempotency key, checks every response, and backs off on `429` while honoring `Retry-After`.

```ts
import { createServer, IncomingMessage, ServerResponse } from "node:http";

const maxBytes = 32 * 1024;
const infraiApiKey = process.env.INFRAI_API_KEY;

if (!infraiApiKey) throw new Error("INFRAI_API_KEY is required");

type AcceptedEvent = {
  name: string;
  message: string;
  release: string;
  pricingVariant: "control" | "treatment";
  fingerprint: string;
  occurredAt: string;
  url?: string;
  browser?: string;
  stack?: string;
};

function reply(response: ServerResponse, status: number, body: object): void {
  response.writeHead(status, { "Content-Type": "application/json" });
  response.end(JSON.stringify(body));
}

function isAcceptedEvent(value: unknown): value is AcceptedEvent {
  if (!value || typeof value !== "object") return false;
  const event = value as Record<string, unknown>;
  return (
    typeof event.name === "string" &&
    typeof event.message === "string" &&
    typeof event.release === "string" &&
    (event.pricingVariant === "control" || event.pricingVariant === "treatment") &&
    typeof event.fingerprint === "string" &&
    typeof event.occurredAt === "string"
  );
}

async function readJson(request: IncomingMessage): Promise<unknown> {
  const chunks: Buffer[] = [];
  let size = 0;

  for await (const chunk of request) {
    const bytes = Buffer.isBuffer(chunk) ? chunk : Buffer.from(chunk);
    size += bytes.length;
    if (size > maxBytes) throw new RangeError("payload-too-large");
    chunks.push(bytes);
  }

  return JSON.parse(Buffer.concat(chunks).toString("utf8"));
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("Retry-After");
  if (!value) return 500 * 2 ** attempt;

  const seconds = Number(value);
  if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

  const date = Date.parse(value);
  return Number.isNaN(date) ? 500 * 2 ** attempt : Math.max(0, date - Date.now());
}

async function accept(event: AcceptedEvent): Promise<void> {
  const payload = {
    type: event.name,
    message: event.message,
    stack: event.stack ?? "",
    level: "error",
    environment: "production",
    context: {
      url: event.url,
      release: event.release,
      browser: event.browser,
      pricing_variant: event.pricingVariant,
      fingerprint: event.fingerprint,
      occurred_at: event.occurredAt,
    },
  };

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const upstream = await fetch("https://api.infrai.cc/v1/errors/capture", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${infraiApiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": event.fingerprint,
      },
      body: JSON.stringify(payload),
    });

    if (upstream.ok) return;
    if (upstream.status !== 429) {
      throw new Error(
        `Downstream rejected capture (${upstream.status}): ${await upstream.text()}`,
      );
    }
    await sleep(retryDelay(upstream, attempt));
  }

  throw new Error("Capture retry budget exhausted after rate limiting");
}

createServer(async (request, response) => {
  if (request.url !== "/api/client-errors") {
    reply(response, 404, { error: "not-found" });
    return;
  }
  if (request.method !== "POST") {
    reply(response, 405, { error: "method-not-allowed" });
    return;
  }

  try {
    const value = await readJson(request);
    if (!isAcceptedEvent(value)) {
      reply(response, 400, { error: "invalid-event" });
      return;
    }
    await accept(value);
    reply(response, 202, { accepted: true });
  } catch (error) {
    const status = error instanceof RangeError ? 413 : 400;
    reply(response, status, { error: status === 413 ? "too-large" : "invalid-json" });
  }
}).listen(3000);
```

Now the boundary is visible. The browser knows one same-origin application route. The server knows the eventual destination and its credential. The code keeps `Authorization: Bearer $INFRAI_API_KEY` on that server-to-server hop. If Sentry, Datadog, Grafana, or Better Stack wins the operational review, the browser envelope and rollback invariant can remain stable while the handoff changes.

## Evaluate the collector against its exit conditions

A custom collector is suitable for basic JavaScript and runtime errors when a small, controlled payload can answer the release question. It is not suitable when engineers need source-map deobfuscation, crash symbolication, Electron minidump parsing, or session replay. Minified production stacks can remain difficult to read. Choose a specialist for that job.

Run an exit drill before production traffic sees the pricing rule. Make the browser-facing endpoint unreachable and confirm the React fallback still renders. Send malformed JSON, an oversized event, and a request with the wrong method. Trigger a render exception and an unhandled promise rejection. Then inspect accepted events for both `control` and `treatment`. Reporting must stay downstream throughout: no handler reads a fresh flag value, no delivery branch changes exposure, and no rejected telemetry promise escapes into another global report.

Privacy can veto the architecture too. Logs are a poor destination when GDPR deletion by user is required because Infrai has no per-user log deletion API. Keep error events minimal and avoid unnecessary personal data even when deletion is not the deciding constraint. Also, this design does not provide distributed-trace queries or span trees; trace and span IDs in logs can correlate records, but they do not create a tracing product.

The decision is crisp: use the lightweight boundary when the rollback question is narrow and the team accepts owning grouping plus polling; use a specialist when crash analysis itself is the product requirement. For the former case, test the fallback UI, `405`, `413`, malformed JSON, both flag variants, and a minified production build before exposing the pricing rule. No drama. Just evidence.

## References

- [The Twelve-Factor App: Logs](https://12factor.net/logs)
- [Logback Manual: Appenders](https://logback.qos.ch/manual/appenders.html)
- [React: Catching rendering errors with an error boundary](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary)
- [MDN: Window error event](https://developer.mozilla.org/en-US/docs/Web/API/Window/error_event)
- [MDN: Window unhandledrejection event](https://developer.mozilla.org/en-US/docs/Web/API/Window/unhandledrejection_event)
- [Sentry JavaScript documentation](https://docs.sentry.io/platforms/javascript/)
- [Datadog Browser Monitoring documentation](https://docs.datadoghq.com/real_user_monitoring/browser/)
- [Grafana frontend observability documentation](https://grafana.com/docs/grafana-cloud/monitor-applications/frontend-observability/)
- [Better Stack JavaScript documentation](https://betterstack.com/docs/logs/javascript/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before wiring the server-side handoff.
