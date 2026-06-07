# Chapter 11 — Vector Memory, Knowledge Graphs, and Hybrid Retrieval

> **Part 3 — Chapter 2 of 5**
> *"The question is never whether to store. The question is always whether you can get it back when it matters." — Practitioner maxim, agent memory engineering*

Chapter 10 established the memory taxonomy, the subsystem interface, and the four-stage lifecycle. This chapter implements the external memory tier — the concrete storage and retrieval machinery for episodic and semantic memory at production scale.

Three technologies form the implementation foundation:
- **Vector databases** — semantic, similarity-based retrieval for unstructured memory records.
- **Knowledge graphs** — relational, structured querying for entity-level semantic memory.
- **Hybrid retrieval pipelines** — combining both with keyword search to achieve the recall and precision that neither method achieves alone.

Topics are ordered as an engineer must understand them: embeddings first, similarity metrics second, vector database architecture third, knowledge graphs fourth, hybrid retrieval pipelines fifth, retrieval quality measurement sixth, production database selection seventh.

> **Scope boundary:** This chapter covers storage and retrieval mechanics only. Consolidation, importance scoring, pruning, and tiered persistence belong to Chapter 12. Statefulness and cross-session continuity belong to Chapter 13. Advanced retrieval dynamics and security belong to Chapter 14.

---

## 11.1 — Embeddings: Turning Meaning into Numbers

### Context

An LLM processes tokens. A vector database stores floating-point numbers. The gap between a human sentence and a queryable numeric representation is bridged by an embedding model — a neural network that maps text to a fixed-length numeric vector in a high-dimensional space. The key insight: **semantic similarity maps to geometric proximity**. Two sentences expressing the same idea produce vectors that are close together in that space, regardless of shared keywords.

This is the fundamental advantage of vector search over keyword search. An agent can ask *"What does this user dislike about the interface?"* and surface a stored record *"user found the dashboard confusing and slow"* — despite no shared keywords between query and record.

### What — Embedding Model Selection (2026)

Selecting an embedding model is a **consequential, once-made decision**. The model's output dimensions, benchmark performance, normalisation behaviour, and cost profile shape the entire retrieval system. Switch models after data has been loaded and you must re-embed everything.

| Model | Dims | MTEB Score | Best for |
|---|---|---|---|
| `text-embedding-3-large` (OpenAI) | 3,072 | 64.6 | High accuracy, general purpose, normalised |
| `text-embedding-3-small` (OpenAI) | 1,536 | — | Speed and cost optimised |
| `embed-v4` (Cohere) | 1,024 | 65.2 | Highest benchmark score, multilingual |
| `Voyage3 Lite` (Voyage AI) | 512 | — | Compact, low-cost deployments |
| `BAAI/bge-large-en-v1.5` | 1,024 | 63.0 | Open-source, self-hosted, data-residency |

**Selection criteria (in priority order):**
1. **Domain validation first.** Evaluate 2–3 candidate models on a golden retrieval set drawn from your actual memory corpus. General MTEB scores are less predictive than domain-specific retrieval tests. The model with the best domain score wins regardless of MTEB ranking.
2. **Normalisation behaviour.** OpenAI and Cohere `embed-v4` produce unit-normalised vectors, allowing dot product to substitute for cosine at lower compute cost. Verify your model's normalisation behaviour before configuring the vector store.
3. **Multilingual requirement.** If the agent operates across languages, Cohere `embed-v4`'s multilingual capability is a significant differentiator.
4. **Self-hosting constraint.** If data residency or compliance requirements prohibit sending text to external APIs, `BAAI/bge-large-en-v1.5` is the strongest open-source option.

### What — Chunking Strategy

Embedding models have a token limit (typically 512–8,192 tokens). Content longer than this limit must be split before embedding. Chunking strategy directly controls retrieval quality and is the decision most frequently underdesigned in production memory systems.

| Strategy | Use when | Primary risk |
|---|---|---|
| **Fixed-size with overlap** | Simple pipeline, homogeneous content, baseline implementation | Splits semantic units at arbitrary token boundaries |
| **Sentence-boundary** | Prose content, conversational memory, narrative episodes | Very short sentences lose surrounding context |
| **Semantic chunking** | Heterogeneous content, mixed-topic documents, high precision | Requires secondary model pass; higher ingestion latency |
| **Natural record boundary** | Agent memory specifically — single facts, events, preferences | Requires disciplined write hygiene at ingestion time |

> **Agent memory design principle:** For agent memory systems, avoid chunking altogether by designing it out at write time. The natural unit of agent memory is the event record or fact statement — a single, semantically complete observation written as one record. Writing well-bounded records at ingestion produces records that are already chunks. **Chunking at retrieval time is a symptom of poorly structured writes at ingestion time.**

### Why

- Embedding model selection is permanent. A model baked into the vector store cannot be changed without re-embedding the entire corpus — including cold-tier records. This is an O(N) re-ingestion job that halts retrieval quality improvements while running.
- Mixed-model vector stores do not throw errors. They silently return nonsense results because a vector generated by `text-embedding-3-large` is geometrically incomparable to one generated by `bge-large-en-v1.5`.
- Chunking quality is the invisible ceiling on retrieval quality. The best reranker cannot compensate for records split mid-sentence such that neither half is retrievable by a natural query.

### How (Java + LangGraph)

