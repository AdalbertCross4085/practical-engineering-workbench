# Async Batch LLM Jobs: Compare 4 Cost Signals Against Realtime APIs

TL;DR: Async batch LLM jobs can be cheaper than realtime API calls for bulk support-ticket work, but only a measured comparison can establish that result. Route work with an interaction deadline through realtime inference. Put summarization, tagging, and extraction for backlog enrichment into batch, then judge the split with four signals: queue age, end-to-end latency, schema-valid yield, and reviewer corrections. For edtech support triage, quality wins when a wrong route delays a student; latency wins when an agent is waiting.

| Pick | Use it when | Primary signal | Failure to expose |
|---|---|---|---|
| Realtime | An agent or automated reply is blocked | End-to-end latency | Fast but low-quality classification |
| Batch | Tickets can wait until a declared deadline | Queue age | Processing that finishes too late |
| Hybrid | Urgency needs a quick pass, while enrichment can wait | Reviewer corrections by path | Two paths that silently disagree |
| Rules first | A deterministic condition can decide | Rule coverage and fallback rate | Brittle rules swallowing ambiguous cases |

This is a deadline decision before it is a purchasing decision. A nightly queue of old tickets and a new account-lockout ticket do not have the same value of time. Treating them alike hides the operational question behind one blended average.

## Can async batch LLM jobs be cheaper than realtime API calls?

Pick realtime for work that sits directly in a support interaction: detecting an urgent ticket, extracting the course or account identifier needed for the next screen, or suggesting a queue while an agent waits. Measure from ticket acceptance to a usable triage record. Provider response time alone omits queueing, validation, and retry time.

Pick batch for deferrable work such as re-tagging a closed-ticket archive, producing summaries for tomorrow's handoff, or extracting fields for a later quality review. A batch interface is evidence that asynchronous processing is an available architectural option, not proof that every workload will be cheaper or good enough. The AWS Bedrock page, for example, lists batch inference among its inference options [1]. This article does not infer a universal discount from that fact.

Waiting can help.

Pick a hybrid when urgency and enrichment have different clocks. A small synchronous decision can assign an initial priority; a deferred job can later produce a fuller summary, taxonomy tags, and structured extraction. The two outputs need explicit version fields and reconciliation rules. Otherwise, an agent sees one route while reporting sees another.

Rules first are useful for exact conditions already present in trusted data. If an authenticated account state gives the answer, do not ask a model to rediscover it. Send ambiguous cases to inference and track how often that fallback occurs.

Measure that fallback.

The diagram in words is short: ticket intake goes to a policy gate; the gate sends deadline-bound work to the realtime worker and deferrable work to a durable queue; both paths feed the same validator, review sampler, and metrics store; only validated records update the support system. One contract. Two clocks.

## Four signals make the trade-off visible

**Queue age** answers whether deferred work still meets its promise. Record `acceptedAt`, `eligibleAt`, `startedAt`, and `completedAt`. Report percentiles by job class, not just a global average. A calm archive-enrichment queue can otherwise conceal urgent work that was accidentally admitted. Alert against the declared deadline, with enough margin for recovery.

**End-to-end latency** belongs to the realtime path. Start the clock at ticket intake and stop it only when a validated result is available to the consumer. Split the trace into queue, inference, validation, and persistence spans. That crisp before-and-after view tells an engineer whether changing execution mode would address the delay at all.

**Schema-valid yield** is the fraction of accepted inputs that produce records passing structural and business validation. Count terminal failures in the denominator. A transport success is not a triage success if `priority` is missing or the destination queue is unknown. Track yield by prompt version, model configuration, language, and execution path so a blended rate cannot bury a regression.

**Reviewer corrections** are the quality counterweight. Sample both paths under the same rubric and log corrected priority, destination, and extracted fields. Raw agreement is useful, but the operational question is sharper: would the correction change who handles the ticket or how quickly it is handled? Weight that consequence in the release decision.

Do not force these into one magic score. A dashboard should put deadline attainment beside schema-valid yield and correction rate. The owner can then state a policy: move a job class to batch only when its completion deadline is met and its reviewed quality remains within the team's predeclared tolerance. The tolerance is a local product decision, not a number borrowed from a provider page.

## Implement one contract across both paths

The focused Node.js implementation below keeps provider details behind a TypeScript interface. It validates every result in one place and emits dimensions that can be compared without mixing unlike work. The same contract covers bulk summarization, tagging, and structured extraction. It intentionally omits a vendor endpoint, model identifier, and assumed price. Those values belong in an adapter and deployment configuration.

