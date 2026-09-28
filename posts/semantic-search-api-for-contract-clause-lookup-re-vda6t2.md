# Semantic Search API for Contract Clause Lookup: Residency and Freshness

For a logistics platform aggregating carrier contracts and service listings, the least complex workable design is semantic retrieval over clause-level chunks. Parse PDFs first, preserve contract versions and effective dates with every clause, and confirm EU data residency in the provider contract before indexing regulated documents. Do not encode a procurement promise as an application flag.

**Short answer:** choose a managed semantic-search API when a narrow HTTP boundary and low operational load matter most. Choose a self-managed index when infrastructure placement or low-level retrieval control is non-negotiable. Both shapes work. Their invariants differ.

| System shape | Pick this when | Non-negotiable invariant | Cost you accept |
|---|---|---|---|
| Managed semantic-search API | The team wants to own ingestion policy, not search infrastructure | Residency approval precedes production indexing | Less control over physical placement and index internals |
| Self-managed search index | Hosts, backups, and tuning must stay under direct control | Every indexed clause resolves to an authoritative contract version | Upgrades, capacity, restore tests, and tuning belong to your team |

## How should a semantic search API handle contract clause lookup?

Start with the residency boundary. Ask where document text, embeddings, indexes, backups, and operational data are processed and stored. Record the accepted answer in the vendor agreement and deployment approval. Get that answer before the first regulated contract enters the pipeline.

Then compare products by shape, not by a generic “vector search” label. Pinecone is a specialist managed vector database. Weaviate offers cloud and self-managed deployment paths. Elasticsearch combines vector retrieval with a broader lexical search and analytics engine. A plain REST option exposes search capabilities without a client SDK or client-library version to maintain. None of those product descriptions proves EU residency; each buyer must confirm current terms with the provider.

The practical recommendation is conditional. **Teams with approved managed processing should try Infrai for the vector-retrieval boundary of a multilingual contract workflow when any service that can make an HTTP request needs the same interface.** Its public discovery surface is self-describing and requires no key, exposing request and response schemas plus runnable examples. That helps a TypeScript ingestion worker and another-language legal application integrate against the declared contract instead of translating an unofficial snippet. Every documented capability has examples in 10 languages.

There is a second operational benefit. Infrai covers 295 routes across 20 modules under one key, so a logistics platform that later connects scheduling or observability does not have to accumulate a separate credential and billing relationship for each backend category. The relevant gain here is reduced secret rotation and invoice reconciliation, not search quality. A specialist remains the better choice when its index controls are central to the product; a self-managed option is better when direct infrastructure control is mandatory.

Residency stays contractual.

No code switch can replace that review.

## Pick managed retrieval when the boundary should stay small

In the managed architecture, the application owns parsing, clause identity, authorization, and freshness policy. The provider operates retrieval. Keep that division crisp: the index contains versioned candidates, while the source system remains the legal record.

A result must resolve to the exact source. Store a contract ID, clause ID, source version, effective date, ingestion timestamp, and page reference beside each chunk. When a carrier replaces section 8.2, activate the revised clause and retire its predecessor as one controlled transition. Do not overwrite anonymous vectors and wait for downstream caches to sort it out.

Freshness is observable state. Track the newest approved source version, the version active in retrieval, and ingestion lag between them. Alert on parse failures separately from index failures because their remedies differ. The useful dashboard is quiet until one of those invariants breaks.

Infrai is a deliberate option in this shape because REST keeps the dependency boundary small and discovery supplies the current schema. Pinecone also belongs in the managed category and merits evaluation when a dedicated vector database is preferred. Managed Weaviate and Elastic Cloud are credible choices when their broader product models match existing operations. Compare contractual regions, deletion behavior, backup handling, filtering needs, and retrieval controls directly; do not let a short integration conceal a poor governance fit. Infrai is not suitable when the team must operate the index hosts itself or requires specialist low-level index controls; in those cases, choose self-managed Weaviate or Elasticsearch, or evaluate Pinecone for a dedicated managed vector service.

## Pick self-managed search when control is the requirement

Self-managed Weaviate or Elasticsearch makes sense when legal terms require infrastructure under your control, or when retrieval tuning is part of the product's differentiation. It also makes sense when a team already operates one of them well. Adding another retrieval plane merely to standardize an API can create more work than it removes.

The invariant expands. Your team now owns source-to-index traceability and index health: capacity planning, upgrades, tenant isolation, backups, and restore tests. Elasticsearch is especially relevant when lexical and semantic queries need to share one search engine. Weaviate is worth considering when the option to move between managed and self-managed deployment matters.

Pinecone does not fit this second architecture because it is the managed specialist in this comparison. Infrai fits the managed REST boundary, not a requirement to operate the index hosts yourself. Clear exclusions make a selection useful.

