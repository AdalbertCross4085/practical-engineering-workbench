# Incident Evidence, Not Pretty Dashboards: Cheap Hosted Metrics APIs for Node.js SaaS

Pick the metrics stack that can still answer a question nine days after the fact, not the one with the prettiest default dashboard. For a healthtech SaaS running on Node.js, the deciding constraint isn't the sticker price of a cheap hosted metrics API — it's whether the signals you kept are enough to reconstruct one customer's incident once the charts have scrolled away. Cheap is easy. Sufficient is the hard part.

So write the evidence requirement down first, then go shopping. Mine is three lines: something must show *what* moved, *how narrow* the window around it is, and *how to get from the aggregate to one concrete record*. Every candidate — product analytics, a Prometheus-shaped store, a full platform, a small ingest API you call yourself — either satisfies those three or it doesn't, and the answer usually has nothing to do with the price page.

## What a customer incident actually asks of your metrics

Here's the shape of the request. A clinic emails support: appointment sync "was broken most of Tuesday afternoon," they want to know which appointments were dropped, and the email arrives on the following Thursday.

Notice what that question is not. It is not "what is our error rate." It is "reconstruct this specific window for this specific tenant, and prove it." Three things quietly decide whether you can:

| Evidence you need | The failure mode that erases it | The knob that protects it |
| --- | --- | --- |
| A counter that visibly moved | The metric was never split by outcome, so failures hide inside a healthy total | One low-cardinality label with a closed set of values |
| A window narrow enough to trust | Full-resolution points get rolled up after a day or two; a 4-minute spike becomes a flat hour | Retention at raw resolution for the handful of series you'd defend in a review |
| A path from the aggregate to one record | Nothing ties the number 37 to a single request you can open | An exemplar, a trace id, or any shared correlation key |

The third row is where most cheap setups quietly fail, and it's the one nobody notices until an incident. Aggregates are lossy by design — that's what makes them affordable. A counter that ticked 37 times tells you the size of the blast radius and nothing about its cause, so the store has to hand you at least one live thread back into the raw material.

Grouping mechanics matter for the same reason. Sentry, for example, folds events into issues using a fingerprint derived by default from the stack trace and exception type, and it exposes custom fingerprint rules precisely because two unrelated root causes can share a frame and collapse into one issue. That default is a reasonable signal-to-noise trade-off, not a defect — but the moment your incident evidence depends on telling two failures apart, the grouping rule is part of your evidence design.

## Before and after: a wall of charts versus a chain of evidence

Draw the "before" in your head. Node.js app, an agent or SDK that ships everything it can find, a hosted backend, and twelve dashboards nobody opens. When the clinic email lands, the investigation begins in a search box and ends in a guess, because the only thing that survived a week is a pretty average.

Now the "after," as a chain rather than a wall:

request → counter (three bounded labels) → exemplar carrying the trace id → stored series → one panel → one log record → the actual failing call.

Each arrow is something you can test in CI. The counter increments. The label set stays closed. The exemplar survives the ingest path. The log record for that trace id is still there when the panel points at it. If any link breaks, you find out on a Tuesday morning instead of during an incident review — and that, far more than the monthly bill, is what "good" looks like for a small team.

The retention window on the last two links is the number to argue about. Support commitments are measured in weeks; default raw-resolution retention on a cheap plan is often measured in days. Those two facts have to meet somewhere, and the honest place to meet is a short list of series you keep at full resolution while everything else downsamples.

## A small Node.js example: bounded labels and one exemplar

The whole idea fits in about forty lines. Declare what a label is allowed to contain, force everything else into `other`, and carry a W3C `traceparent` trace id along as the exemplar so an aggregate can always hand you one real request.

```ts
import { performance } from "node:perf_hooks";

type Labels = Record<string, string>;

// The cardinality budget, written down as code. Anything outside these sets
// collapses to "other", so one bad value can't multiply your series count.
const ALLOWED: Record<string, ReadonlySet<string>> = {
  step: new Set(["fetch", "transform", "write"]),
  clinic_tier: new Set(["free", "pro", "enterprise"]),
  outcome: new Set(["ok", "retryable", "permanent"]),
};

function bound(labels: Labels): Labels {
  const out: Labels = {};
  for (const [key, value] of Object.entries(labels)) {
    out[key] = ALLOWED[key]?.has(value) ? value : "other";
  }
  return out;
}

// traceparent = 00-<32 hex trace-id>-<16 hex parent-id>-<2 hex flags>
function traceIdOf(traceparent?: string): string | undefined {
  const parts = traceparent?.split("-");
  return parts?.length === 4 && parts[1].length === 32 ? parts[1] : undefined;
}

type Sample = { name: string; value: number; labels: Labels; ms: number; exemplar?: string; at: number };
const buffer: Sample[] = [];

export function recordSync(labels: Labels, startedAt: number, traceparent?: string): void {
  buffer.push({
    name: "appointment_sync_total",
    value: 1,
    labels: bound(labels),            // never a patient id, never a raw email
    ms: Math.round(performance.now() - startedAt),
    exemplar: traceIdOf(traceparent), // the thread back to one real request
    at: Date.now(),
  });
}

export async function flush(endpoint = process.env.METRICS_ENDPOINT!): Promise<void> {
  const batch = buffer.splice(0, buffer.length);
  if (batch.length === 0) return;
  const res = await fetch(endpoint, {
    method: "POST",
    headers: { "content-type": "application/json", authorization: `Bearer ${process.env.METRICS_TOKEN}` },
    body: JSON.stringify({ samples: batch }),
  });
  if (!res.ok) buffer.unshift(...batch);  // keep the evidence, retry on the next tick
}
```