```ts
type Priority = "urgent" | "normal" | "low";
type Mode = "realtime" | "batch";

type Ticket = {
  id: string;
  text: string;
  acceptedAt: string;
  agentWaiting: boolean;
};

type Triage = {
  ticketId: string;
  priority: Priority;
  destination: string;
  summary: string;
  promptVersion: string;
};

type Observation = {
  mode: Mode;
  ticketId: string;
  promptVersion: string;
  acceptedAt: string;
  completedAt: string;
  schemaValid: boolean;
};

interface InferencePort {
  realtime(ticket: Ticket): Promise<unknown>;
  enqueue(ticket: Ticket): Promise<{ jobId: string }>;
}

interface DeferredQueue {
  put(message: { jobId: string; ticket: Ticket }): Promise<void>;
}

interface MetricsSink {
  record(observation: Observation): void;
}

const priorities = new Set<Priority>(["urgent", "normal", "low"]);

function validate(value: unknown, ticketId: string): Triage {
  if (typeof value !== "object" || value === null) throw new Error("invalid triage object");
  const row = value as Record<string, unknown>;
  if (!priorities.has(row.priority as Priority)) throw new Error("invalid priority");
  if (typeof row.destination !== "string" || row.destination.length === 0) {
    throw new Error("invalid destination");
  }
  if (typeof row.summary !== "string" || typeof row.promptVersion !== "string") {
    throw new Error("invalid summary metadata");
  }
  return {
    ticketId,
    priority: row.priority as Priority,
    destination: row.destination,
    summary: row.summary,
    promptVersion: row.promptVersion
  };
}

async function triageTicket(
  ticket: Ticket,
  port: InferencePort,
  queue: DeferredQueue,
  metrics: MetricsSink
): Promise<Triage | { queued: true; jobId: string }> {
  if (!ticket.agentWaiting) {
    const { jobId } = await port.enqueue(ticket);
    await queue.put({ jobId, ticket });
    return { queued: true, jobId };
  }

  let parsed: Triage;
  try {
    parsed = validate(await port.realtime(ticket), ticket.id);
  } catch (error) {
    metrics.record({
      mode: "realtime",
      ticketId: ticket.id,
      promptVersion: "unknown",
      acceptedAt: ticket.acceptedAt,
      completedAt: new Date().toISOString(),
      schemaValid: false
    });
    throw error;
  }

  metrics.record({
    mode: "realtime",
    ticketId: ticket.id,
    promptVersion: parsed.promptVersion,
    acceptedAt: ticket.acceptedAt,
    completedAt: new Date().toISOString(),
    schemaValid: true
  });
  return parsed;
}
```

The batch consumer should call the same `validate` function and write the same observation shape with `mode: "batch"`. Add queue timestamps there because the consumer, not the request handler, knows when work starts and finishes. Keep ticket text out of metric labels; identifiers with high cardinality and customer content do not belong in ordinary aggregate dimensions.

There is one deliberate sharp edge in the snippet: enqueueing at the inference adapter and then writing another queue can create a split state if either operation fails. In a production design, choose one durable system of record, use an idempotency key, and make redelivery harmless. Do not hide this with an automatic loop. Count attempts and surface terminal failures.

Retries are work.

## Compare cost without inventing savings

Start with a unit that survives architecture changes: one accepted ticket that produces one validated triage record. Then account for provider processing, input and output volume, queue operations, object storage, validation, retries, review labor, and records that miss their deadline. Use actual invoices and telemetry for the candidate configuration. Published prices can seed an estimate, but they cannot reveal the retry and correction behavior of this workload. Run the comparison on a fixed, versioned ticket set. Preserve the same requested fields, quality rubric, and acceptance rule across modes. Randomly assign eligible tickets when the operating policy permits it, or replay a frozen set when it does not. Report distributions and counts: admitted tickets, completed records, invalid records, reviewed records, corrections, attempts, and deadline misses. A useful break-even expression is `(batch total cost + batch rework cost) / accepted batch records` versus the equivalent realtime value. Keep missed-deadline consequences beside that arithmetic rather than pretending they have a universal dollar value. If batch processes fewer valid records, dividing only by successful provider responses flatters it. This is also why a provider's API discount cannot stand in for an end-to-end cost claim: the numerator expands outside inference, while the denominator shrinks whenever validation or deadline checks reject an output.

Privacy controls also affect the decision. Support text may contain sensitive information. Minimize submitted fields, restrict access, define retention, and audit both paths. If an organization is a covered entity or business associate, 45 CFR Part 164 is the authoritative place to evaluate applicable privacy and security obligations [2]. Applicability is a legal and organizational determination; an industry label alone does not establish it.

## Limits and the release decision

This method does not supply a universal savings percentage, latency target, or quality threshold. Provider terms change, workloads differ, and the cited public source establishes an available batch mode rather than a result for this ticket corpus. A defensible number requires a measured run against the organization's own data and invoice model.

No shortcut fixes that.

Ship the routing policy only after both paths share validation, review sampling, and comparable telemetry. Keep realtime for deadlines that matter. Admit the rest to batch when queue age stays inside the declared window and reviewer corrections remain acceptable. That is the useful decision: observable, reversible, and tied to student-support outcomes rather than a headline discount.

## Sources

1. AWS, "Amazon Bedrock," https://aws.amazon.com/bedrock/
2. Electronic Code of Federal Regulations, "45 CFR Part 164," https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164