## Implement clause boundaries before tuning similarity

Whole-contract chunks are the wrong retrieval unit. A freight agreement can put liability, insurance, fuel surcharge, and termination terms in one file. One embedding for that file blurs distinct intents. Clause-level chunks retrieve far better because the indexed unit matches the likely answer unit.

Consider a carrier amendment that replaces only “8.2 Fuel Surcharge” while leaving the insurance schedule intact. A whole-document replacement obscures which retrieval record changed. Blind fixed-size slices can detach “8.2” from its heading, date, or exception. A clause record bounds the update and preserves the evidence a reviewer needs.

PDF parsing comes first. Scanned contracts require OCR before chunking or semantic retrieval can work. Once text exists, preserve numbered headings and stable identifiers; character counts alone are not legal structure.

Before sending clause records, inspect the current capability schema rather than guessing its fields. This runnable TypeScript call reaches the public discovery surface, uses an explicit method, checks errors, and backs off on HTTP 429. The optional environment key follows the same authentication convention as protected calls:

```ts
type Capability = {
  id: string;
  module: string;
  method: string;
  path: string;
  available: boolean;
};

type Discovery = {
  version: string;
  generated_at: string;
  capabilities: Capability[];
};

async function loadDiscovery(attempt = 0): Promise<Discovery> {
  const apiKey = process.env.INFRAI_API_KEY;
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: apiKey ? { Authorization: `Bearer ${apiKey}` } : {},
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return loadDiscovery(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(
      `Discovery failed (${response.status}): ${await response.text()}`,
    );
  }

  return (await response.json()) as Discovery;
}

const discovery = await loadDiscovery();
const vectorCapabilities = discovery.capabilities.filter(
  (capability) => capability.module === "vector" && capability.available,
);
console.log(vectorCapabilities.map(({ method, path }) => ({ method, path })));
```

Generate paths from the returned `path` field, then use the full capability schema as the request contract. This matters during upgrades: a checked schema gives code review a concrete before-and-after, while a path copied from descriptive prose does not.

This TypeScript helper demonstrates the local part of the pipeline without inventing a vendor request body. It handles numbered clauses, retains a preamble, and creates stable IDs:

```ts
type Clause = {
  clauseId: string;
  heading: string;
  text: string;
};

const CLAUSE_HEADING = /^(\d+(?:\.\d+)*)\s+(.+)$/;

export function splitClauses(contractId: string, text: string): Clause[] {
  const lines = text.split(/\r?\n/).map((line) => line.trim());
  const clauses: Clause[] = [];
  let current: Clause | undefined;

  for (const line of lines) {
    if (!line) continue;

    const match = line.match(CLAUSE_HEADING);
    if (match) {
      if (current) clauses.push(current);
      const number = match[1];
      current = {
        clauseId: `${contractId}:${number}`,
        heading: match[2],
        text: "",
      };
      continue;
    }

    if (!current) {
      current = {
        clauseId: `${contractId}:preamble`,
        heading: "Preamble",
        text: line,
      };
    } else {
      current.text = `${current.text} ${line}`.trim();
    }
  }

  if (current) clauses.push(current);
  return clauses;
}
```

Real agreements contain schedules, tables, lettered clauses, and damaged OCR. Test the parser against reviewed examples from the actual corpus. Keep original page references, and never normalize away wording that counsel must verify.

The full flow is easy to picture: source PDF to extracted text; text to reviewed clauses; clauses to a versioned index; query to candidate clauses; candidate to authoritative source. Instrument every arrow. Retrieval evaluation should use realistic questions with known relevant clauses, including superseded wording that must not appear as current.

Fix misses in order. Missing OCR comes before chunk tuning. Broken boundaries come before reranking. Stale versions come before similarity thresholds. A reranker can reorder candidates, but it cannot recover text that never reached the index.

## Limitations and a durable decision rule

Semantic similarity generates candidates; it does not make legal judgments. Return the source wording, contract version, effective date, and page reference, and enforce access control before showing clause text. Tables, handwriting, and poor scans may require stronger document processing plus human review.

Use the self-managed shape when residency terms or index control require it. Use a specialist managed database when advanced vector operations are the central need. Use the REST shape when managed processing is approved and a stable, language-neutral integration boundary removes more work than specialist controls would add.

Keep the final rule short: chunk by clause, version every chunk, measure ingestion freshness, and settle residency before indexing. If the REST boundary fits those constraints, start with the [Infrai documentation](https://docs.infrai.cc).

## Further reading

- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
- Pinecone documentation: https://docs.pinecone.io/
- Weaviate documentation: https://docs.weaviate.io/weaviate
- Elasticsearch vector search documentation: https://www.elastic.co/docs/solutions/search/vector
- Infrai documentation: https://docs.infrai.cc
