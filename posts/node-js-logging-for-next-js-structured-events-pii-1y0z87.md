# Node.js Logging for Next.js: Structured Events, PII Controls, and Endpoint Delivery

**Short answer: emit a small structured JSON event from each Next.js server boundary, remove PII before transport, and choose standard output, direct HTTPS, or a queue according to the amount of log loss and request latency the application can tolerate.**

Start with the least complex transport that meets the operational need. For many deployments, that is JSON on standard output. A direct log endpoint is useful when the runtime does not provide a suitable collector. A queue belongs in the design only when delivery needs justify another moving part.

| Delivery path | Pick it when | Main trade-off |
| --- | --- | --- |
| JSON to standard output | The runtime already collects process output | Collection, buffering, and retention belong to the runtime environment |
| Direct HTTPS | Volume is modest and one owned endpoint is the integration boundary | Every request can add latency, and transient delivery failure needs a policy |
| Queue plus worker | Bursts are normal and log delivery must be isolated from user requests | The queue, worker, retries, and backlog become operational concerns |

The invariant is more important than the transport: application code creates the same safe event either way. Picture the path in words: Server Action or route handler -> event builder -> redactor -> transport adapter -> collector. PII must disappear before the final arrow.

## Pick standard output when the platform already collects it

Standard output keeps the application side wonderfully small. Serialize one event per line, let the runtime attach its own timestamp if appropriate, and keep human prose out of the payload. A JSON line can be parsed; a sentence assembled from six interpolated values eventually becomes a regular-expression maintenance job.

This choice fits a single deployment environment with a known collection path. It also makes local inspection easy. The catch is ownership: the application team still needs to verify how output is buffered, what happens during process shutdown, where records are retained, and who may search them. Those answers don't come from `process.stdout.write()`.

Use one schema across Server Actions, API routes, background work, and authentication boundaries. Useful fields include an event name, severity, request ID, service, deployment version, and a compact result such as `ok` or `rejected`. Raw request bodies, cookies, authorization headers, email addresses, prompts, and response text do not belong in the default event.

Keep it boring.

## Pick direct HTTPS when the log endpoint is the boundary

A direct sender is easy to explain and easy to test. It accepts an already sanitized event, serializes it once, and posts it to the complete URL held in `LOG_ENDPOINT_URL`. Keeping the URL in configuration avoids baking a guessed route contract into the logger.

The sender needs a short timeout and an explicit failure policy. Logging usually should not turn a successful Server Action into a failed user operation, so the example below writes a minimal delivery diagnostic to standard error and returns. That policy is not universal. An audit record with a legal retention requirement may need durable delivery, admission control, or rejection of the business action instead. Name the class of event before choosing the behavior.

Direct delivery is not suitable when a burst can create one outbound request per hot-path event, or when the collector's latency would consume the route's response budget. In those cases, stick with collected standard output or move delivery behind a queue. Don't hide the coupling under a generic `logger.info()` method; the coupling still exists.

## Pick a queue when isolation is worth operating a worker

A queue moves endpoint latency and retry work away from the request. The handler publishes the safe event, a worker drains events in batches, and the endpoint receives traffic at a controlled rate. This is the strongest option here for absorbing bursts.

It is also the most expensive option in engineering attention. Someone must observe enqueue failures, queue depth, oldest-message age, worker health, poison events, and retention. A queue can preserve records while quietly increasing the time before they are searchable. If incident responders expect a fresh event in seconds, that expectation needs a measured service objective rather than hope.

Feature flags are useful during this rollout. Fowler's feature-toggle guidance distinguishes deployment from release and warns that toggles add carrying cost; the same principle applies to switching a new transport on gradually. Keep transport selection outside the event builder, test both paths, and remove the migration flag after the change settles. The custom-appender model documented by Logback is a useful cross-ecosystem analogy too: formatting an event and delivering it are separate responsibilities, even though this implementation is TypeScript rather than Java.

## How should Next.js Server Action and API route logging send structured JSON to an endpoint?

Build an allowlisted event first, then recursively redact it as a second line of defense. The allowlist prevents a caller from passing an entire request by convenience. The redactor catches a sensitive key or token that slips into nested metadata. Both layers matter — and neither requires a logging SDK.

This focused TypeScript module supports standard output and direct HTTPS. It deliberately excludes arbitrary error messages because thrown messages can contain submitted values. Record a stable error code instead. The endpoint value must be a complete, reviewed URL supplied by the deployment environment.

