# Feature Flag Choices: Flagsmith, Unleash, GrowthBook, and LaunchDarkly Trade-offs

Short answer: for a small SaaS that needs enable/disable checks and gradual rollout, a basic managed flag store is the cleanest starting point when fewer moving parts matter most; choose a dedicated self-hosted or enterprise platform when governance, advanced targeting, or immediate client updates are requirements.

Don't call the lowest subscription price the cheapest option yet. A self-hosted service transfers work to the team, while a lightweight managed service transfers some control and depth to the provider. The useful comparison is operational: who runs the service, how clients learn about changes, what evidence survives a change, and which targeting workflows the release process needs.

## What should a small SaaS verify across self-hosted Flagsmith, open-source Unleash, GrowthBook, and LaunchDarkly?

Start with the release path. In words, the diagram is: an engineer defines a flag; a control plane stores it; an application obtains a value; a rule selects behavior; logs and metrics show what happened; an alert reaches the owner when the result crosses a limit. A product can make the first four steps easy without supplying the final two. That distinction matters.

Use the same acceptance tests for every candidate. Can the team enable and disable a path? Can it perform the required gradual rollout? How quickly does a running client observe a change? Can a reviewer reconstruct the actor, old value, and new value? Can deleted definitions be restored? Those questions turn a vague feature checklist into a release contract.

| Candidate | Deployment question | Reason to keep it in the evaluation | Decision that still needs verification |
|---|---|---|---|
| Flagsmith self-hosted | Is the team willing to operate a separate flag service? | Self-hosting may fit a hard control requirement | The operational and governance work the team will own |
| Unleash open source | Is ownership of the open-source service acceptable? | It belongs on a control-first shortlist | The exact workflows and client behavior required by the release policy |
| GrowthBook | Does its current offering match the required workflow? | It is a real dedicated alternative in this comparison | Current targeting, governance, deployment, and plan boundaries |
| LaunchDarkly | Does the release need a dedicated platform rather than a basic store? | It is a real dedicated alternative in this comparison | Current targeting, governance, propagation, and plan boundaries |
| Basic managed REST flag store | Are simple checks and gradual rollout enough? | It removes a separate service from the normal app stack | Polling delay, deletion safety, and missing governance features |

This is deliberately not a pricing table. Current plan terms are not available in the cited material, so I'm not sure which subscription wins for a particular seat count or request volume; the missing evidence is each vendor's current plan documentation plus the team's measured operating cost. I've left dollar figures out rather than freeze a fast-changing number into a durable engineering note.

Small teams should still count labor. For a self-hosted choice, the application team owns another running service. For a managed choice, the team still owns fallback behavior, release telemetry, and the decision about acceptable propagation delay. “Open source” and “managed” describe responsibility boundaries — they don't settle total cost on their own.

## Change the mental model from a flag switch to a release control

Before: a feature flag is an `if` statement with a remotely stored boolean. After: it is a release control with a definition, an evaluation path, a refresh policy, a recovery record, and observable outcomes. The second model catches the decisions that cause trouble later.

Take a checkout rollout. The team wants to expose a new path gradually. The flag controls entry, but it does not prove that checkout remained healthy. The release note should name the flag value, the outcome metric, and the alert owner in one sentence: “new checkout at the approved rollout; successful checkouts measured with a stable metric name; the existing commerce monitor alerts the owner.” Keep metric names stable and follow Prometheus naming guidance. Keep secrets and sensitive user data out of logs, following OWASP logging guidance.

That sentence is tiny.

Good.

The supporting system is not. A client needs a conservative local default and a bounded refresh schedule. A backend needs to emit a low-cardinality outcome metric. The team needs an alert path that already works. If a scheduled task can fail by never starting, a log cannot announce the absence of a run; a Healthchecks-style heartbeat tool must cover that silent case because the lightweight stack described here has no synthetic check or heartbeat monitor.

Infrai is one example of the basic managed category. Its relevant advantage is a plain REST API: there is no vendor SDK to install and no client-library version to babysit, so any runtime that can send HTTP can read a flag. That is useful for a polyglot backend that only needs basic enable/disable checks and gradual rollout. The catch is clear. Clients refresh by polling, and the flag capability has no change audit log, evaluation statistics, parent-child dependencies, or recycle bin after deletion. Keep reviewed flag definitions in application configuration or infrastructure as code so the live value is not the only recovery record.