```java
// SPEC RULE VEC-001
// EmbeddingService — single interface for all embedding calls
public interface EmbeddingService {

    /**
     * Embed a single text string. Throws EmbeddingException if the model
     * is unavailable or the input exceeds the token limit.
     */
    float[] embed(String text);

    /**
     * Batch embed a list of strings. More efficient than N individual calls.
     */
    List<float[]> embedBatch(List<String> texts);

    /** Returns the declared output dimension of this model. */
    int dimensions();

    /** Returns true if the model produces unit-normalised vectors. */
    boolean isNormalised();
}
```

**Design rules:**
- `VEC-001` — `EmbeddingService` MUST be a single Spring `@Bean`. No component calls an embedding API directly. All embedding flows through this bean.
- `VEC-002` — The embedding model name, version, and output dimension MUST be declared in `application.yml` and validated at startup against the live model API. A dimension mismatch at startup is a fatal error.
- `VEC-003` — Mixing embedding models within the same vector store collection is prohibited. If a model change is required, a `CorpusReEmbeddingJob` MUST be executed on a shadow index before the primary index is replaced.
- `VEC-004` — Every memory record written to the vector store MUST include a `embeddingModelId` metadata field (e.g., `text-embedding-3-large-v1`). This enables detection of mixed-model pollution.
- `VEC-005` — Agent memory MUST be written as single-fact or single-event records at ingestion time. Records that contain multiple independent facts MUST be split at write time by `MemoryWriteService`, not at retrieval time.

---

## 11.2 — Similarity Metrics: The Model-Store Contract

### Context

The similarity metric is a **contract between the embedding model and the vector database**. It must be decided once, locked in at database creation, and never changed without a full re-index. The three primary metrics are cosine similarity, dot product, and Euclidean distance. Each has a specific domain of applicability for text embeddings.

### What

**Cosine similarity** measures the angle between two vectors, ignoring magnitudes:

\[ \cos(A, B) = \frac{A \cdot B}{\|A\| \|B\|} = \frac{\sum_i A_i B_i}{\sqrt{\sum_i A_i^2} \cdot \sqrt{\sum_i B_i^2}} \]

Range: \([-1.0, +1.0]\), higher is more similar. Cosine is robust to documents of varying length because it normalises out magnitude.

**Dot product similarity** is the raw inner product:

\[ \text{dot}(A, B) = A \cdot B = \sum_i A_i B_i \]

When embeddings are unit-normalised (\(\|A\| = \|B\| = 1\)), dot product and cosine similarity are mathematically equivalent — the denominator of the cosine formula becomes \(1 \times 1 = 1\). If embeddings are **not** normalised, dot product mixes directional similarity with vector magnitude.

**Euclidean distance** measures straight-line distance between two points:

\[ d(A, B) = \sqrt{\sum_i (A_i - B_i)^2} \]

Lower distance = more similar. Magnitude-sensitive. Appropriate for image embeddings and spatial representations. **Not recommended for standard NLP text embeddings**.

**Similarity metric selection guide:**

| Metric | Use when | Avoid when |
|---|---|---|
| **Cosine similarity** | Standard text embeddings; documents of varying length; default/safe choice | Embedding model is explicitly designed for dot product |
| **Dot product** | Unit-normalised text embeddings; performance-critical systems where query latency matters | Embeddings are not normalised to unit length |
| **Euclidean distance** | Image embeddings; spatial/coordinate representations | Standard NLP text embeddings |

> **Efficiency tip:** If your embedding model produces unit-normalised vectors, prefer dot product over cosine. It is computationally cheaper at query time (no normalisation step). Verify normalisation: embed the same sentence twice, compute `v · v`. If the result is `1.0`, dot product is equivalent and faster.

### Why

Metric mismatches do not throw errors. They silently degrade retrieval quality in a way that is extremely difficult to diagnose — results look plausible but relevance is consistently wrong. This class of failure often appears as a regression in agent response quality weeks after a misconfiguration.

### How (Java + LangGraph)

```java
// SPEC RULE VEC-010
// SimilarityMetric enum — declared once, used everywhere
public enum SimilarityMetric {
    COSINE,
    DOT_PRODUCT,
    EUCLIDEAN
}

// VectorStoreConfig — locks in metric at database creation
@ConfigurationProperties(prefix = "memory.vector-store")
public record VectorStoreConfig(
    String  collectionName,
    int     dimensions,
    SimilarityMetric metric,          // locked at creation; changing requires full re-index
    boolean embeddingsAreNormalised   // if true, dot product preferred over cosine
) {}
```

**Design rules:**
- `VEC-010` — The similarity metric MUST be declared in `VectorStoreConfig` and validated against the embedding model's `isNormalised()` result at startup. If `isNormalised() == true` and `metric == COSINE`, a startup warning MUST be emitted recommending dot product.
- `VEC-011` — Metric selection MUST be documented in the agent architecture document as a permanent data contract. Changing the metric after records are stored requires full re-indexing and is treated as a breaking change.
- `VEC-012` — `EUCLIDEAN` MUST NOT be selected for memory stores backed by text embedding models unless an explicit architectural decision log entry justifies the exception.

---

## 11.3 — Vector Database Architecture: ANN, HNSW, and IVF

### Context

A vector database is a storage system built to store high-dimensional vectors and execute nearest-neighbour queries against them at scale. The core query — *"find the top-k records most similar to this query vector"* — cannot be answered efficiently by a B-tree, hash index, or standard SQL engine. It requires specialised data structures for high-dimensional Approximate Nearest-Neighbour (ANN) search.

### What — Three-Layer Architecture

A production vector database combines three layers that must work together correctly:

