# Marketplace Help Center Grounding with Managed Vector Search (Small SaaS Constraints)

**TL;DR:** Keep product content in the database your small team already operates when it can meet the measured retrieval target. Add vector retrieval there, but make evidence IDs, revisions, and citations part of the response contract from day one. Choose a hosted vector API when an instrumented replay shows that the current store misses the target or creates unacceptable operational work. The deciding artifact is not a feature matrix or an advertised unit price. It is a trace showing which product-content revision produced each answer.

| Pick | Pick this when | Grounding check | Operational cost to count |
| --- | --- | --- | --- |
| Vector search beside relational product data | Product text, permissions, and revisions already live together; measured load fits the system | Join every hit back to the current product-content revision before generation | Index maintenance, database headroom, backups, and query tuning |
| Hosted vector API | An isolated service boundary is acceptable and measured retrieval needs justify it | Store stable evidence IDs and verify returned IDs against the source of truth | Data synchronization, retries, monitoring, access review, and incident ownership |
| Lexical-first or hybrid retrieval | Exact model numbers, SKUs, and policy phrases carry meaning that embeddings may blur | Preserve the matched passage and retrieval method in the trace | Two retrieval paths, fusion logic, and a larger evaluation matrix |

This is the practical answer to "pgvector or a hosted API?" for a small marketplace help center. Start with the least complex option that can return attributable evidence. Make it earn promotion through tests. A cheap request that cannot explain its source is expensive the first time a seller disputes an answer.

## Should a small SaaS help center use managed vector search?

You are choosing a failure boundary.

Trace first.

The database-adjacent option keeps the product record, help article, visibility rule, revision, and vector close enough to validate in one controlled read path. That reduces synchronization machinery. It does not guarantee relevant retrieval, and it does not remove the need to observe indexing lag, query latency, or stale evidence.

A hosted API moves vector indexing and search behind another boundary. That can be a reasonable trade when the measured workload demands it. The boundary also creates a second copy of identifiers and searchable content, so the application must define what happens when indexing succeeds late, a delete arrives out of order, or a result points at a revision that is no longer publishable. Those are application questions, independent of vendor.

That is the trade-off.

RAG combines retrieval with generation so generated output can draw on external knowledge rather than relying only on model parameters. That mechanism makes retrieval quality central to the answer, but retrieval alone does not create a citation. The application must carry the selected evidence through generation and render references that a reader can inspect.

For marketplace content, use a stable content identity and a changing revision identity. A product can keep `productId` while every published help-center revision gets a distinct `revisionId`. Index chunks with both. At query time, reject a hit unless its revision is still current and visible to the requesting audience. This one rule prevents a persuasive answer from quietly citing an expired return policy.

## Pick each option for a reason

Pick database-adjacent vector search when the team already trusts its relational database as the product-content source of truth and the measured corpus and query load fit within its operating envelope. The attractive property is locality: authorization and revision checks can happen where their authoritative records live. Keep enough headroom for ordinary marketplace traffic. Retrieval is a secondary workload, not permission to starve checkout or catalog writes.

Pick a hosted vector API after a replayable evaluation demonstrates a need that the current path does not meet. Define that need as a service-level objective, such as a latency percentile under a named concurrency and corpus snapshot, or a relevance threshold on a reviewed query set. Do not substitute "managed" for an ownership model. Someone still owns ingestion lag, credentials, deletion propagation, and degraded behavior.

Pick lexical-first or hybrid retrieval when exact strings decide correctness. Marketplace questions often contain SKU-like tokens, compatibility codes, quoted policy language, or model names. Semantic similarity may find related prose while missing the exact constraint. A hybrid path has more moving parts, yet the trade is visible: extra evaluation and fusion work in exchange for a chance to preserve both conceptual and literal matches.

Do not select from a price page alone. Different systems meter different resources, and the application workload includes embedding, ingestion, storage, queries, database capacity, network transfer, observability, and engineering time. Use the same replay corpus and traffic envelope for every candidate. Then record the full boundary.

Neither option is a fit everywhere. Database-adjacent search is not a fit when its measured workload consumes the headroom reserved for the marketplace's primary transactions, while an external API is not a fit when policy forbids another content copy or the team cannot own synchronization failures. Hybrid retrieval has its own limitation: the team must evaluate and operate two retrieval paths plus their fusion rule. These boundaries are more useful than a universal winner.

## Instrument the evidence path

The deepest implementation work belongs at the application boundary, because that contract survives a storage change. Give every retrieval attempt a trace ID. Record the query fingerprint, corpus snapshot, retrieval mode, candidate evidence IDs, accepted revisions, scores as opaque diagnostics, and the citations actually sent to generation. Avoid logging raw customer queries by default; help-center questions can still contain order details or personal data.

