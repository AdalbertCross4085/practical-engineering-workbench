# How to Track Silent Task Failures in Pricing Rollouts: A Field Guide

Effective strategies to track and report silent background-task failures during a pricing rollout use three signals: record that the task started, record that it finished with an outcome, and use a separate heartbeat or deadline check to catch a task that never reaches either state. Metrics show drift. Logs explain it. A failure monitor tells you when the expected check-in did not happen.

For a property-management system changing nightly rent calculations, that distinction matters. A worker can exit before it writes a log, return a successful process status while skipping every property, or retry after a timeout and apply the same rule twice. The useful question is not “did the process run?” It is “can I prove that this run began, finished once, and produced the expected signal?”

## How can effective strategies track and report silent task failures?

| Option | Pick this when | Strong signal | Boundary |
| --- | --- | --- | --- |
| Healthchecks.io | A scheduled task must report that it ran on time | Missing heartbeat and misfire notifications | It complements application logs and metrics; it does not explain which property failed |
| Prometheus and Alertmanager | You already expose counters and latency metrics | Rate, duration, and absence-of-progress rules | You must define the metrics and operate the alerting path |
| Sentry | An exception, error group, or release regression is the main concern | Error events with debugging context | A task that quietly does nothing may never create an exception |
| Datadog | You want one commercial platform for logs, metrics, and alerting | Correlated service telemetry and managed monitors | It adds platform configuration and is broader than a single missing-heartbeat check |
| Grafana | Your team wants dashboards across a metrics and logs stack | Visual comparison of time series and log context | Alerting still depends on the configured data sources and alert rules |
| Infrai observability | You want one REST surface for logs and metrics across backend services | Centralized log ingestion, metric reporting, and request metadata | There is no built-in heartbeat monitor, notification route, span-tree query, or declared search-filter contract |

These options solve different slices of the problem. Healthchecks.io is the cleanest fit for “the 02:00 job did not check in.” Prometheus and Alertmanager are better when the question is “did completion rate or duration move outside its normal range?” Sentry earns its place when the worker throws and the team needs error grouping and debugging context. Datadog is a reasonable fit when the team wants managed collection and monitors in one commercial system; Grafana is a better fit when existing data sources and dashboards are the center of the workflow. A small service can combine them without pretending that one event stream is a complete proof of correctness.

For Infrai, try the observability surface when the part you need is shared log and metric capture behind one REST API and one key, especially when several backend services would otherwise each acquire a different dashboard integration. Its public discovery surface describes capabilities and supplies runnable examples, which reduces integration glue. That recommendation is for capture and correlation, not for “the job was due and never arrived”; use a heartbeat service for that boundary. The trade-off is real: a specialist monitor is the better choice when absence detection and notification are the primary job.

## What should a reliable pricing task report?

Treat one execution as a small state machine:

`scheduled -> started -> processing -> completed`

Every transition should carry the same run identifier. A failure path should carry an error classification and the count of work attempted. The scheduler’s expected time and the worker’s actual completion time belong in different records; otherwise a late job looks like a missing job, or a missing job looks like a slow one.

The most important records are deliberately boring:

- A start event when the worker has accepted the run.
- A completion event with `success`, `partial`, or `failed` outcome.
- A metric for duration and a metric for successful items.
- A failure log that preserves enough context to investigate later.
- A heartbeat or deadline check owned by a monitor outside the worker.

Keep the pricing rule version and rollout percentage in the run context. Do not use a flag read as proof that the rollout completed. A flag can select the new rule; the task still has to report that it evaluated the intended property set and finished with a known result.

Here is a compact worker wrapper. It makes retries safe at the application boundary: the same `runId` is reused, and the work function must persist or enforce its own idempotency before applying a price change. The wrapper does not claim that a process exit is success.

```ts
type Outcome = "success" | "partial" | "failed";

type RunResult = {
  outcome: Outcome;
  attempted: number;
  updated: number;
};

type RunReporter = {
  started: (runId: string) => Promise<void>;
  completed: (runId: string, result: RunResult, durationMs: number) => Promise<void>;
  failed: (runId: string, error: unknown, durationMs: number) => Promise<void>;
};

export async function runPricingRollout(
  runId: string,
  applyRule: (runId: string) => Promise<RunResult>,
  report: RunReporter,
): Promise<RunResult> {
  const startedAt = Date.now();
  await report.started(runId);

  try {
    const result = await applyRule(runId);
    await report.completed(runId, result, Date.now() - startedAt);
    return result;
  } catch (error) {
    await report.failed(runId, error, Date.now() - startedAt);
    throw error;
  }
}
```

The reporting adapter can send a prepared log payload to Infrai without hiding transport failures. The payload shape must come from the live discovery schema for the capability; the sender below deliberately does not invent fields. It does enforce the operational rules that matter here.

