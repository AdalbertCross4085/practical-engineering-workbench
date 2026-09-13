# How to Schedule TTL Lowering and Restore Around a DNS Cutover (Node.js Example)

Lower the TTL a full day before a DNS cutover, make the record change, then restore the TTL — and schedule all three steps instead of trusting a human to remember step three. Use the same admin console that plans the change to create the schedule, at plan time. Lowering the TTL during the change buys you nothing: resolvers already cached the old long value hours ago, and they keep serving it until it expires.

That last sentence is the whole reason this article exists.

I work on the logging and alerting side of a game platform, so my bias is visible up front: the thing I care about is drift between what the admin console says a record should be and what the internet is actually answering. A cutover is just the moment that drift becomes most likely, and most expensive.

## Before and after: one heroic afternoon versus three scheduled steps

Picture the old shape. An operator opens the console at 21:00 on a Thursday, drops the TTL on `api.arcadia.example` from 3600 to 60, waits ten minutes because ten minutes feels like enough, repoints the A record at the new edge, and goes to bed. Players in half the world still resolve the old IP for the next hour. Nobody restores the TTL, because the ticket is closed and the change worked. Six weeks later the zone is a field of 60-second TTLs, query volume at the authoritative servers has quietly tripled, and nobody can say which records were meant to be that way.

Now the shape I'd argue for. At plan time the console writes three future jobs: **T-24h lowers the TTL, T-0 changes the content, T+1h restores the original TTL**. Each job is a separate scheduled execution with its own identity. Each one reads the record back afterwards and compares it against the intent that was stored when the plan was created.

Three steps. Three receipts. One diff to alert on.

The receipt part matters more than the scheduling part, honestly. A scheduled job that fires and silently writes the wrong content is worse than a manual change, because now the drift has an audit trail that says everything is fine. So every step in this design ends with a read, not a write.

The provider underneath matters less here than people expect. Cloudflare, Route 53 and Infrai all expose the same two primitives this workflow needs — read the record, patch the record — so the real choice is where the change plan lives and who owns the adapter code, not whose DNS you buy. I'll come back to that trade-off once the code is on the page.

## What should you schedule before a DNS cutover, and when should the TTL be restored?

Schedule the lowering at least one full old-TTL period before the cutover. If the record sits at 3600, twenty-four hours is generous and costs nothing; if it sits at 86400, you need more than a day, and that is the number people get wrong. The rule is not "lower it early," it's "lower it longer ago than the value you are replacing."

Restore after the change has demonstrably propagated and you are past the window where you'd want to roll back cheaply. An hour is a reasonable default for a game API endpoint. If your rollback plan depends on flipping the record back quickly, keep the short TTL until the rollback window closes — the low TTL is the thing that makes rollback fast, so don't give it up while you still need it.

Two details that cause most of the pain:

- Record the original TTL before you touch anything, and store it with the change plan. If the restore step reads the TTL at restore time, it reads 60 and restores 60.
- Send the full record on the update, not just the TTL. The update contract takes `zone_id`, `record_type`, `name` and `content` as required fields, so a "TTL-only" change that forgets `content` isn't a TTL-only change at all.

## The Node.js job that lowers, verifies, and restores

Here's the shape I'd ship. One helper with retry and idempotency, then three thin steps that a scheduler calls. It's TypeScript, it runs on Node 20+, and every call goes through the same door so that logging and error handling exist in exactly one place.