Here is a compact TypeScript boundary. The adapter may call a database index or an external service. The policy stays in application code.

```ts
type Candidate = {
  evidenceId: string;
  productId: string;
  revisionId: string;
  text: string;
  score: number;
};

type CurrentRevision = {
  productId: string;
  revisionId: string;
  public: boolean;
};

interface Retriever {
  search(query: string, limit: number): Promise<Candidate[]>;
}

interface Catalog {
  currentRevisions(productIds: string[]): Promise<CurrentRevision[]>;
}

type RetrievalTrace = {
  traceId: string;
  corpusRevision: string;
  candidateEvidenceIds: string[];
  acceptedEvidenceIds: string[];
  rejected: Record<string, "stale" | "not_public" | "missing">;
};

async function retrieveGroundedContext(
  query: string,
  traceId: string,
  corpusRevision: string,
  retriever: Retriever,
  catalog: Catalog,
): Promise<{ passages: Candidate[]; trace: RetrievalTrace }> {
  const candidates = await retriever.search(query, 12);
  const current = await catalog.currentRevisions(
    [...new Set(candidates.map((candidate) => candidate.productId))],
  );
  const byProduct = new Map(current.map((item) => [item.productId, item]));
  const rejected: RetrievalTrace["rejected"] = {};

  const passages = candidates.filter((candidate) => {
    const record = byProduct.get(candidate.productId);
    if (!record) {
      rejected[candidate.evidenceId] = "missing";
      return false;
    }
    if (!record.public) {
      rejected[candidate.evidenceId] = "not_public";
      return false;
    }
    if (record.revisionId !== candidate.revisionId) {
      rejected[candidate.evidenceId] = "stale";
      return false;
    }
    return true;
  });

  return {
    passages,
    trace: {
      traceId,
      corpusRevision,
      candidateEvidenceIds: candidates.map((item) => item.evidenceId),
      acceptedEvidenceIds: passages.map((item) => item.evidenceId),
      rejected,
    },
  };
}
```

The number `12` is an explicit candidate budget, not a universal optimum. Tune it against the evaluation set. The important behavior is structural: retrieval produces candidates; the catalog validates them; generation receives only accepted passages. If no passage survives, return a no-evidence state instead of asking the model to improvise.

No evidence, no answer.

A diagram in words: query enters, retrieval fans out, current-revision validation narrows the set, generation consumes accepted passages, and the response links citations back to evidence. One trace ID connects every step. Crisp.

Emit counters for retrieval attempts, empty accepted sets, and rejection reasons. Measure latency for retrieval and validation separately. Alert on sustained changes in empty-result rate, stale-revision rejection, indexing lag, and end-to-end latency; thresholds must come from the service objective and normal traffic, not invented universal values. Logs explain individual traces. Metrics show population shifts. A sampled trace joins them when an answer is challenged.

## Test grounding before comparing infrastructure

Build a versioned evaluation set from real information needs, but remove personal data. Each case needs a query, the corpus revision, acceptable evidence IDs, forbidden or stale evidence IDs, and an expected answerability state. Include typo-heavy product names, exact identifiers, policy questions, ambiguous category terms, and questions for which the corpus contains no answer. The last group tests restraint.

Replay the same set against each candidate architecture. Track retrieval recall over the labeled evidence, citation validity after revision checks, empty-set behavior, and latency distributions. Generation quality can be reviewed separately because a fluent model response can conceal a retrieval miss. Keeping the layers distinct tells the team what to fix.

Deployment needs the same discipline. Shadow a new index against recorded queries, compare evidence IDs without exposing its answers, and canary the read path before switching all traffic. During reindexing, attach a corpus revision to both the trace and the index alias or equivalent application pointer. A rollback then restores a known evidence snapshot rather than a vague previous state.

Error handling should be boring. Time out retrieval within the response budget, retry only operations that are safe to repeat, and cap retries so a dependency slowdown does not multiply load. If evidence validation is unavailable, do not present generated product guidance as grounded. A plain search result or a temporary no-answer response preserves the contract better.

The before/after is easy to review. Before, the team debates database extensions and APIs from feature pages. After, every candidate runs the same corpus snapshot, evidence labels, traffic replay, and failure drills. The selection becomes an engineering record: observed quality, observed latency, operational burden, and known limits.

## Limits to keep visible

This design does not prove that a cited passage is true. It proves which approved revision supported the answer. Editorial review and catalog governance remain upstream responsibilities.

It also does not produce one permanent infrastructure choice. Corpus size, traffic shape, team capacity, and relevance needs change. Keep the retriever interface narrow, preserve stable evidence identities, and rerun the evaluation when those inputs materially change. The best small-team choice is the one that meets the grounding objective with the fewest unowned failure modes.

## Further reading

- https://arxiv.org/abs/2005.11401