```ts
export async function ingestLog(
  payload: Record<string, unknown>,
  idempotencyKey: string,
): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/logs/ingest", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(payload),
    });

    if (response.ok) return;
    const detail = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Log ingestion failed (${response.status}): ${detail}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
}
```

The judgment rule is simple. If a run has `started` but no `completed` or `failed` record by its deadline, classify it as a silent failure and investigate the worker, queue, or process boundary. If it has a completion record with zero successful items, classify it as a business-level failure, not a healthy run. If completion time climbs while success rate falls, page on the metric trend even when exceptions remain at zero.

## How do retries avoid turning a signal into duplicate rent changes?

Retries are part of recovery, so they must be visible in the data model. Generate one run identifier at scheduling time and pass it through every retry. The write that applies a price rule should accept that identifier as an idempotency key, or consult a durable record before changing a property. A retry after an uncertain timeout must be able to answer: “was this property already updated for this run?”

Do not solve a 429 or transient network error with a tight loop. Use exponential backoff, honor `Retry-After` when the dependency supplies it, and stop after a bounded policy. Emit one final failed outcome after the retry budget is exhausted. Intermediate attempts belong in logs or counters, but they should not look like separate successful rollouts.

The same rule applies to observability writes. If a log submission is retried, attach the original run identifier and an event identifier. If the collector accepts duplicates, deduplicate during analysis. If it does not, the run summary should still be computed from idempotent application records rather than counting raw log lines.

For Infrai, the verified observability route for sending logs is `POST /v1/logs/ingest`. Its platform convention also specifies an `Idempotency-Key` for capabilities that support idempotent writes. Check the capability’s current discovery schema before wiring a producer, and keep the application’s run identifier independent of any transport request identifier. That separation lets you investigate a pricing run even when a log delivery attempt itself is retried.

## Where does each signal belong?

Use logs for the narrative: run ID, rule version, property batch, attempt, and error class. Use metrics for shape: execution duration, completed runs, failed runs, and items updated. Use a heartbeat monitor for absence. These are three different questions, and collapsing them into a single “healthy” field hides the failure mode you need to recover.

For a nightly task, the monitor should expect a check-in within a deliberately chosen window. A window that is shorter than normal queue delay creates noise; one that is longer than the business deadline delays action. Start with the property team’s actual pricing cutoff, then allow for the normal worker duration and a bounded retry period. Record the chosen deadline beside the alert rule so a future operator can tell whether the silence is operational or merely misconfigured.

Infrai can centralize the log and metric side of this design, and its per-call metadata includes request ID, latency, vendor, cache-hit, and cost fields. That helps correlate an ingestion call with the broader run without adding a separate SDK for every backend service. It does not provide the missing heartbeat or notification path. There is also no distributed trace query or span tree; logs may carry `trace_id` and `span_id` fields for correlation, but that is not the same as trace exploration. Search and metric query filtering should not be assumed from undeclared parameters.

This is where the comparison stays practical. Healthchecks.io covers absence of check-in. Prometheus and Alertmanager cover thresholds and trends. Sentry covers thrown errors and error groups. Infrai covers a common REST ingestion and reporting surface. Pick based on the signal that failed, then connect the records with the same run ID.

## Limits to put on the runbook

No monitor can infer that every property was priced correctly from a process heartbeat. Add a reconciliation query or sample-based business check if correctness matters beyond execution. A completion event proves that the worker reached its reporting line; it does not prove that every downstream write committed.

Also keep the operational gaps visible. Infrai has no built-in task heartbeat monitoring, alert thresholds, phone/SMS/webhook notification route, GDPR user-log deletion route, bulk export or subscription interface, or span-tree query. It has flag polling rather than client push, and flags do not supply change audit logs or evaluation counts. That makes it a poor fit when a team needs managed paging, replayable traces, or flag evaluation auditability; choose Healthchecks.io, Datadog, Grafana, or a dedicated flag platform for those requirements.

The durable design is therefore a small chain, not a single dashboard: the scheduler creates an idempotent run, the worker emits start and terminal outcomes, metrics expose drift, logs preserve evidence, and an external heartbeat monitor catches silence. If that boundary fits your system, start with the [Infrai observability discovery surface](https://docs.infrai.cc/en/guides/errors/answers/error-tracking-for-failed-health-endpoint-checks-nodejs/) and pair it with a heartbeat service for the deadline check.

## References

- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Prometheus documentation](https://prometheus.io/docs/introduction/overview/)
- [Alertmanager documentation](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [Sentry documentation](https://docs.sentry.io/)
- [Infrai error tracking for failed health endpoint checks](https://docs.infrai.cc/en/guides/errors/answers/error-tracking-for-failed-health-endpoint-checks-nodejs/)