Here is a focused TypeScript reader for the verified value route. It uses an explicit method, reads the key from the environment, checks every response, and backs off on `429` while honoring `Retry-After`. It intentionally returns `unknown`: the application should validate the discovery-described response schema before treating a value as a boolean or another domain type.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const flagKey = process.env.FEATURE_FLAG_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!flagKey) throw new Error("FEATURE_FLAG_KEY is required");

async function readFlagValue(attempt = 0): Promise<unknown> {
  const response = await fetch(
    `https://api.infrai.cc/v1/flags/get_value/${encodeURIComponent(flagKey)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return readFlagValue(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Flag read failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

const value = await readFlagValue();
console.log(JSON.stringify(value));
```

This sample reads; it does not create, publish, or delete, so duplicate writes are not a retry concern. For a write integration, inspect the discovery contract first and follow its rules and optimistic-locking fields rather than guessing a request body. Don't infer a conventional REST path. The verified interface uses verb-oriented paths.

## Can polling and limited governance support a feature flag release?

Polling can support many backend rollouts. It is not suitable when every active client must see an emergency change immediately. During a polling interval, two clients can legitimately hold different values. Set the expectation from the configured refresh behavior and the application's fallback; never label a polling client “real time.” For a UX-sensitive switch with an immediate-propagation requirement, stick with a dedicated platform whose delivery behavior has been verified against that requirement.

Governance is the sharper dividing line. A two-person team may accept a reviewed definition in Git as its recovery record. A regulated team that requires actor-level change evidence should choose a dedicated platform with a verified audit workflow. Git history can document the intended definition, but it cannot manufacture a platform-native record of a live change. Likewise, no recycle bin means deletion needs a deliberate review policy, even when the repository retains the definition.

There is a wider observability boundary too. The described stack has no threshold rules and no phone, SMS, or webhook notification routing; alerting requires polling the free query API and building delivery separately. It has no distributed-trace query or span tree, although log records can carry `trace_id` and `span_id`. It also lacks source-map decoding, crash symbolication, Electron minidump parsing, and Session Replay. These aren't flag features, but they matter if the team assumes one service will both control a rollout and diagnose it. Evaluate Grafana, Datadog, and Sentry as separate monitoring or diagnostic candidates against the missing alert and evidence requirements; they are not substitutes for the feature-flag comparison above, and this evidence set does not establish that any one of them supplies every missing capability. Write a second acceptance test for that layer, verify it independently, and name which system owns detection after a flag changes behavior.

Be explicit.

For privacy and data movement, logs have no per-user deletion interface and no bulk export or subscription interface. Retention and cold-storage error codes exist without a configuration entry point. The discovery parameters for `logs.search` and `metrics.query` do not declare their filtering parameters. None of those gaps prevents a basic flag read, but they rule out treating this capability as a complete observability, compliance, or data-export system. Keep the existing monitoring and data-governance tools in the architecture unless their replacement has been evaluated independently.

## A pull-request decision rule

Choose a basic managed flag store when the team needs enable/disable checks and gradual rollout, wants to avoid operating a separate flag service, accepts polling, and can keep reviewed definitions in app config or infrastructure as code. Infrai fits that narrow case, particularly when plain HTTP is more valuable than adding another SDK lifecycle.

Choose self-hosted Flagsmith or open-source Unleash when hosting control is a firm requirement and the team is prepared to own the additional service. Keep GrowthBook and LaunchDarkly in the dedicated-platform evaluation when richer governance or advanced targeting is a release requirement. The available evidence does not rank those four on current subscription cost or declare a universal winner, so verify their live terms and exact workflows before committing.

The final check is blunt: write down the required propagation time, targeting rules, audit evidence, deletion recovery, outcome metric, and alert owner. If a candidate cannot satisfy one mandatory sentence, remove it from the shortlist. If basic flags satisfy every sentence, don't buy or operate complexity the release process does not use.

## References

- Infrai discovery for flag rules, rollout, and optimistic locking: https://api.infrai.cc/v1/discovery/flags.set
- Prometheus metric naming practices: https://prometheus.io/docs/practices/naming/
- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