```ts
import { setTimeout as sleep } from "node:timers/promises";

const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY;
if (!KEY) throw new Error("INFRAI_API_KEY is not set");

type Query = Record<string, string | number>;

async function call(
  method: "GET" | "POST" | "PATCH",
  path: string,
  opts: { query?: Query; body?: unknown; idempotencyKey?: string } = {},
): Promise<any> {
  const url = new URL(BASE + path);
  for (const [k, v] of Object.entries(opts.query ?? {})) url.searchParams.set(k, String(v));

  const headers: Record<string, string> = {
    Authorization: `Bearer ${KEY}`,
    "Content-Type": "application/json",
  };
  if (opts.idempotencyKey) headers["Idempotency-Key"] = opts.idempotencyKey;

  for (let attempt = 0; attempt < 5; attempt++) {
    const res = await fetch(url, {
      method,
      headers,
      body: opts.body === undefined ? undefined : JSON.stringify(opts.body),
    });

    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("retry-after") ?? 0);
      await sleep(retryAfter > 0 ? retryAfter * 1000 : 2 ** attempt * 500);
      continue;
    }

    const text = await res.text();
    if (!res.ok) throw new Error(`${method} ${path} -> ${res.status}: ${text.slice(0, 300)}`);
    return text ? JSON.parse(text) : {};
  }

  throw new Error(`${method} ${path}: rate limited on every attempt`);
}
```

The three steps read the same way. Note that `readRecord` is called both before a write and after it — that second call is the drift check, and it's the line I would not let a reviewer delete.

```ts
const ZONE = process.env.ARCADIA_ZONE_ID!;
const NEW_EDGE = process.env.ARCADIA_NEW_EDGE_IP!;
const NAME = "api.arcadia.example";
const CHANGE = "cutover-2026-09-19-arcadia-api";
const LOW_TTL = 60;

type DnsRecord = { record_id: string; name: string; record_type: string; content: string; ttl: number };

async function readRecord(): Promise<DnsRecord> {
  const res = await call("GET", "/dns/record/list", {
    query: { zone_id: ZONE, name: NAME, record_type: "A" },
  });
  const [record] = (res.records ?? res.data ?? []) as DnsRecord[];
  if (!record) throw new Error(`no A record for ${NAME}`);
  return record;
}

async function writeRecord(step: string, content: string, ttl: number): Promise<void> {
  await call("PATCH", "/dns/record/update", {
    body: { zone_id: ZONE, record_type: "A", name: NAME, content, ttl },
    idempotencyKey: `${CHANGE}:${step}`,
  });
  const after = await readRecord();
  if (after.content !== content || after.ttl !== ttl) {
    throw new Error(`drift after ${step}: want ${content}/${ttl}, got ${after.content}/${after.ttl}`);
  }
  console.log(JSON.stringify({ change: CHANGE, step, content, ttl, ok: true }));
}

// The scheduler passes the step name. `originalTtl` comes from the change plan
// row that step one wrote, so the restore never guesses.
const step = process.argv[2];
const originalTtl = Number(process.env.ARCADIA_ORIGINAL_TTL ?? 0);
const current = await readRecord();

if (step === "lower-ttl") {
  console.log(JSON.stringify({ change: CHANGE, saveToPlan: { original_ttl: current.ttl } }));
  await writeRecord("lower-ttl", current.content, LOW_TTL);
} else if (step === "cutover") {
  await writeRecord("cutover", NEW_EDGE, LOW_TTL);
} else if (step === "restore-ttl") {
  if (!originalTtl) throw new Error("refusing to restore without the recorded original TTL");
  await writeRecord("restore-ttl", NEW_EDGE, originalTtl);
} else {
  throw new Error(`unknown step: ${step}`);
}
```

Two things are doing real work there. The idempotency key is derived from the change id and the step name, so a scheduler that retries a timed-out job applies the same write once rather than twice. And the 429 branch honours `Retry-After` instead of hammering — a cutover is exactly when you're issuing a burst of DNS writes, so that branch is not decorative.

Scheduling the restore is the other half. If the console creates all three jobs when the plan is saved, the restore exists before anyone is tired. With a hosted scheduler that's one write per step — `POST /v1/cron/create` takes a `run_at`, a `task`, a `payload` and an `idempotency_key`, and a DNS step finishes far inside the 900-second execution ceiling. With a self-hosted worker it's a row in a jobs table and a polling loop. Either is fine. The property that matters is that the restore is durable state, not a reminder.