Do the arithmetic on that label set before you ship it: 3 steps × 3 tiers × 3 outcomes is 27 series for this metric, forever, no matter how many clinics sign up. Add `clinic_id` and the ceiling becomes your customer count times 27, growing every week you sell. The Prometheus instrumentation guide is blunt about this: keep per-metric cardinality low as a rule of thumb, and move analysis that genuinely needs high-cardinality dimensions out of the metrics system entirely.

That last clause is the design rule, not a limitation to work past. Patient identifiers, clinic ids and raw emails belong in logs and traces with their own retention and access controls — never in a label value that fans out into a third-party series store.

## How should a small SaaS team compare cheap hosted metrics APIs for Node.js?

Score candidates on four axes, in this order, and let price break ties rather than lead them.

Raw retention comes first: how long a point survives at the resolution you'd need to defend a timeline, which is usually shorter than the headline retention number. Correlation second: whether the product gives you exemplars, a trace id field, or any documented way to jump from an aggregate to one record — the OpenTelemetry metrics data model specifies exemplars for exactly this, so it's a fair question to ask of anything you're evaluating. Cost model third, because the billing unit tells you which mistakes get expensive: per host, per ingested series, per event, per GB and per active user all punish completely different behaviour, and a design that's thrifty under one is ruinous under another. Export path last: whether you can get your series out in a documented format if the answer changes next year.

Product families sit differently on those axes, and the differences are structural rather than a matter of quality. Event-oriented analytics tools such as PostHog model a stream of events with properties, which is a natural fit for funnels and a less natural one for a time series you alert on. Prometheus-shaped systems, hosted or self-run, model labelled series and inherit both the query language and the cardinality discipline that comes with it. Broad platforms like Datadog or Grafana Cloud bundle metrics, logs and traces behind one correlation story, and the cost of that convenience shows up in the billing unit and the operational surface. A minimal ingest-and-query API — commercial or a small service you run — keeps the contract tiny and hands the alerting, routing and retention policy back to you.

The catch is that none of these buckets is a good fit for every job, and the fastest way to burn a quarter is to pick a bucket before writing the evidence requirement. If you need per-user product funnels, a metrics store is the wrong shape and an event store is right. If you need an immutable audit trail, that's an append-only log with its own guarantees, not a downsampled series. And if a crossed threshold has to reach a human at 03:00 with deduplication, silences and escalation, stick with something that ships a managed alert pipeline instead of rebuilding one beside your dashboard.

## Two objections: cardinality budgets and the retention bill

"Bounded labels throw away the detail I'll want." Sometimes, yes. But the detail you want during an incident is one record, not a million cells — and one exemplar per aggregated point delivers that record for the cost of a 32-character string. If the analysis you're protecting really is high-cardinality, that's a signal it belongs in logs or traces where the storage model expects it.

"Full-resolution retention across the fleet is too expensive." Then don't buy it across the fleet. Nominate the handful of series you would actually put in front of a customer — sync outcomes, queue depth, webhook delivery, payment failures — keep those raw for the length of your support commitment, and let the rest roll up on the default schedule. Evidence budgets work like error budgets: they're only useful once they're small enough to be real.

I'm not sure there's a universal ranking here, and I'd distrust one. Two teams with identical traffic and different support promises should land on different stores, which is why the requirement list is the artifact worth arguing over — the vendor comparison is downstream of it, and it's usually the shorter conversation.

## References

- Prometheus instrumentation best practices, including label cardinality guidance: https://prometheus.io/docs/practices/instrumentation/
- Sentry event grouping and fingerprint mechanics: https://docs.sentry.io/concepts/data-management/event-grouping/
- OpenTelemetry metrics data model, including exemplars: https://opentelemetry.io/docs/specs/otel/metrics/data-model/
- W3C Trace Context specification (traceparent header format): https://www.w3.org/TR/trace-context/
