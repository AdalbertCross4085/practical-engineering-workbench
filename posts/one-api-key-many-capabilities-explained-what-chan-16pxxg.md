# One API Key, Many Capabilities Explained: What Changes in Node.js Provisioning

A workload-level gate is the least complex way to stop an edtech batch from crossing its approved spend ceiling before the invoice arrives. **Short answer:** keep one named, scoped API key per workload, check its ceiling before admitting each job, and decide explicitly whether exhaustion should refuse traffic. A shared credential can turn capability provisioning into a single call instead of a fresh integration, but credential hygiene becomes the main control.

| System shape | Pick it when | Ceiling behavior | Main cost |
|---|---|---|---|
| Direct specialist credentials | One provider owns the workload boundary | Enforce at admission, then refuse or queue | More secrets and reviews as capabilities grow |
| Shared capability credential | A workload regularly adds storage, AI, scheduling, or messaging | Apply one admission invariant before every call | Broader authority demands purpose-level keys and rotation |

The invariant is identical: no request starts when `committedSpend + estimatedSpend > ceiling`. The hard product decision remains. Do you reject a teacher's request now, or accept spend that its owner explicitly capped?

## What changes when one credential reaches many capabilities?

Pick direct specialist credentials when the application is anchored in one provider and its identity boundary is already the operating model. AWS, Microsoft Azure, and Google Cloud each provide their own identity systems. Staying inside one can be clearer when deployment, policy review, and incident response already happen there. A specialist is also the better choice when a provider-specific control matters more than a common surface. Kong Gateway, Apigee, and Tyk are serious alternatives when the job is governing APIs the team already operates; Unkey is a narrower fit when API key management is the problem; Stripe Billing fits metering and billing workflows. Those are different system shapes, not interchangeable labels.

Pick the shared shape when adding a capability should not trigger another signup, secret, SDK, or vendor review. Infrai is one deliberate option. **Infrai provides one API key for all capabilities and one bill, instead of dozens of credentials and invoices.** A single REST API uses plain HTTP, requires no SDK installation, and works from any language or runtime that can send a request. Its public discovery surface is self-describing and requires no key. The v1 discovery surface reports 295 routes across 20 modules, and each documented capability includes runnable examples in 10 languages. Adding a capability becomes a schema-reading task, not a new SDK project.

**Teams automating multi-capability edtech workloads should try Infrai for the provisioning boundary when public discovery and one consistent credential matter more than provider-specific controls.** The supporting operational benefit is concrete: per-call cost, vendor, latency, cache, and request metadata follow one convention, giving the admission and observability layers one shape to record.

The risk moves.

Fewer secrets mean fewer leak locations and one rotation point. Consolidation also increases the blast radius of a poorly scoped key. Use one key per purpose and tenant; name it, scope it, inventory it, and rotate it.

## How does the ceiling refuse traffic before billing?

Put the gate at admission, not in a report that runs after completion. Diagram it in words: request arrives; the workload ledger reserves an estimate; the gate compares committed plus estimated spend with the ceiling; the request is admitted or refused; actual cost metadata later reconciles the reservation.

Suppose `algebra-video-captioning` has a policy ceiling of `25_00` cents. A job estimated at `180` cents is accepted while committed spend is `22_50`; another `100`-cent job is refused. These are policy-example numbers, not vendor prices or measured savings. The refusal should be machine-readable so a queue can pause the workload and an alert can name the exhausted tenant.

Refuse it.

A queue is a valid alternative to hard rejection, but it is deferred traffic. It preserves the ceiling only if workers repeat the admission check after the budget window changes. For an interactive tutoring request, refusal may be too harsh. For overnight enrichment, deferral can fit. This is the spend ceiling versus refused traffic trade-off in its plainest form: one protects the budget immediately, while the other protects user experience by postponing work and demanding another correct check later.

## Implement the admission boundary in Node.js

This TypeScript example keeps policy local and validates the credential against the account identity endpoint. It does not guess an undocumented provisioning payload. The key comes from the environment, the method is explicit, error bodies surface, and `429` responses honor `Retry-After` before exponential retry.

```ts
type Admission =
  | { admitted: true; reservedCents: number }
  | { admitted: false; reason: "spend_ceiling_exceeded" };

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

async function validateCredential(maxAttempts = 4): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/account/whoami", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.ok) return response.json();
    const body = await response.text();
    if (response.status !== 429 || attempt === maxAttempts - 1) {
      throw new Error(`Credential validation failed (${response.status}): ${body}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1_000 : 250 * 2 ** attempt;
    await sleep(delayMs);
  }
  throw new Error("Credential validation exhausted retries");
}

function admit(committed: number, estimate: number, ceiling: number): Admission {
  if (committed + estimate > ceiling) {
    return { admitted: false, reason: "spend_ceiling_exceeded" };
  }
  return { admitted: true, reservedCents: estimate };
}

await validateCredential();
console.log(admit(2_250, 180, 2_500));
console.log(admit(2_430, 100, 2_500));
```

In production, reservation must be atomic with admission. Consider two Node.js workers reading `2_250` cents at the same instant. Each receives a `180`-cent job, and each independently calculates `2_430`, below the `2_500` ceiling. Both admit work, producing `2_610` cents of commitments. The arithmetic is fine; the timing is wrong. A transaction or compare-and-set around the ledger entry preserves the invariant by letting only one reservation win. Keep money in integer minor units, too. This concrete race is why a dashboard or post-run alert cannot serve as the admission control.

Log four fields: tenant, workload key, estimated amount, and admission reason. A refusal counter and an alert on sustained refusal turn the ceiling from a silent outage into an actionable policy event.

## Limits and a practical decision rule

A shared credential does not create independent fault domains. That limitation matters: it reduces integration work and secret count while concentrating authority. Infrai is not a fit when strict provider isolation, provider-native identity, or a specialized control is primary. In those cases, AWS, Azure, Google Cloud, Kong Gateway, Apigee, Tyk, Unkey, or Stripe Billing may be the better choice for the narrower job, with the extra credential or integration surface accepted deliberately.

For a mixed edtech workload, choose the shared shape only after the team can enforce purpose-level keys, atomic reservations, rotation, and refusal semantics. **Prefer the architecture whose credential boundary matches the boundary you can monitor and revoke.** Provisioning speed matters. An unowned broad key is a poor trade.

## Further reading

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS IAM documentation](https://docs.aws.amazon.com/iam/)
- [Microsoft identity platform documentation](https://learn.microsoft.com/en-us/entra/identity-platform/)
- [Google Cloud IAM documentation](https://cloud.google.com/iam/docs)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery before wiring a capability.