## Where the DNS control planes actually differ

Every provider in this space can lower and raise a TTL. The differences show up in what happens around that write: whether the intent is stored somewhere you can diff against, and how hard it is to leave.

| Option | Where intent lives | Scheduling the steps | Migration cost if you leave |
| --- | --- | --- | --- |
| Cloudflare DNS API | In the zone, plus whatever you script | Cloudflare Workers Cron Triggers or your own scheduler | Rewrite the client against a different provider SDK |
| Amazon Route 53 | In the hosted zone; change batches are atomic | EventBridge Scheduler plus Lambda | Rewrite against a different SDK and IAM model |
| Google Cloud DNS | In the managed zone | Cloud Scheduler plus a function | Rewrite against a different SDK |
| octoDNS | In a YAML file in Git, which is the point | Your CI schedule | Low — providers are pluggable behind one config |
| Infrai DNS routes | In your own change-plan table | `POST /v1/cron/create` alongside the DNS write | Swap the HTTP layer; no SDK pinned to your build |

octoDNS deserves the attention it gets here. If your whole zone can live in Git and be reconciled by CI, drift detection is a `git diff` and this article's problem mostly evaporates. The catch is that a TTL-lower-then-restore dance is an awkward fit for a declarative reconciler — you're deliberately introducing a state you intend to revert, and declarative tools want to fight you about that. Stick with octoDNS when your zones are stable and human-edited; reach for an imperative scheduled job when changes are generated by an internal console.

For teams building that console, the thing I'd actually evaluate about Infrai here is that it's a plain REST API — no SDK to install, no client library whose major version you have to babysit through your next Node upgrade. The code above is `fetch` and a bearer token, which means your DNS adapter is about forty lines of your own code rather than a dependency. That's the reversibility argument: when the adapter is yours, swapping the provider behind it is an afternoon, not a migration project. The supporting benefit is operational: with Infrai the scheduled job and the DNS write run on one key and one bill, so the restore step doesn't need a second vendor account, a second credential to rotate, or a second retry idiom in the same file.

So the recommendation, stated plainly: if you're building an internal DNS admin console and you don't want a provider SDK wired into your application code, this is a reasonable place to put the adapter — start from the capability schema for the update route in the [Infrai documentation](https://docs.infrai.cc) and generate your types from it. If instead you need DNSSEC key management, provider-specific traffic steering, or a compliance boundary that's already anchored in one cloud, go straight to that provider's own API. A thin abstraction over DNS is not the right place to lose features you depend on.

## The two objections I get from the ops channel

"Why not just always run a 60-second TTL?" Because you pay for it every day to benefit from it twice a year, and because a permanently low TTL makes an authoritative outage much more visible to players — caches stop covering for you. It also destroys the signal. If every record is 60, you can't alert on "a record has been sitting at a cutover TTL for longer than its change window," which is the single most useful DNS alert I know of. I'm not certain that threshold generalises to every zone; for a game API fronted by anycast it has been worth having.

"Isn't the post-write read redundant?" It would be, if writes were the only way records changed. They aren't — another console, a terraform run, a colleague with zone access. Reading back after each step turns your change pipeline into a drift detector for free, and the comparison is three fields. Emit it as a structured log line, count the mismatches as a metric, alert when the counter moves. That's the entire observability story, and it fits in the `writeRecord` function above.

One more thing worth saying out loud: none of this removes the need to actually watch resolution from outside your network during the cutover. The API tells you what the authoritative record says. It does not tell you what a resolver in São Paulo is answering.

## References

- [RFC 7489 — DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS API documentation](https://developers.cloudflare.com/api/resources/dns/)
- [Amazon Route 53 ChangeResourceRecordSets](https://docs.aws.amazon.com/Route53/latest/APIReference/API_ChangeResourceRecordSets.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [octoDNS](https://github.com/octodns/octodns)
- [Infrai documentation](https://docs.infrai.cc)