| Layer | Role | Engineering concern |
|---|---|---|
| **Vector index** | The data structure that makes similarity search fast (HNSW or IVF) | Recall vs. latency trade-off; memory consumption |
| **Metadata store** | Per-record attributes: userId, memory type, timestamp, importance score, TTL, tags | Enables filtered queries; hooks for Chapter 12's scoring and TTL policies |
| **Persistence layer** | Durable storage for vectors and metadata with replication and crash recovery | Write durability; multi-region distribution |

### What — HNSW Index

HNSW (Hierarchical Navigable Small World) is a **graph-based ANN index**. It organises vectors as nodes in a layered graph where each node is connected to a fixed number of its approximate nearest neighbours. The hierarchy has multiple layers: the top layer is sparse with long-range connections for coarse navigation; the bottom layer is dense with fine-grained local connections for precise retrieval.

**HNSW production characteristics:**
- Query latency in single-digit milliseconds at 100 M records.
- Recall typically 95–99% at default `ef` settings.
- Memory-intensive — the full graph structure lives in RAM.
- Handles dynamic inserts without requiring full index rebuild.
- Slow to build initially for large datasets — plan index construction time into deployment schedules.

> **HNSW is the dominant index for agent memory hot-tier workloads** because agent memory stores are dynamic — records are written frequently and must be immediately retrievable.

### What — IVF Index

IVF (Inverted File Index) is a **clustering-based ANN index**. It uses k-means to partition the vector space into `nlist` clusters. Each record is stored in the cluster whose centroid it is nearest to. At query time, only the `nprobe` clusters nearest to the query are searched.

**IVF variants:**

| Variant | Storage | Recall | Memory | Use for |
|---|---|---|---|---|
| **IVF-FLAT** | Raw full-precision vectors | 95–99% | High | Warm-tier; moderate scale |
| **IVF-PQ** | Lossy quantised vectors | 70–90% | Low (up to 64x compression) | Cold-tier archival storage |

### What — HNSW vs. IVF Comparison

| Property | HNSW | IVF-FLAT | IVF-PQ |
|---|---|---|---|
| Query speed | Very fast (~1–10 ms) | Moderate (20–50 ms) | Moderate |
| Recall | 95–99% | 95–99% | 70–90% |
| Memory usage | High (graph in RAM) | High (raw vectors) | Low (compressed) |
| Index build time | Slow | Fast | Fast |
| Dynamic inserts | Excellent | Poor (rebuild required) | Poor (rebuild required) |
| Best for | Dynamic hot-tier agent memory | Static batch corpora | Archival cold-tier storage |

### What — Metadata Filtering Architectures

Production agent queries almost always combine similarity search with metadata filters: *"retrieve records for this specific user, written within the last 30 days, tagged as semantic facts, with importance score above 0.4."*