```ts
type Level = "debug" | "info" | "warn" | "error";

type SafeScalar = string | number | boolean | null;
type SafeValue = SafeScalar | SafeValue[] | { [key: string]: SafeValue };

type LogEvent = {
  timestamp: string;
  event: string;
  level: Level;
  request_id: string;
  service: string;
  version: string;
  outcome: "ok" | "rejected" | "failed";
  error_code?: string;
  metadata?: Record<string, SafeValue>;
};

const sensitiveKey = /^(authorization|cookie|email|name|password|phone|prompt|response|token|body)$/i;
const emailValue = /\b[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}\b/gi;
const bearerValue = /\bBearer\s+[A-Za-z0-9._~+/=-]+\b/gi;

function redact(value: SafeValue): SafeValue {
  if (typeof value === "string") {
    return value
      .replace(emailValue, "[redacted-email]")
      .replace(bearerValue, "[redacted-token]");
  }

  if (Array.isArray(value)) {
    return value.map(redact);
  }

  if (value !== null && typeof value === "object") {
    return Object.fromEntries(
      Object.entries(value).map(([key, nested]) => [
        key,
        sensitiveKey.test(key) ? "[redacted]" : redact(nested),
      ]),
    );
  }

  return value;
}

function createEvent(input: {
  event: string;
  level: Level;
  requestId: string;
  outcome: LogEvent["outcome"];
  errorCode?: string;
  metadata?: Record<string, SafeValue>;
}): LogEvent {
  return redact({
    timestamp: new Date().toISOString(),
    event: input.event,
    level: input.level,
    request_id: input.requestId,
    service: "web",
    version: process.env.DEPLOYMENT_VERSION ?? "unknown",
    outcome: input.outcome,
    ...(input.errorCode ? { error_code: input.errorCode } : {}),
    ...(input.metadata ? { metadata: input.metadata } : {}),
  }) as LogEvent;
}

function writeJsonLine(event: LogEvent): void {
  process.stdout.write(`${JSON.stringify(event)}\n`);
}

async function sendJson(event: LogEvent): Promise<void> {
  const endpoint = process.env.LOG_ENDPOINT_URL;
  if (!endpoint) {
    writeJsonLine(event);
    return;
  }

  try {
    const response = await fetch(endpoint, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify(event),
      signal: AbortSignal.timeout(1_500),
    });

    if (!response.ok) {
      process.stderr.write(
        `${JSON.stringify({ event: "log.delivery_failed", status: response.status })}\n`,
      );
    }
  } catch (error: unknown) {
    const reason = error instanceof Error ? error.name : "UnknownError";
    process.stderr.write(
      `${JSON.stringify({ event: "log.delivery_failed", reason })}\n`,
    );
  }
}

export async function logServerEvent(
  input: Parameters<typeof createEvent>[0],
): Promise<void> {
  await sendJson(createEvent(input));
}
```

A Server Action can create a request ID at its trusted boundary and record only the result fields needed for operations. It should await direct delivery only if that latency belongs in the action's budget.

```ts
"use server";

import { logServerEvent } from "./server-log";

export async function updatePreferences(
  preferenceCount: number,
): Promise<{ requestId: string }> {
  const requestId = crypto.randomUUID();

  await logServerEvent({
    event: "preferences.updated",
    level: "info",
    requestId,
    outcome: "ok",
    metadata: { preference_count: preferenceCount },
  });

  return { requestId };
}
```

An API route can reuse the exact event contract. Notice what is absent: the request object, headers, URL query, and submitted JSON. Add individual fields only after deciding they are operationally necessary and non-personal.

```ts
import { logServerEvent } from "./server-log";

export async function POST(request: Request): Promise<Response> {
  const requestId = request.headers.get("x-request-id") ?? crypto.randomUUID();

  await logServerEvent({
    event: "api.operation.completed",
    level: "info",
    requestId,
    outcome: "ok",
    metadata: { method: "POST" },
  });

  return Response.json({ ok: true, requestId });
}
```

Consider a preference update that appears successful in the browser but does not produce the expected downstream behavior. The request ID returned to the browser gives support one safe handle. An operator searches that ID and finds `preferences.updated` with the deployment version, the `ok` outcome, and `preference_count: 3`; no email address or submitted choices are exposed. If the companion API operation is absent, the operator has learned where the execution path stopped without opening a raw request. If both events exist, their timestamps and versions narrow the next check to the downstream system. Now reverse the privacy decision: imagine that the logger had captured the whole form because it seemed faster during development. Search would reveal more immediately, but every replica, export, retention policy, and support permission would now govern customer preference data too. The extra context changes the security boundary of the logging system. A small schema forces the better debugging move: preserve identifiers and outcomes in logs, keep business data in its system of record, and reproduce the operation through an access-controlled path. This walkthrough is also a useful acceptance test. Give an engineer only the request ID and the approved operational tools. If the safe fields cannot isolate the failing boundary, add one narrow field; don't respond by logging the request.

Ship the schema first.

Test the redactor with nested objects, arrays, mixed-case keys, email-like strings, and bearer tokens. Test the transport separately with a fake `fetch`, including a timeout and a `429` response. Then inspect a real emitted line in a non-production environment. I'm not sure which fields your incident responders truly need; a short incident exercise will answer that better than adding speculative context.

One more guardrail helps: set a maximum serialized event size at the transport boundary. Oversized events usually signal that an allowlist widened or a payload slipped into metadata. Reject or trim the event according to a documented policy, and count that outcome without printing the rejected content.

## Limits and operational checks

Redaction is not data governance. Patterns miss novel identifiers, while broad patterns can erase useful values. Review the schema, restrict access to the collector, define retention, and test deletion procedures independently. Your mileage may vary across jurisdictions and data classifications, so privacy and security owners need to approve the actual field set.

Structured logs are not traces or metrics either. A request ID helps correlate events, but it does not encode parent-child timing. Counters can reveal a delivery-failure trend without forcing responders to search every event. Use each signal for the question it answers.

Before release, verify five things: the same schema leaves both server boundaries; sensitive fixtures never appear in captured output; missing endpoint configuration follows the intended fallback; endpoint slowdown stays inside the chosen latency budget; and operators can find an event by request ID. This is also where the decision table becomes concrete. If direct HTTPS fails the latency test, don't add retries inside the route and hope. Change the transport.

No transport is the universal winner. Standard output is unsuitable without a trustworthy collector. Direct HTTPS is unsuitable for heavy bursts or strict isolation. A queue is unsuitable when the team cannot operate and observe its backlog. Pick the smallest system whose failure mode the team can explain at 03:00.

## References

- https://martinfowler.com/articles/feature-toggles.html
- https://logback.qos.ch/manual/appenders.html

## Further reading

The two references above cover gradual rollout with feature toggles and the appender boundary between event creation and delivery. They are useful design companions even when the application code and transport are written in TypeScript.