| Architecture | Mechanism | Risk |
|---|---|---|
| **Pre-filtering** | Apply metadata filter first, then run ANN on the surviving subset | Expensive — ANN index traversed over a restricted node set it was not built for |
| **Post-filtering** | Run ANN to get top-k candidates, then discard non-matching records | **Recall collapse** when filter is highly selective — silently returns fewer results than k with no error |
| **Integrated filtering** | Metadata constraints embedded directly into ANN index traversal (Qdrant's model) | Best recall and latency — avoids both above risks |

> ⚠️ **Post-filtering recall collapse** is the most common source of unexplained retrieval failures in high-selectivity workloads. If you filter to a specific user's records and that user has fewer records than the requested k, post-filtering silently returns an incomplete result. Verify your database's filtering architecture before writing production query logic.

### Why

- Teams that treat a vector database as "a database with a vector column" miss the three-layer architecture and build systems where metadata filtering, replication, and index behaviour interact in unexpected ways. The most common manifestation is intermittent recall degradation that cannot be reproduced in test because the test dataset is small enough for brute-force fallback.
- HNSW's memory requirement is load-bearing for capacity planning. A 1 M-record store with 1,536-dim float32 vectors requires approximately 6 GB of RAM for the raw vectors alone, plus 20–40% overhead for the HNSW graph structure.

### How (Java + LangGraph)

```java
// SPEC RULE VEC-020
// VectorStore — the single interface for all vector DB operations
public interface VectorStore {

    RecordId upsert(VectorRecord record);

    List<ScoredRecord> query(float[] queryVector,
                             int topK,
                             MetadataFilter filter);

    void delete(RecordId id);

    void expireByTtl(Instant now);
}

// VectorRecord schema
public record VectorRecord(
    RecordId        id,
    float[]         vector,
    String          content,           // formatted summary for context injection
    MemoryType      type,
    String          userId,
    String          embeddingModelId,  // VEC-004
    double          importanceScore,
    Instant         createdAt,
    Instant         expiresAt,         // null = no TTL
    List<String>    tags
) {}

// MetadataFilter — composable, typed
public record MetadataFilter(
    String          userId,
    MemoryType      type,
    Instant         createdAfter,
    Instant         createdBefore,
    Double          minImportanceScore,
    List<String>    tags
) {}
```

**Design rules:**
- `VEC-020` — `VectorStore` MUST be a Spring `@Bean`. No component interacts with the vector database directly — all access via this interface.
- `VEC-021` — The vector store MUST use HNSW for the hot-tier collection (active episodic and semantic records). IVF-PQ MAY be used for the cold-tier archive collection (Chapter 12).
- `VEC-022` — Integrated metadata filtering MUST be used if the chosen vector database supports it (Qdrant, Weaviate, Redis VSS). Post-filtering MUST NOT be used as the default strategy for userId or memoryType filters.
- `VEC-023` — `VectorStore.query()` MUST emit a `VectorQueryEvent` with: `queryId`, `topK`, `filterFields`, `resultCount`, `latencyMs`, `recallEstimate`. Published to Prometheus as `vector_query_latency_ms` histogram.
- `VEC-024` — If `resultCount < topK` and a metadata filter was applied, a `RECALL_SHORTFALL` warning MUST be logged. This is always a filtering architecture issue, never silently ignored.

---

## 11.4 — Knowledge Graphs: Relational Semantic Memory

### Context

Vector similarity retrieves records that are *semantically similar* to a query. Knowledge graphs retrieve records that are *relationally connected* to an entity. These are fundamentally different access patterns, and for a class of production agent queries — *"What are all the constraints related to this user's deployment environment?"* — knowledge graph traversal is more precise and more complete than vector similarity search.

### What — Graph Data Model

A knowledge graph models the world as **nodes** (entities) and **edges** (typed relationships between entities). Every edge is directed and labelled.

**Triple format (the fundamental unit):**
```
(subject, predicate, object)
(user:4421, PREFERS, language:TypeScript)
(user:4421, OPERATES_IN, environment:AWS-eu-west-1)
(environment:AWS-eu-west-1, PROHIBITS, service:public-internet-egress)
```

**Property graph model (extends triples with properties on nodes and edges):**

| Element | Example | Properties |
|---|---|---|
| Node | `user:4421` | `createdAt`, `tier`, `region` |
| Node | `language:TypeScript` | `version`, `paradigm` |
| Edge | `(user:4421)-[PREFERS {since: 2025-01-10, confidence: 0.9}]->(language:TypeScript)` | `since`, `confidence`, `source` |

### What — Multi-Hop Reasoning

The power of knowledge graphs is **multi-hop traversal** — following relationship chains to answer questions that require joining multiple facts:

**Query:** *"What services are prohibited in this user's deployment environment?"*

```cypher
MATCH (u:User {id: "4421"})
      -[:OPERATES_IN]->(env:Environment)
      -[:PROHIBITS]->(svc:Service)
RETURN svc.name, svc.category
```

This retrieves `public-internet-egress` as a prohibited service — a fact that cannot be retrieved by vector similarity alone, because no single stored record says *"this user cannot use public-internet-egress"*. The constraint emerges from a two-hop traversal.

**Cypher-style patterns for agent memory queries:**

| Query intent | Traversal pattern |
|---|---|
| User preferences | `(u:User)-[:PREFERS]->(entity)` |
| User constraints | `(u:User)-[:OPERATES_IN]->(env)-[:PROHIBITS]->(entity)` |
| Task dependencies | `(task:Task)-[:DEPENDS_ON*1..3]->(resource)` |
| Conflict detection | `(u:User)-[:HOLDS]->(b1:Belief), (u:User)-[:HOLDS]->(b2:Belief) WHERE b1 CONTRADICTS b2` |
| Historical patterns | `(u:User)-[:EXPERIENCED {outcome: 'FAILURE'}]->(scenario)` |

### What — When to Use Knowledge Graphs vs. Vector Search

| Retrieval need | Use knowledge graph | Use vector search |
|---|---|---|
| "What are all facts about entity X?" | ✅ Direct node lookup | ❌ Misses relational facts |
| "What's related to X by 2–3 hops?" | ✅ Graph traversal | ❌ Not possible |
| "What records are semantically similar to this query?" | ❌ Not designed for this | ✅ Core strength |
| "What does the agent remember about topic Y?" | ❌ Only if topic is modelled as entity | ✅ Core strength |
| "What constraints apply to this entity?" | ✅ Traversal via constraint edges | ❌ Misses implicit constraints |

### What — Cross-Store Consistency Obligation

When both a vector store and a knowledge graph are maintained, the same semantic fact exists in two representations. **Both must be updated atomically or the system exhibits split-brain:** the vector store says one thing and the knowledge graph says another.

**Cross-store write protocol (mandatory for dual-store deployments):**
1. Write to vector store (primary).
2. Write to knowledge graph (secondary).
3. If step 2 fails: queue a compensating write via `CrossStoreConsistencyQueue`; do not silently discard.
4. `CrossStoreConsistencyMonitor` runs every 5 minutes comparing entity counts and emits `CONSISTENCY_DRIFT` alerts.

### Why

- Knowledge graphs are the correct technology for user modelling, constraint management, and multi-hop reasoning in production agents. Teams that model all memory as unstructured text in a vector store cannot express the relational structure that enterprise agent use cases require.
- The cross-store consistency obligation is the most underspecified aspect of hybrid stores. A delete in the vector store that is not reflected in the knowledge graph will cause the agent to retrieve contradictory facts — a high-confidence semantic memory belief from the graph and no supporting evidence from the vector store.

### How (Java + LangGraph)

```java
// SPEC RULE VEC-030
// KnowledgeGraphStore — interface for all graph operations
public interface KnowledgeGraphStore {

    NodeId upsertNode(String label, String entityId, Map<String, Object> properties);

    void upsertEdge(NodeId from, String relationshipType,
                    NodeId to, Map<String, Object> properties);

    List<GraphRecord> traverse(String startEntityId,
                               String relationshipPattern,
                               int maxHops);

    void deleteNode(NodeId id);
}

// GraphRecord — result of a traversal
public record GraphRecord(
    String path,           // e.g., "user:4421 -[OPERATES_IN]-> env:AWS-eu-west-1"
    List<String> entities,
    Map<String, Object> edgeProperties
) {}
```

**Design rules:**
- `VEC-030` — `KnowledgeGraphStore` MUST be a Spring `@Bean`. No component accesses the graph database directly.
- `VEC-031` — All writes that target both the vector store and the knowledge graph MUST be mediated by `DualStoreMemoryWriter`, which implements the cross-store write protocol (steps 1–4 above).
- `VEC-032` — `CrossStoreConsistencyMonitor` MUST be deployed as a Spring `@Scheduled` component from the first dual-store deployment. Consistency drift MUST emit a Prometheus counter `memory_consistency_drift_total`.
- `VEC-033` — Max traversal depth MUST be bounded at `VEC-033: maxHops` (default 3). Unbounded traversals are prohibited in production query paths.
- `VEC-034` — Knowledge graph nodes MUST carry the same `embeddingModelId`-equivalent field for the vector representation of the same fact, enabling cross-store lookup by entity ID.

---

## 11.5 — Hybrid Retrieval Pipelines

### Context

Neither vector similarity nor keyword search alone achieves the recall and precision required for production agent memory retrieval. Vector search excels at semantic similarity but misses exact matches and rare technical terms. Keyword (BM25) search excels at precise term matching but misses paraphrases and synonyms. Knowledge graph traversal excels at relational queries but misses unstructured records. **Hybrid retrieval combines all three** to achieve the 2026 production standard.

### What — The Four-Method Taxonomy

| Method | Mechanism | Best for | Miss rate on |
|---|---|---|---|
| **BM25 (keyword)** | TF-IDF weighted term frequency | Exact technical terms, named entities, version numbers | Paraphrases, synonyms, conceptually related queries |
| **Dense vector search** | ANN nearest-neighbour on query embedding | Semantic similarity, paraphrases, concept-level queries | Exact rare terms, low-frequency tokens |
| **Sparse vector (SPLADE)** | Learned sparse expansion of lexical features | Balance of exact and semantic | Complex multi-hop relational queries |
| **Graph traversal** | Cypher/Gremlin multi-hop relationship following | Relational facts, constraint chains, entity-level queries | Unstructured narrative records |

### What — The Production Baseline: BM25 + Dense + RRF

The 2026 production baseline for agent memory retrieval is a **two-path parallel pipeline** fused with Reciprocal Rank Fusion (RRF):

```
Query
  ├─► BM25 search ───────────────────────────────┐
  │    (top-20 keyword candidates)               ├─► RRF fusion ─► top-5 selected ─► inject
  └─► Dense vector search ─────────────────────-┘
       (top-20 similarity candidates)
```

**Reciprocal Rank Fusion formula:**

\[ \text{RRF}(d) = \sum_{r \in R} \frac{1}{k + r(d)} \]

where \( R \) is the set of result lists (BM25, dense), \( r(d) \) is the rank of document \( d \) in list \( r \), and \( k \) is a smoothing constant (typically \( k = 60 \)).

RRF is robust: it does not require score normalisation across retrieval methods and outperforms linear interpolation of raw scores in most benchmarks.

### What — HyDE: Hypothetical Document Embeddings

HyDE (Gao et al., 2022) is a query augmentation technique for abstract or underspecified queries:

1. **Generate a hypothetical answer** to the query using the LLM (no tools, pure generation).
2. **Embed the hypothetical answer** rather than the original query.
3. **Run standard dense retrieval** with the hypothetical answer's embedding.

**Use when:** The query is short, abstract, or uses different vocabulary than the stored records.

**Example:**
- Query: *"deployment constraints"*
- Hypothetical answer: *"The user operates in AWS eu-west-1 and is prohibited from using public-internet-egress services due to enterprise security policy."*
- The hypothetical answer's embedding retrieves the actual constraint records more accurately than the 2-word query embedding.

> ⚠️ **HyDE adds one LLM call** to the retrieval pipeline. Gate it behind `retrieval.hyde.enabled` and enable it only for task classes where query underspecification is a measured recall problem.

### What — GraphRAG: Knowledge-Graph-Grounded Retrieval

GraphRAG (Edge et al., Microsoft, 2024) extends hybrid retrieval with graph traversal as a third retrieval path:

```
Query
  ├─► BM25 search ─────────────────────────────────────┐
  ├─► Dense vector search ─────────────────────────────┤
  └─► Graph traversal ─────────────────────────────────┤
       (entity extraction ─► multi-hop traversal)      ├─► RRF/merge ─► rerank ─► inject
```

The graph traversal path:
1. Extract named entities from the query using NER.
2. Look up each entity in the knowledge graph.
3. Traverse up to `maxHops` hops following relevant edge types.
4. Collect graph records as a third candidate list.
5. Merge with BM25 and dense candidates before reranking.

**Use when:** The agent's semantic memory is highly relational (user profiles, constraint hierarchies, organisational structures, multi-entity workflows).

### What — Reranking: Precision After Recall

ANN retrieval is optimised for **recall** (finding relevant records in the top-k). Reranking is a second stage optimised for **precision** (ranking the truly relevant records above marginally relevant ones).

**Two-stage retrieval architecture:**

| Stage | Goal | Model type | Latency | Scale |
|---|---|---|---|---|
| **Stage 1: Retrieval** | Recall — find all potentially relevant records | Bi-encoder (embedding similarity) | ~5–20 ms | All records |
| **Stage 2: Reranking** | Precision — rank the retrieved candidates correctly | Cross-encoder (full attention) | ~50–200 ms | Top-20 from Stage 1 |

**Cross-encoder rerankers (2026):**
- `Cohere Rerank 3` — highest general-purpose benchmark score.
- `BAAI/bge-reranker-v2-m3` — best open-source option; strong multilingual performance.
- `Jina Reranker v2` — optimised for long-document reranking.

> **Reranking is mandatory for production agents.** The difference in precision between a bi-encoder top-5 and a cross-encoder reranked top-5 is typically 15–25 percentage points on retrieval benchmarks. Skipping reranking means injecting the less relevant 20% of retrieved records into the context on a non-trivial fraction of queries.

### Why

- Single-method retrieval produces systematically wrong injections for different query types. An agent that only uses dense retrieval misses exact version numbers and named entities. An agent that only uses BM25 misses paraphrased facts. These failure modes are task-class-specific and not evenly distributed — they are concentrated on exactly the query types where accurate context is most important.
- RRF's robustness to score normalisation is operationally significant. Raw score distributions from BM25 (unbounded) and dense search (cosine: −1 to +1) are incomparable. Linear interpolation requires empirical weight tuning that breaks when record volume or model changes. RRF rank-normalises implicitly.

### How (Java + LangGraph)

```java
// SPEC RULE VEC-040
// HybridRetrievalPipeline — the production retrieval implementation
@Component
public class HybridRetrievalPipeline {

    private static final int CANDIDATE_K   = 20;
    private static final int INJECTION_K   = 5;
    private static final int RRF_K_SMOOTH  = 60;

    private final VectorStore         vectorStore;
    private final BM25Index            bm25Index;
    private final KnowledgeGraphStore  graphStore;
    private final RerankerService      reranker;
    private final HyDEService          hydeService;   // optional; gated by config
    private final HybridRetrievalConfig config;

    public List<MemoryRecord> retrieve(String query, MetadataFilter filter) {
        String effectiveQuery = config.hydeEnabled()
            ? hydeService.generateHypotheticalAnswer(query)
            : query;

        List<ScoredRecord> denseResults  = vectorStore.query(
            embeddingService.embed(effectiveQuery), CANDIDATE_K, filter);
        List<ScoredRecord> bm25Results   = bm25Index.query(query, CANDIDATE_K, filter);
        List<ScoredRecord> graphResults  = config.graphEnabled()
            ? graphStore.traverseFromQuery(query, config.maxGraphHops())
            : List.of();

        List<ScoredRecord> fused  = rrfFuse(denseResults, bm25Results, graphResults,
                                             RRF_K_SMOOTH);
        List<ScoredRecord> reranked = reranker.rerank(query, fused.subList(0, CANDIDATE_K));

        return reranked.subList(0, INJECTION_K).stream()
            .map(ScoredRecord::record)
            .toList();
    }
}
```

**Design rules:**
- `VEC-040` — `HybridRetrievalPipeline` is the ONLY component that queries the vector store and BM25 index for agent memory retrieval. `MemoryInjectionNode` calls this pipeline exclusively.
- `VEC-041` — RRF MUST be used as the fusion strategy. Raw score interpolation is prohibited.
- `VEC-042` — HyDE MUST be gated by `retrieval.hyde.enabled` (default `false`). Enabling it requires documenting the query class that drives the decision.
- `VEC-043` — GraphRAG path MUST be gated by `retrieval.graph.enabled` (default `false`). Enable only when knowledge graph is deployed.
- `VEC-044` — Cross-encoder reranking MUST be applied before the final top-k selection. Returning bi-encoder top-k directly to `MemoryInjectionNode` is prohibited in production.
- `VEC-045` — `HybridRetrievalPipeline.retrieve()` MUST be instrumented with: `retrieval_pipeline_latency_ms` (histogram, broken down by stage: bm25, dense, graph, rerank), `retrieval_empty_rate` (gauge), `retrieval_recall_shortfall_total` (counter for when `resultCount < INJECTION_K`).

---

## 11.6 — Retrieval Quality Measurement

### Context

Retrieval quality is the invisible control variable over agent behaviour quality. An agent that retrieves wrong context will reason incorrectly even with a perfect reasoning engine. Measuring retrieval quality is not an optional analytical exercise — it is a mandatory operational discipline, required before any retrieval system is promoted to production.

### What — Metrics

**Four primary retrieval quality metrics:**

| Metric | Formula | What it measures | Target |
|---|---|---|---|
| **Precision@k** | \( \frac{\text{relevant in top-}k}{k} \) | Fraction of retrieved records that are relevant | ≥ 0.70 at k=5 |
| **Recall@k** | \( \frac{\text{relevant in top-}k}{\text{total relevant}} \) | Fraction of all relevant records captured in top-k | ≥ 0.80 at k=20 |
| **Mean Reciprocal Rank (MRR)** | \( \frac{1}{|Q|} \sum_{i=1}^{|Q|} \frac{1}{\text{rank}_i} \) | Average rank of the first relevant result across queries | ≥ 0.75 |
| **NDCG@k** | \( \frac{\text{DCG@k}}{\text{IDCG@k}} \) | Graded relevance — rewards high-relevance records ranked first | ≥ 0.70 at k=5 |

### What — Golden Retrieval Set

All retrieval quality metrics require a **golden retrieval set**: a curated collection of (query, relevant-records) pairs drawn from your actual memory corpus and validated by domain experts or by observing agent behaviour outcomes.

**Golden set construction (4 steps):**
1. Sample 200–500 representative queries from production query logs or agent run histories.
2. For each query, annotate which stored records are relevant (binary or graded relevance).
3. Stratify by query type: semantic similarity queries, exact match queries, relational queries, abstract queries.
4. Version-control the golden set as a test artifact. Update when the memory corpus changes materially.

> The golden set is the most valuable test artifact in the entire memory system. Teams that skip it ship retrieval quality regressions silently.

### What — Continuous Monitoring

| Signal | Measurement | Alert threshold |
|---|---|---|
| Empty result rate | `retrieval_empty_rate` (% of queries returning 0 results) | > 2% |
| Recall shortfall rate | % of queries where `resultCount < topK` | > 5% |
| Rerank inversion rate | % of queries where reranker reorders top-1 | Track as leading indicator |
| Latency p99 | `retrieval_pipeline_latency_ms p99` | > 500 ms |

### How (Java + LangGraph)

```java
// SPEC RULE VEC-050
// RetrievalQualityEvaluator — run against golden set in CI and staging
@Component
public class RetrievalQualityEvaluator {

    public RetrievalQualityReport evaluate(
            List<GoldenQuery> goldenSet,
            HybridRetrievalPipeline pipeline) {

        double precisionSum = 0, recallSum = 0, mrr = 0, ndcg = 0;

        for (GoldenQuery gq : goldenSet) {
            List<MemoryRecord> retrieved = pipeline.retrieve(
                gq.query(), gq.filter());
            precisionSum += precisionAtK(retrieved, gq.relevantIds(), 5);
            recallSum    += recallAtK(retrieved, gq.relevantIds(), 20);
            mrr          += reciprocalRank(retrieved, gq.relevantIds());
            ndcg         += ndcgAtK(retrieved, gq.gradedRelevance(), 5);
        }

        int n = goldenSet.size();
        return new RetrievalQualityReport(
            precisionSum / n, recallSum / n, mrr / n, ndcg / n);
    }
}

public record RetrievalQualityReport(
    double precisionAt5,
    double recallAt20,
    double mrr,
    double ndcgAt5
) {
    /** Returns true if all metrics meet production entry thresholds. */
    public boolean meetsProductionThresholds() {
        return precisionAt5 >= 0.70
            && recallAt20   >= 0.80
            && mrr          >= 0.75
            && ndcgAt5      >= 0.70;
    }
}
```

**Design rules:**
- `VEC-050` — A golden retrieval set of ≥ 200 queries MUST exist before the memory retrieval system is promoted to production. CI MUST run `RetrievalQualityEvaluator` on every PR that touches retrieval logic.
- `VEC-051` — All four metrics MUST meet production thresholds (`meetsProductionThresholds() == true`) before a retrieval change is merged. A failing quality report blocks merge.
- `VEC-052` — Retrieval quality MUST be re-evaluated after every embedding model change, vector store schema change, or pruning run that removes > 1% of the corpus.
- `VEC-053` — `retrieval_empty_rate > 2%` MUST trigger a PagerDuty alert. Empty retrieval is not a no-op — it means the agent reasons without memory context.

---

## 11.7 — Production Vector Database Selection

### Context

Four vector databases dominate production agent memory deployments in 2026. Selection is driven by filtering architecture, operational model (managed vs. self-hosted), consistency guarantees, and the team's existing infrastructure.

### What — 2026 Production Options

| Database | Filtering architecture | Index types | Operational model | Best for |
|---|---|---|---|---|
| **Qdrant** | Integrated (payload-aware traversal) | HNSW | Self-hosted / Cloud | Best filtering; integrated sparse + dense hybrid |
| **Weaviate** | Integrated (object-level filters) | HNSW | Self-hosted / Cloud | Schema-first; GraphQL API; multi-modal |
| **pgvector** (PostgreSQL) | SQL WHERE clause (pre-filter) | HNSW, IVF | Self-hosted / RDS | Existing PostgreSQL infrastructure; ACID transactions |
| **Pinecone** | Metadata filter (post-filter default) | Proprietary ANN | Managed only | Fully managed; zero ops; low team DB expertise |

**Selection decision matrix:**

| Scenario | Recommended | Rationale |
|---|---|---|
| New project, no infrastructure constraints, best retrieval quality | **Qdrant** | Integrated filtering; built-in sparse + dense hybrid; open-source |
| Existing PostgreSQL team; ACID compliance required | **pgvector** | SQL familiarity; transactional writes; pre-filter acceptable at < 1 M records |
| Multi-modal memory (text + images); rich schema | **Weaviate** | Native multi-modal support; declarative schema |
| Smallest possible operational footprint; fully managed | **Pinecone** | Zero infrastructure; managed scaling; acceptable for < 10 M records |

**Critical selection rules:**

| Rule | Binding constraint |
|---|---|
| pgvector at scale | HNSW in pgvector requires index rebuild for schema changes. At > 5 M records, pre-filter performance degrades significantly. Migrate to Qdrant or Weaviate before hitting this threshold. |
| Pinecone post-filtering | Pinecone's default post-filtering causes recall collapse at high selectivity. Namespace-partition by userId to mitigate before scale. |
| Qdrant sparse + dense | Native SPLADE-style sparse vector support in Qdrant 1.7+ enables true hybrid BM25-equivalent retrieval within a single index. Use this before maintaining a separate BM25 index. |

### Why

Database selection has a high switching cost — changing vector databases after the corpus is loaded requires full re-ingestion, re-embedding, and re-testing. A team that starts with Pinecone because it is the easiest to provision, then hits post-filtering recall collapse at 10 M records and needs to migrate to Qdrant, absorbs a multi-week engineering project at a moment of high operational pressure.

### How (Java + LangGraph)

**Design rules:**
- `VEC-060` — Vector database selection MUST be documented in the agent architecture document with a justification referencing the decision matrix above.
- `VEC-061` — The selected database MUST be validated against the following checklist before first production write: (a) integrated or pre-filter architecture confirmed; (b) metric selected and verified against embedding model; (c) HNSW configured for hot-tier collection; (d) metadata schema defined including all five mandatory fields from VEC-052.
- `VEC-062` — A load test MUST be executed against the selected database at 2× projected production volume before go-live. The test MUST measure p99 query latency and recall@5 under concurrent query load.
- `VEC-063` — The `VectorStore` Spring `@Bean` MUST be backed by a concrete implementation class named `[Database]VectorStore` (e.g., `QdrantVectorStore`, `PgVectorStore`). The implementation class is hidden behind the `VectorStore` interface from all consumers, enabling database migration without changing application code.

---

## 11.8 — Design Rules Summary

| Rule ID | Scope | Rule |
|---|---|---|
| `VEC-001` | Embedding | `EmbeddingService` MUST be a single Spring `@Bean`. No direct embedding API calls. |
| `VEC-002` | Embedding | Embedding model name, version, dimension declared in `application.yml`; validated at startup. |
| `VEC-003` | Embedding | Mixing models in the same collection is prohibited. Model changes require `CorpusReEmbeddingJob` on shadow index. |
| `VEC-004` | Embedding | Every vector record MUST include `embeddingModelId` metadata field. |
| `VEC-005` | Embedding | Records with multiple independent facts MUST be split at write time by `MemoryWriteService`. |
| `VEC-010` | Metrics | `SimilarityMetric` declared in `VectorStoreConfig`; validated at startup. |
| `VEC-011` | Metrics | Metric selection documented as permanent data contract in architecture document. |
| `VEC-012` | Metrics | `EUCLIDEAN` prohibited for text embedding stores without documented exception. |
| `VEC-020` | Vector DB | `VectorStore` MUST be a Spring `@Bean`. No direct DB access from other components. |
| `VEC-021` | Vector DB | HNSW for hot-tier; IVF-PQ permitted for cold-tier archive. |
| `VEC-022` | Vector DB | Integrated metadata filtering MUST be used; post-filtering prohibited as default for high-selectivity filters. |
| `VEC-023` | Vector DB | `VectorQueryEvent` emitted on every query with latency and result count. |
| `VEC-024` | Vector DB | `RECALL_SHORTFALL` warning logged when `resultCount < topK` under a metadata filter. |
| `VEC-030` | Graph | `KnowledgeGraphStore` MUST be a Spring `@Bean`. |
| `VEC-031` | Graph | Dual-store writes MUST use `DualStoreMemoryWriter` with cross-store protocol. |
| `VEC-032` | Graph | `CrossStoreConsistencyMonitor` deployed from first dual-store deployment. |
| `VEC-033` | Graph | Max traversal depth bounded at `maxHops` (default 3). |
| `VEC-034` | Graph | Graph nodes MUST carry entity-ID cross-reference to vector store records. |
| `VEC-040` | Hybrid | `HybridRetrievalPipeline` is the only retrieval entrypoint. |
| `VEC-041` | Hybrid | RRF MUST be used as fusion strategy. Raw score interpolation prohibited. |
| `VEC-042` | Hybrid | HyDE gated by `retrieval.hyde.enabled` (default `false`). |
| `VEC-043` | Hybrid | GraphRAG gated by `retrieval.graph.enabled` (default `false`). |
| `VEC-044` | Hybrid | Cross-encoder reranking MUST be applied before final top-k injection. |
| `VEC-045` | Hybrid | Pipeline MUST emit per-stage latency, empty-rate, and shortfall metrics. |
| `VEC-050` | Quality | Golden retrieval set of ≥ 200 queries required before production promotion. |
| `VEC-051` | Quality | All four metrics must meet thresholds; failing report blocks merge. |
| `VEC-052` | Quality | Quality re-evaluated after model change, schema change, or > 1% corpus pruning. |
| `VEC-053` | Quality | `retrieval_empty_rate > 2%` triggers PagerDuty alert. |
| `VEC-060` | DB Selection | Database selection documented in architecture document with decision matrix justification. |
| `VEC-061` | DB Selection | Pre-production checklist validation required before first production write. |
| `VEC-062` | DB Selection | Load test at 2× projected volume required before go-live. |
| `VEC-063` | DB Selection | `VectorStore` interface backed by named implementation class enabling zero-application-code migration. |

---

## 11.9 — Transition to Chapter 12

This chapter built the complete external memory retrieval machinery: how embeddings encode meaning, how similarity metrics define proximity, how vector databases index and serve nearest-neighbour queries, how knowledge graphs complement similarity with relational structure, how hybrid pipelines combine all three, how to measure and gate retrieval quality, and how to select among the four production vector databases.

The retrieval machinery answers one half of the external memory engineering challenge: *how to get records back when they are needed*. Chapter 12 addresses the other half: *how to keep the memory store accurate, current, and bounded over time*.

Chapter 12 covers:
- **Consolidation** — how episodic records are distilled into durable semantic facts.
- **Importance scoring** — the composite formula driving consolidation selection and pruning decisions.
- **Pruning** — how stale and low-value records are removed without degrading retrieval quality.
- **Tiered persistence** — how records migrate from hot to warm to cold storage as their importance decays.
- **Cross-store consistency** — full implementation of the consistency protocol introduced in Section 11.4.
- **Audit and compliance** — write-immutable logging for production memory operations.
