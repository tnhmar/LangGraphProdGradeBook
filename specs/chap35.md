# Chapter 35 Spec — RAG Pipelines and Multi-Agent Retrieval

> **Volume 1 reference:** Part 7 — Tooling and Integrations, Chapter 35 (final chapter of Volume 1)
> **Depends on:** chap32 (`TOOL-*`), chap33 (`ENT-*`), chap34 (`ADV-*`), chap8 (State Manager), chap10 (Memory Architecture), chap12 (Memory Lifecycle / HITL)
> **Java + LangGraph implementation target:** Volume 2 — Chapter 3: RAG Tool Handler; Chapter 4: Agentic RAG Runtime; Chapter 12: Distributed Tracing
> **Bridges to:** Volume 2, Chapter 1 (Design Principles — bridge from pattern to production)

> *"Retrieval is not a lookup. It is a judgment about what the agent needs to know, made under time pressure, with incomplete information about what the agent will ask next."*
> — Attributed to information retrieval practitioners

---

## Chapter Purpose

Part 3 established the memory substrate — vector stores, knowledge graphs, hybrid retrieval, and the lifecycle of stored knowledge. This chapter builds the layer **above** that substrate: the RAG pipeline — the sequence of operations that transforms a raw query into a grounded, ranked, and context-ready result set that the agent can reason over.

Where Part 3 asked *how is knowledge stored and retrieved?*, this chapter asks *how is retrieval designed as a production system that an agent calls as a tool?*

By the end of this chapter the reader can:
- Describe the full six-stage RAG pipeline and encapsulate it as a registered tool contract
- Implement query rewriting, decomposition, and HyDE to improve recall on complex queries
- Select and apply chunking strategies (fixed-size, hierarchical parent-child, semantic) appropriate to the corpus
- Build a hybrid retrieval pipeline (dense + sparse BM25) merged with Reciprocal Rank Fusion
- Implement cross-encoder reranking and LLM-based reranking with appropriate latency trade-offs
- Assemble context within a token budget with explicit source numbering and attribution
- Design agentic RAG patterns: iterative sufficiency checking, Corrective RAG (CRAG), and Self-RAG
- Select embedding models based on domain alignment, dimensionality, multilingual requirements, and latency/storage constraints
- Coordinate multi-agent retrieval across heterogeneous sources with deduplication and conflict annotation
- Measure RAG quality across all four RAGAS dimensions and map metric drops to pipeline stages

---

## Section 35.1 — RAG as a Tool Pipeline

### What RAG Actually Is

Retrieval-Augmented Generation (RAG) is the pattern of grounding an LLM's response in externally retrieved content rather than relying solely on parametric knowledge baked into model weights. The basic idea is simple — retrieve relevant documents, inject them into the context window, generate a grounded response. The **production implementation is not simple**.

A production RAG system is a **multi-stage pipeline**. Treating RAG as a single "search and inject" operation is the source of most RAG quality failures in production — the system works in demos because the demo queries are simple, and breaks in production because the queries are not.

### The Six Canonical Pipeline Stages

| # | Stage | Responsibility |
|---|---|---|
| 1 | **Query transformation** | Rewrite or expand the raw query to improve retrieval recall |
| 2 | **Retrieval** | Execute one or more search operations against the knowledge index |
| 3 | **Filtering** | Remove results that fail metadata, access control, or freshness criteria |
| 4 | **Reranking** | Reorder retrieved candidates by relevance using a more precise scoring model |
| 5 | **Context assembly** | Select, truncate, and format the top results for injection into the context window |
| 6 | **Attribution** | Attach source metadata to every injected passage for citation and audit provenance |

### RAG as a Registered Tool

The entire RAG pipeline is encapsulated as a registered tool with a well-defined contract. The agent calls `knowledge.retrieve` with a query and receives a ranked, attributed result set. The internal pipeline stages are invisible to the agent's reasoning loop.

```json
{
  "name": "knowledge.retrieve",
  "version": "1.0.0",
  "description": "Retrieve passages from the internal knowledge base relevant to a question or topic. Use this tool before answering any question about company policies, product documentation, technical specifications, or historical decisions. Returns ranked passages with source citations.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": { "type": "string", "description": "The question or topic to retrieve information about.", "maxLength": 1024 },
      "topK": { "type": "integer", "default": 5, "minimum": 1, "maximum": 20 },
      "filters": { "type": "object", "description": "Optional metadata filters: docType, dateRange, department.", "additionalProperties": true }
    },
    "required": ["query"]
  },
  "outputSchema": {
    "type": "array",
    "items": {
      "type": "object",
      "properties": {
        "passage": { "type": "string" },
        "sourceId": { "type": "string" },
        "title": { "type": "string" },
        "score": { "type": "number" },
        "date": { "type": "string" }
      }
    }
  },
  "sideEffects": "read-only",
  "timeoutMs": 3000,
  "maxRetries": 2
}
```

> 📝 **The RAG tool's output schema is the contract between the pipeline and the agent.** Every passage MUST carry a `sourceId` and `title`. The agent cannot produce reliable citations without them. Never return raw text blobs without provenance metadata — unattributed injected content is the primary source of agent hallucination in RAG systems.

### Design Rules

| Rule ID | Rule |
|---|---|
| `RAG-PIPE-001` | The full RAG pipeline MUST be encapsulated behind a single `knowledge.retrieve` tool contract — pipeline internals MUST NOT be exposed to the agent's reasoning loop. |
| `RAG-PIPE-002` | Every retrieved passage MUST carry `sourceId`, `title`, `score`, and `date` — unattributed passages are prohibited. |
| `RAG-PIPE-003` | The `knowledge.retrieve` tool MUST be declared `sideEffects: read-only` — retrieval MUST NOT write to any backend. |
| `RAG-PIPE-004` | The `knowledge.retrieve` tool MUST declare a `timeoutMs` of ≤ 3000 ms — multi-stage pipelines that exceed this MUST use the submit-poll pattern from chap34 Section 34.4. |

---

## Section 35.2 — Query Transformation

### Why Raw Queries Fail

The query the agent submits to the retrieval tool is rarely the optimal query for the retrieval index. A conversational query like *"what changed in the refund policy last quarter?"* contains temporal context, implicit subject reference, and colloquial phrasing that a vector index matches less reliably than an explicit query like *"refund policy updates Q4 2025"*.

Query transformation bridges this gap. It runs **before retrieval** and produces one or more improved queries from the original.

### Query Rewriting

Query rewriting uses a fast, lightweight LLM call to reformulate the raw query into a form better suited for the retrieval index.

```java
@Component
public class QueryRewriter {

    private static final String REWRITE_PROMPT =
        "You are a search query optimizer. Given a user question, rewrite it as a " +
        "concise, keyword-rich search query optimized for a document retrieval system. " +
        "Output only the rewritten query. No explanation.";

    private final LlmClient fastLlm;   // e.g., gpt-4o-mini — NOT the reasoning LLM

    public String rewrite(String rawQuery) {
        return fastLlm.complete(REWRITE_PROMPT, rawQuery,
            CompletionOptions.builder()
                .maxTokens(64)
                .temperature(0.0)
                .build()
        ).strip();
    }
}
```

### Query Decomposition

Complex questions contain multiple information needs that a single retrieval call cannot satisfy. Query decomposition breaks a multi-part question into a list of atomic sub-queries, each retrievable independently. Sub-query results are merged in the context assembly stage.

```java
@Component
public class QueryDecomposer {

    private static final String DECOMPOSE_PROMPT =
        "You are a query analyst. Given a complex question, identify the distinct " +
        "information needs it contains. Output a JSON array of simple, atomic sub-questions. " +
        "Each sub-question should be answerable from a single document. " +
        "Output only the JSON array.\n\n" +
        "Example:\n" +
        "Input: Compare our EMEA and APAC refund policies and identify any compliance gaps with GDPR.\n" +
        "Output: [\"EMEA refund policy\", \"APAC refund policy\", \"GDPR refund policy requirements\"]";

    private final LlmClient fastLlm;
    private final ObjectMapper mapper;

    public List<String> decompose(String query) {
        String response = fastLlm.complete(DECOMPOSE_PROMPT, query,
            CompletionOptions.builder().maxTokens(256).temperature(0.0).build());
        try {
            return mapper.readValue(response, new TypeReference<>() {});
        } catch (JsonProcessingException e) {
            return List.of(query);  // Fallback: use original query
        }
    }
}
```

### Hypothetical Document Embedding (HyDE)

HyDE inverts the retrieval query: instead of searching for documents similar to the question, it generates a hypothetical answer to the question and searches for documents similar to that answer. The intuition is that a document answering a question looks more like a generated answer than like the question itself.

```java
@Component
public class HydeQueryBuilder {

    private static final String HYDE_PROMPT =
        "Write a short, factual paragraph that would answer the following question. " +
        "This will be used as a retrieval query, not shown to users.";

    private final LlmClient fastLlm;
    private final EmbeddingModel embedder;

    public float[] buildHydeEmbedding(String question) {
        String hypotheticalDoc = fastLlm.complete(HYDE_PROMPT, question,
            CompletionOptions.builder().maxTokens(128).temperature(0.3).build());
        return embedder.embed(hypotheticalDoc);  // Embed the hypothetical answer, not the question
    }
}
```

> ⚠️ **HyDE has a significant failure mode:** if the hypothetical document contains hallucinated facts, the retrieval query is biased toward documents that confirm those hallucinations rather than documents that actually answer the question. Use HyDE selectively — for knowledge base queries where the question and the answer are semantically distant (technical documentation, legal policies, specification sheets). Avoid it for FAQ lookups and named entity searches where the question and document are already semantically close. Run offline evaluation before deploying in production.

### Design Rules

| Rule ID | Rule |
|---|---|
| `RAG-QT-001` | Query transformation MUST use a lightweight fast model (e.g., `gpt-4o-mini`) — the agent's primary reasoning LLM MUST NOT be used for query rewriting. |
| `RAG-QT-002` | Query decomposition MUST fall back to the original query if JSON parsing fails — a transformation error MUST NOT abort the retrieval pipeline. |
| `RAG-QT-003` | HyDE MUST only be used for corpora where offline evaluation confirms a recall improvement — deploying HyDE without offline validation is prohibited. |
| `RAG-QT-004` | Query decomposition results MUST be retrieved in parallel using the fan-out pattern from chap34 Section 34.1 — sequential sub-query retrieval is a latency defect. |

---

## Section 35.3 — Chunking and Indexing Strategy

### Why Chunking Matters

Vector retrieval operates on chunks, not documents. A 1,000-page policy manual cannot be embedded as a unit — the resulting embedding averages the semantics of the entire document and matches nothing precisely. The chunking decision is one of the most consequential choices in RAG system design.

- Chunks too large: retrieve correct documents but inject excessive context
- Chunks too small: retrieve precise passages but lose surrounding context that gives the passage meaning

### Chunking Strategy Selection Guide

| Strategy | Mechanism | Best For |
|---|---|---|
| **Fixed-size with overlap** | Split every N tokens; overlap M tokens between chunks | General-purpose baseline; homogeneous prose corpora |
| **Sentence-level** | Split on sentence boundaries using NLP tokenizer | FAQ documents, policy text, short-fact corpora |
| **Paragraph-level** | Split on paragraph or section boundaries | Technical documentation, reports with clear structure |
| **Semantic chunking** | Split where embedding similarity between adjacent sentences drops below threshold | Mixed-topic documents; inconsistently structured content |
| **Hierarchical parent-child** | Small child chunks for retrieval; larger parent chunks for context injection | Long documents where retrieval precision and context breadth are both required |
| **Document-level** | Entire document as one chunk | Short reference documents, code files, structured records |

### Hierarchical Parent-Child Chunking (Recommended Default)

The hierarchical parent-child strategy solves the precision/context trade-off directly:
- **Small child chunks** (100–200 tokens) are embedded and used for retrieval — their small size makes them precise
- When a child chunk is retrieved, the **full parent chunk** (400–800 tokens) is injected into the context window

```java
@Component
public class HierarchicalChunkBuilder {

    public record Chunk(String chunkId, String parentId, String text,
                        float[] embedding, Map<String, Object> metadata) {}

    public List<Chunk> buildHierarchicalChunks(String document, String docId,
                                                int childSize, int parentSize,
                                                int overlap) {
        List<Chunk> chunks = new ArrayList<>();
        List<String> parents = TokenSplitter.split(document, parentSize, overlap);

        for (int pIdx = 0; pIdx < parents.size(); pIdx++) {
            String parentText = parents.get(pIdx);
            String parentId = docId + ":p" + pIdx;

            // Parent chunk — not retrieved directly, injected on child hit
            chunks.add(new Chunk(parentId, null, parentText, null,
                Map.of("type", "parent", "docId", docId)));

            List<String> children = TokenSplitter.split(parentText, childSize, 0);
            for (int cIdx = 0; cIdx < children.size(); cIdx++) {
                String childText = children.get(cIdx);
                String childId = parentId + ":c" + cIdx;
                chunks.add(new Chunk(childId, parentId, childText, null,
                    Map.of("type", "child", "docId", docId)));
            }
        }
        return chunks;
    }
}

@Component
public class ParentExpansionRetriever {

    private final VectorStore vectorStore;
    private final ChunkStore chunkStore;

    // Retrieve child chunks by embedding, then expand to parent chunks for injection
    public List<Chunk> retrieveWithParentExpansion(float[] queryEmbedding, int topK) {
        List<VectorHit> childHits = vectorStore.search(queryEmbedding,
            Map.of("type", "child"), topK);

        Set<String> seen = new HashSet<>();
        List<Chunk> parents = new ArrayList<>();
        for (VectorHit hit : childHits) {
            String parentId = (String) hit.metadata().get("parentId");
            if (seen.add(parentId)) {
                parents.add(chunkStore.get(parentId));
            }
        }
        return parents;
    }
}
```

> ⚠️ **Index freshness is a silent quality killer.** A RAG system whose index has not been updated since the underlying documents changed will confidently retrieve stale information. Treat the indexing pipeline as a production data pipeline with an SLA: monitor index lag, alert when it exceeds the acceptable staleness threshold, and design incremental update pipelines rather than relying on periodic full re-indexing.

### Design Rules

| Rule ID | Rule |
|---|---|
| `RAG-CHUNK-001` | Metadata filters MUST be applied at the vector store query level — post-retrieval filtering wastes latency and silently reduces result counts below the requested `topK`. |
| `RAG-CHUNK-002` | Every chunk MUST carry metadata: `docType`, `department`, `date`, `accessClassification`, and `language` — chunks without filtering metadata cannot support access-controlled retrieval. |
| `RAG-CHUNK-003` | Hierarchical parent-child chunking MUST be used as the default for mixed-length document corpora — fixed-size chunking is only acceptable for homogeneous short-document corpora. |
| `RAG-CHUNK-004` | Index freshness SLA MUST be defined and monitored — `indexLagSeconds` MUST emit as a named metric; alert threshold MUST be configured before production deployment. |
| `RAG-CHUNK-005` | Chunk size parameters (childSize, parentSize, overlap) MUST be tuned with offline retrieval evaluation on production queries — default values are a starting point, not a production setting. |

---

## Section 35.4 — Retrieval Strategies: Dense, Sparse, and Hybrid

### Dense Retrieval

Dense retrieval embeds both queries and documents into a shared vector space and retrieves by approximate nearest-neighbour (ANN) search. It excels at semantic similarity — finding documents that mean the same thing even when they use different words.

**Dense retrieval's weakness is lexical gap:** it can miss documents that share specific technical terms, product names, or identifiers with the query but are not semantically close in embedding space.

### Sparse Retrieval (BM25)

BM25 (Best Match 25) is a probabilistic ranking function scoring documents by term frequency and inverse document frequency, with saturation adjustments for long documents. It excels precisely where dense retrieval struggles — exact term matching for product codes, identifiers, proper nouns, and technical terminology.

**BM25's weakness:** it fails where dense retrieval excels — paraphrase, synonym, and concept-level similarity.

### Hybrid Retrieval with Reciprocal Rank Fusion

Hybrid retrieval runs both dense and sparse retrieval **in parallel** and merges the result lists using Reciprocal Rank Fusion (RRF). For document \(d\) appearing in ranked lists from \(n\) rankers:

\[\text{RRF}(d) = \sum_{i=1}^{n} \frac{1}{k + r_i(d)}\]

where \(r_i(d)\) is the rank of document \(d\) in list \(i\), and \(k = 60\) is the smoothing constant from the original RRF paper (Cormack, Clarke, and Buettcher, 2009). Documents absent from a list contribute zero.

```java
@Component
public class HybridRetriever {

    private final VectorStore vectorStore;
    private final Bm25Index bm25Index;

    public List<RetrievalHit> hybridRetrieve(String query, float[] queryEmbedding,
                                              int topK, int rrfK) {
        // Fan-out: dense and sparse in parallel (chap34 ADV-FAN pattern)
        CompletableFuture<List<VectorHit>> denseFuture =
            CompletableFuture.supplyAsync(() -> vectorStore.search(queryEmbedding, topK));
        CompletableFuture<List<BM25Hit>> sparseFuture =
            CompletableFuture.supplyAsync(() -> bm25Index.search(query, topK));

        List<VectorHit> denseHits = denseFuture.join();
        List<BM25Hit> sparseHits = sparseFuture.join();

        // RRF score fusion
        Map<String, Double> scores = new HashMap<>();
        Map<String, RetrievalHit> docs = new HashMap<>();

        for (int rank = 1; rank <= denseHits.size(); rank++) {
            VectorHit h = denseHits.get(rank - 1);
            scores.merge(h.id(), 1.0 / (rrfK + rank), Double::sum);
            docs.put(h.id(), RetrievalHit.from(h));
        }
        for (int rank = 1; rank <= sparseHits.size(); rank++) {
            BM25Hit h = sparseHits.get(rank - 1);
            scores.merge(h.id(), 1.0 / (rrfK + rank), Double::sum);
            docs.put(h.id(), RetrievalHit.from(h));
        }

        return scores.entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .limit(topK)
            .map(e -> docs.get(e.getKey()))
            .toList();
    }
}
```

> 💡 **Hybrid retrieval consistently outperforms either dense or sparse retrieval alone across diverse enterprise corpora.** The RRF constant `k=60` is a robust default supported by empirical research — tune it only if offline evaluation on your query set shows a clear improvement from a different value. The latency overhead of running both indexes in parallel is negligible compared to the quality gain.

### Design Rules

| Rule ID | Rule |
|---|---|
| `RAG-RET-001` | Hybrid retrieval (dense + BM25 + RRF) MUST be used as the production default for enterprise RAG systems — pure dense or pure sparse retrieval requires offline evaluation evidence to justify. |
| `RAG-RET-002` | Dense and sparse retrieval MUST run in parallel using the fan-out pattern from chap34 — sequential retrieval in a hybrid pipeline is a latency defect. |
| `RAG-RET-003` | RRF `k=60` MUST be used as the default constant — deviation requires offline evaluation evidence. |
| `RAG-RET-004` | Similarity metric (cosine / dot product / Euclidean) MUST match the embedding model's training objective — metric mismatches degrade retrieval quality silently without raising errors. |

---

## Section 35.5 — Reranking

### First-Stage Retrieval vs. Precision Ranking

First-stage retrieval — whether dense, sparse, or hybrid — is optimised for **recall**: returning a candidate set that contains the relevant documents. It is not optimised for **precision**: ordering those candidates so that the most relevant one is ranked first.

The scoring functions used in first-stage retrieval (cosine similarity, BM25) score documents independently of each other and of the full query context. This independence is their limitation — they cannot model the interaction between the query and the full document content.

Reranking takes the first-stage candidate set (typically 20–50 documents) and scores each one against the query using a **cross-encoder** — a more expensive but more precise model that considers the full query and document jointly.

### Cross-Encoder Reranking

A cross-encoder takes a `(query, document)` pair as input and produces a single relevance score. Unlike bi-encoders which embed query and document independently, the cross-encoder sees both simultaneously, allowing it to model fine-grained relevance.

```java
@Component
public class CrossEncoderReranker {

    // cross-encoder/ms-marco-MiniLM-L-6-v2 — ~30-80ms on CPU for batch of 20 pairs
    private final CrossEncoderModel crossEncoder;

    public List<RetrievalHit> rerank(String query, List<RetrievalHit> candidates, int topK) {
        // Batch all pairs in a single forward pass — do NOT score sequentially
        List<String[]> pairs = candidates.stream()
            .map(c -> new String[]{query, c.text()})
            .toList();

        float[] scores = crossEncoder.predictBatch(pairs);

        List<ScoredHit> scored = new ArrayList<>();
        for (int i = 0; i < candidates.size(); i++) {
            scored.add(new ScoredHit(scores[i], candidates.get(i)));
        }

        return scored.stream()
            .sorted(Comparator.comparingDouble(ScoredHit::score).reversed())
            .limit(topK)
            .map(ScoredHit::hit)
            .toList();
    }
}
```

> 📝 **Cross-encoder inference must use batch processing.** For a candidate set of 20 documents, the MiniLM-L6-v2 cross-encoder runs in approximately 30–80 ms on CPU when all pairs are batched in a single forward pass. Per-pair sequential inference would be 100–300 ms. For larger candidate sets or stricter latency requirements, run the cross-encoder on GPU or use a lighter distilled model.

### LLM-Based Reranking (RankGPT)

An alternative is to prompt the reasoning LLM itself with the query and a numbered list of passages, asking it to output a ranked ordering. This produces high-quality rankings at the cost of an additional LLM inference call.

| Approach | Latency | Quality | Use When |
|---|---|---|---|
| **Cross-encoder** | 30–80 ms (CPU batch) | High | High-throughput pipelines; standard enterprise RAG |
| **LLM reranking (RankGPT)** | 300–800 ms | Very high | High-value, latency-tolerant tasks: regulatory analysis, complex research |

> ⚠️ **Always evaluate reranking quality with an offline benchmark before selecting a model.** Benchmark scores on MS MARCO do not reliably predict quality on enterprise-specific corpora. Domain characteristics matter more than leaderboard position.

### Design Rules

| Rule ID | Rule |
|---|---|
| `RAG-RANK-001` | Reranking MUST be applied to the first-stage retrieval output (20–50 candidates) before context assembly — injecting unranked first-stage results into the context window is prohibited. |
| `RAG-RANK-002` | Cross-encoder inference MUST batch all candidate pairs in a single forward pass — sequential per-pair inference is a latency defect. |
| `RAG-RANK-003` | LLM-based reranking MUST NOT be used in high-throughput agent pipelines — it is reserved for quality-critical, latency-tolerant tasks. |
| `RAG-RANK-004` | The reranking model MUST be evaluated on production-representative queries before deployment — MS MARCO benchmark scores are not a sufficient selection criterion. |

---

## Section 35.6 — Context Assembly and Prompt Injection

### From Retrieved Passages to Injected Context

Reranking produces an ordered list of relevant passages. Context assembly turns that list into the exact text injected into the agent's prompt. This contains several non-trivial decisions: how to format passages for LLM comprehension, how to handle the token budget, and how to structure attribution so the agent can cite sources.

### Token Budget Management

The context window has a finite token budget shared between the system prompt, conversation history, retrieved passages, and the space for the model's response. Context assembly enforces a **retrieval budget** — a maximum token allocation for all injected passages combined.

```java
@Component
public class ContextAssembler {

    private final Tokenizer tokenizer;

    public String assembleContext(List<RetrievalHit> passages, int tokenBudget) {
        StringBuilder assembled = new StringBuilder();
        int tokensUsed = 0;

        for (int i = 1; i <= passages.size(); i++) {
            RetrievalHit passage = passages.get(i - 1);
            String block = String.format("[Source %d] %s\n%s\n\n",
                i, passage.title(), passage.text());

            int blockTokens = tokenizer.countTokens(block);
            if (tokensUsed + blockTokens > tokenBudget) {
                // Try truncating the passage to fit
                String truncated = tokenizer.truncate(passage.text(),
                    tokenBudget - tokensUsed - tokenizer.countTokens(
                        String.format("[Source %d] %s\n\n", i, passage.title())));
                if (truncated.length() > 100) {  // Only inject if meaningful content remains
                    assembled.append(String.format("[Source %d] %s\n%s\n\n",
                        i, passage.title(), truncated));
                }
                break;
            }

            assembled.append(block);
            tokensUsed += blockTokens;
        }

        return assembled.toString();
    }
}
```

### Attribution Metadata Injection

Every passage injected into the context MUST carry a source number that maps back to the full source metadata. The source number enables the agent to produce reliable citations. The mapping from source number to `{sourceId, title, date, url}` is stored in the agent's session state (chap8) alongside the assembled context.

```java
public record AttributedContext(
    String formattedText,            // Formatted passages with [Source N] headers
    List<SourceMetadata> sources,    // Ordered list: sources.get(0) = [Source 1]
    int tokenCount                   // Total tokens used by assembled context
) {}
```

### "Lost in the Middle" Mitigation

Research (Liu et al., 2023) demonstrates that LLMs attend poorly to content placed in the middle of a long context — the most relevant passage should be placed **first** or **last** in the assembled context block, not buried in the middle. Rank-order injection (most relevant first) is the safe default.

### Design Rules

| Rule ID | Rule |
|---|---|
| `RAG-ASSEM-001` | Context assembly MUST enforce a token budget — injecting all reranked passages without budget management is prohibited. |
| `RAG-ASSEM-002` | Every passage block MUST carry an explicit `[Source N]` header with title — plain text injection without source numbering is prohibited. |
| `RAG-ASSEM-003` | Source number to full source metadata mapping MUST be stored in the agent's session state (chap8) — the agent MUST be able to produce full citations. |
| `RAG-ASSEM-004` | Passages MUST be injected in rank order (most relevant first) to mitigate the "lost in the middle" failure mode. |
| `RAG-ASSEM-005` | Visible delimiters (e.g., `---`, `[Source N]`) MUST separate passages — undifferentiated concatenated text degrades LLM comprehension quality. |

---

## Section 35.7 — Agentic RAG and Adaptive Retrieval

### Beyond Naive RAG

Naive RAG treats retrieval as a single pipeline: the agent calls once per information need — retrieve, inject context, generate. It works for simple fact lookups. It fails for complex research tasks where one retrieval pass does not provide sufficient grounding.

**Agentic RAG** makes retrieval an **iterative process** controlled by the agent itself. In 2026, agentic RAG is the dominant production pattern for enterprise knowledge work, research agents, and multi-hop question answering.

### Iterative Retrieval with Sufficiency Checking

The core loop: (1) retrieve, (2) evaluate whether the results answer the question, (3) if sufficient → generate; if insufficient → reformulate and retrieve again.

```json
{
  "name": "rag.assessSufficiency",
  "description": "Evaluate whether retrieved passages provide enough information to answer the original question. Use this after knowledge.retrieve to decide whether to generate a final answer or retrieve more context.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "question": { "type": "string" },
      "passages": { "type": "array", "items": { "type": "string" } }
    },
    "required": ["question", "passages"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "sufficient": { "type": "boolean" },
      "confidence": { "type": "number" },
      "gap": { "type": "string" }
    }
  }
}
```

The agent's reasoning loop after retrieval:
1. Call `knowledge.retrieve(question)`
2. Call `rag.assessSufficiency(question, passages)`
3. If `sufficient = true` → generate the final answer
4. If `sufficient = false` → use `gap` explanation to reformulate the query and retrieve again

### Corrective RAG (CRAG)

CRAG extends the sufficiency check with **fallback strategies**. When the retrieved passages are irrelevant, outdated, or contradictory, the agent detects this and routes to an alternative source — web search, a different knowledge base, or human escalation.

Required additional tools:
- `rag.diagnoseIssues(passages)` — classifies retrieval failures: `irrelevant`, `contradictory`, `insufficientDepth`
- `web.search` or `kb.alternative` — fallback retrieval from external or secondary sources

The agent decides the fallback strategy based on the diagnosis:
- Irrelevant passages → web search
- Contradictory passages → most-recent source
- Insufficient depth → query expansion

### Self-RAG

Self-RAG makes retrieval decisions **during generation** rather than in a separate pre-retrieval step. The LLM emits special retrieval tokens during generation that trigger additional retrieval mid-response:

1. Generate response with retrieval tokens enabled
2. If a retrieval token is emitted → interrupt generation, retrieve, inject new passages, and resume
3. Repeat until the response is complete

> 📝 **Self-RAG requires an agent runtime that supports interruptible generation and mid-stream context injection.** The patterns for implementing this are covered in Volume 2, Chapter 4 (Agent Runtime Architecture). Naive RAG and iterative RAG do not require these runtime capabilities and are more broadly applicable.

| Pattern | Complexity | Latency | Quality | Use When |
|---|---|---|---|---|
| **Naive RAG** | Low | Lowest | Baseline | Simple fact lookups, FAQ |
| **Iterative + sufficiency check** | Medium | Medium | High | Most enterprise knowledge work |
| **CRAG** | Medium-High | Medium | High | Mixed-quality corpora; multiple knowledge sources |
| **Self-RAG** | High | Highest | Highest | Complex research; multi-hop reasoning |

### Design Rules

| Rule ID | Rule |
|---|---|
| `RAG-AGENT-001` | Iterative retrieval with sufficiency checking MUST be implemented as the minimum viable agentic RAG pattern — naive one-shot RAG is not acceptable for complex enterprise queries. |
| `RAG-AGENT-002` | The `rag.assessSufficiency` tool MUST return a structured `gap` explanation — a boolean `sufficient` flag without a gap description prevents effective query reformulation. |
| `RAG-AGENT-003` | Corrective RAG fallback strategies MUST be pre-defined per retrieval failure type (`irrelevant`, `contradictory`, `insufficientDepth`) — improvised fallback at runtime is a reliability defect. |
| `RAG-AGENT-004` | Self-RAG MUST only be implemented on agent runtimes that support verified interruptible generation — partial implementations that lose generation state are worse than naive RAG. |
| `RAG-AGENT-005` | Maximum retrieval iterations MUST be bounded (default: 3) — unbounded iterative retrieval is a latency and cost vector. |

---

## Section 35.8 — Embedding Model Selection

### Why Embedding Model Matters

The wrong embedding model for your corpus and query distribution will degrade end-to-end RAG quality regardless of how well the pipeline is engineered downstream. Embedding model selection requires balancing four factors: domain alignment, dimensionality, multilingual support, and latency/storage trade-offs.

### Domain Alignment Guide

| Model | Domain Strength | Notes |
|---|---|---|
| `text-embedding-3-large` (OpenAI) | General-purpose, technical, financial | 3072-dim; highest precision; higher latency/cost |
| `text-embedding-3-small` (OpenAI) | General-purpose, high-throughput | 1536-dim; fast; good for most enterprise workloads |
| `bge-large-en-v1.5` | Balanced, widely benchmarked | 1024-dim; open weights; strong baseline |
| `jina-embeddings-v2-base-en` | Code and technical documentation | Optimised for developer knowledge bases |
| `legal-bert-base-uncased` | Legal corpora, compliance policy | Fine-tuned on legal domain; strong for policy RAG |
| `cohere-embed-multilingual-v3.0` | Multilingual enterprise documents | Best cross-language retrieval for global deployments |
| `multilingual-e5-large` | Mixed-language corpora | Open weights; good zero-shot cross-language performance |
| Custom fine-tune | Your specific corpus | Fine-tune a base model (e.g., `nomic-embed-text-v1.5`) on your domain using contrastive loss |

### Dimensionality Trade-offs

| Dimension | Precision | Storage | ANN Query Latency |
|---|---|---|---|
| 256–512 | Lower | Low | Very fast |
| 768–1024 | Balanced | Medium | Fast |
| 1536 | High | Medium-High | Moderate |
| 3072 | Highest | High | Slower |

### Late-Interaction Models (ColBERT)

ColBERT stores per-token embeddings for documents and performs fine-grained token-level interaction at query time. It offers significantly higher recall than standard bi-encoders at the cost of ~10x storage and query latency. With 2025–2026 hardware improvements, increasingly viable for high-precision enterprise RAG deployments.

> 💡 **Always evaluate embedding models on your specific corpus and query set before production deployment.** Use the RAGAS evaluation framework from Section 35.9 with 100–200 real production queries. General benchmark rankings (MTEB) are a starting point but do not predict quality on enterprise-specific documents. Domain alignment matters more than leaderboard position.

### Design Rules

| Rule ID | Rule |
|---|---|
| `RAG-EMBED-001` | Embedding model selection MUST be validated with offline RAGAS evaluation on production-representative queries — MTEB leaderboard scores are not a sufficient selection criterion. |
| `RAG-EMBED-002` | Multilingual enterprise deployments MUST use a multilingual-capable embedding model — monolingual embedders silently degrade recall on non-English content. |
| `RAG-EMBED-003` | The similarity metric (cosine / dot product / Euclidean) MUST be locked in at index creation time and MUST match the embedding model's training objective — changing the metric after indexing requires full re-indexing. |

---

## Section 35.9 — Multi-Agent Retrieval Patterns

### Why Multi-Agent Retrieval

A single RAG pipeline retrieving from a single knowledge index does not satisfy enterprise use cases where knowledge is distributed across heterogeneous sources: a vector-indexed document store, a SQL database, a live web search API, a proprietary knowledge graph, and a structured product catalog.

Multi-agent retrieval assigns **specialized retrieval agents** to each source. A Retrieval Orchestrator fans out retrieval tasks to Source Agents in parallel, collects their results, and assembles a unified context set.

### The Retrieval Orchestrator Pattern

```java
@Component
public class RetrievalOrchestrator {

    private final Map<String, RetrievalAgent> agentPool;
    private final FanOutDispatcher fanOut;         // chap34 ADV-FAN-*
    private final ContextAssembler contextAssembler;
    private final DuplicationDetector deduplicator;
    private final ConflictDetector conflictDetector;

    public AttributedContext orchestrate(String query, List<SubQuery> sourceQueries,
                                          int topK) {
        // Fan out to specialized source agents in parallel
        List<FanOutCall> calls = sourceQueries.stream()
            .filter(sq -> agentPool.containsKey(sq.agentName()))
            .map(sq -> new FanOutCall(sq.agentName(),
                Map.of("query", sq.query(), "topK", topK)))
            .toList();

        List<FanOutResult> results = fanOut.fanOut(calls, false,
            Duration.ofMillis(2500));   // Global deadline within tool timeoutMs

        // Flatten, skip failed retrievals
        List<RetrievalHit> allPassages = results.stream()
            .filter(FanOutResult::succeeded)
            .flatMap(r -> ((List<RetrievalHit>) r.result()).stream())
            .toList();

        // Deduplication — MinHash or exact hash on normalized text
        List<RetrievalHit> deduplicated = deduplicator.deduplicate(allPassages, 0.85);

        // Conflict annotation — mandatory before context assembly
        List<RetrievalHit> annotated = conflictDetector.detectAndAnnotate(deduplicated, 0.85);

        return contextAssembler.assembleContext(annotated, /* tokenBudget= */ 4096);
    }
}
```

### Deduplication

The same content may be indexed in multiple sources. Deduplication uses text similarity (MinHash or exact hash on normalized text) to detect and remove near-duplicate passages before context assembly. Injecting the same passage twice wastes token budget.

### Conflict Detection and Annotation

Two sources may contain contradictory versions of the same fact. Rather than silently injecting both passages, the system annotates conflicting passages with a structured conflict marker:

```
[Source 3] — CONFLICTS WITH Source 5 — EMEA Refund Policy (updated 2025-11-01)
Digital goods are refundable within 14 days of purchase.

[Source 5] — CONFLICTS WITH Source 3 — EMEA Refund Policy (legacy 2024-03-15)
Digital goods are non-refundable after purchase.
```

The agent receives both versions and can reason about the discrepancy or escalate to a human (chap12 HITL gate).

### Trust Boundary Enforcement

In enterprise deployments, different retrieval agents operate in different security zones — an HR agent can retrieve from the HR knowledge base but not from Finance. The retrieval orchestrator enforces these trust boundaries by reusing the tool layer's authorization model from chap32: each source agent is a registered tool with a declared data residency and access classification. The fan-out only dispatches to agents the calling session is authorized to use.

### Design Rules

| Rule ID | Rule |
|---|---|
| `RAG-MULTI-001` | The retrieval orchestrator MUST use the fan-out pattern (chap34 ADV-FAN-*) to dispatch to source agents in parallel — sequential dispatch across sources is a latency defect. |
| `RAG-MULTI-002` | Deduplication MUST be applied before context assembly — injecting duplicate passages wastes token budget and inflates context window cost. |
| `RAG-MULTI-003` | Conflicting passages MUST be annotated with explicit conflict markers before injection — silent injection of contradictory content without annotation is a reliability defect. |
| `RAG-MULTI-004` | Trust boundary enforcement for source agents MUST reuse the tool layer's authorization model (chap32) — separate authorization logic for retrieval agents is a security duplication defect. |
| `RAG-MULTI-005` | Every passage injected by a multi-source orchestrator MUST carry a `sourceId` tracing back to the originating agent and upstream knowledge store — for citation quality and audit logging. |

---

## Section 35.10 — RAG Quality Measurement

### Why RAG Quality Is Hard to Measure

RAG quality cannot be assessed by retrieval metrics alone (recall@K, MRR) or generation metrics alone (BLEU, ROUGE). The system's job is to produce a grounded, accurate, attributable response. That requires evaluating the full pipeline as an integrated system.

### The Four Quality Dimensions

| Dimension | What It Measures | Acceptable Range |
|---|---|---|
| **Context Recall** | What fraction of the information needed to answer the question was present in the retrieved passages? | ≥ 0.80 for high-stakes retrieval |
| **Context Precision** | What fraction of the retrieved passages were actually relevant? | ≥ 0.70 for most workloads |
| **Answer Faithfulness** | Does the generated answer stay within the bounds of the retrieved context, without hallucinating? | ≥ 0.85 |
| **Answer Relevance** | Does the generated answer actually address the question asked? | ≥ 0.80 |

### RAGAS Evaluation Framework

RAGAS (Retrieval-Augmented Generation Assessment) operationalises these four dimensions using LLM-as-judge scoring. It requires a test dataset of `{question, ground_truth_answer, retrieved_context, generated_answer}` tuples.

```java
@Component
public class RagasEvaluator {

    private final LlmClient judgeLlm;

    // Evaluate all four dimensions on a test dataset
    public RagasReport evaluate(List<RagasExample> testDataset) {
        double contextRecall = evaluateContextRecall(testDataset);
        double contextPrecision = evaluateContextPrecision(testDataset);
        double faithfulness = evaluateFaithfulness(testDataset);
        double answerRelevance = evaluateAnswerRelevance(testDataset);

        return new RagasReport(contextRecall, contextPrecision,
                               faithfulness, answerRelevance);
    }

    // Example output: {contextRecall: 0.88, contextPrecision: 0.74,
    //                   faithfulness: 0.91, answerRelevancy: 0.87}
}
```

### Connecting Metrics to Pipeline Stages

| Metric | Low Score → Root Cause |
|---|---|
| **Context Recall ↓** | Retrieval missing relevant documents; index stale; chunking too granular; query transformation ineffective |
| **Context Precision ↓** | First-stage retrieval too broad; reranker underperforming; chunk boundaries wrong |
| **Faithfulness ↓** | LLM hallucinating beyond retrieved context; retrieved passages contradictory or ambiguous; context window too small |
| **Answer Relevance ↓** | Wrong documents retrieved for query type; query decomposition losing key sub-questions; passages poorly formatted in prompt |

> 📝 **Build your RAGAS evaluation dataset from real production queries, not synthetic ones.** Sample 100–200 queries from production logs, annotate ground truth answers (or use a stronger LLM reviewed by a domain expert), and run this evaluation suite on every significant pipeline change. Synthetic benchmarks measure what you designed for; production-query benchmarks measure what your users actually need.

### Design Rules

| Rule ID | Rule |
|---|---|
| `RAG-QUAL-001` | All four RAGAS dimensions (context recall, context precision, faithfulness, answer relevance) MUST be measured before production deployment — partial evaluation is not acceptable. |
| `RAG-QUAL-002` | RAGAS evaluation MUST use real production-representative queries — synthetic benchmark queries are not a substitute for production-query evaluation. |
| `RAG-QUAL-003` | Metric thresholds MUST be declared in pipeline configuration — undeclared thresholds produce no deployment gate. |
| `RAG-QUAL-004` | RAGAS evaluation MUST be re-run on every significant pipeline change (new embedding model, changed chunk size, reranker swap, new knowledge source) — RAG systems that are not continuously evaluated degrade silently. |
| `RAG-QUAL-005` | When a metric drops below threshold, the diagnostic mapping (Section 35.10 table) MUST be applied to identify the failing stage before any changes are made — unguided changes to a multi-stage pipeline are as likely to degrade other dimensions as to improve the target metric. |

---

## Section 35.11 — Design Rules Master Table

| Rule ID | Scope | Rule Summary |
|---|---|---|
| `RAG-PIPE-001` | Pipeline | RAG pipeline behind single registered tool contract |
| `RAG-PIPE-002` | Pipeline | Every passage carries `sourceId`, `title`, `score`, `date` |
| `RAG-PIPE-003` | Pipeline | Tool declared `sideEffects: read-only` |
| `RAG-PIPE-004` | Pipeline | `timeoutMs ≤ 3000`; longer pipelines use submit-poll |
| `RAG-QT-001` | Query transform | Lightweight fast model — not the reasoning LLM |
| `RAG-QT-002` | Query transform | Decomposition falls back to original query on parse error |
| `RAG-QT-003` | Query transform | HyDE requires offline validation before deployment |
| `RAG-QT-004` | Query transform | Sub-queries retrieved in parallel (chap34 fan-out) |
| `RAG-CHUNK-001` | Chunking | Metadata filters at vector store query level — not post-retrieval |
| `RAG-CHUNK-002` | Chunking | Every chunk carries: `docType`, `department`, `date`, `accessClassification`, `language` |
| `RAG-CHUNK-003` | Chunking | Hierarchical parent-child as default for mixed-length corpora |
| `RAG-CHUNK-004` | Chunking | Index freshness SLA: `indexLagSeconds` metric + alert |
| `RAG-CHUNK-005` | Chunking | Chunk size tuned with offline evaluation |
| `RAG-RET-001` | Retrieval | Hybrid (dense + BM25 + RRF) as production default |
| `RAG-RET-002` | Retrieval | Dense and sparse in parallel (chap34 fan-out) |
| `RAG-RET-003` | Retrieval | RRF `k=60` as default constant |
| `RAG-RET-004` | Retrieval | Similarity metric matches model training objective |
| `RAG-RANK-001` | Reranking | Reranking applied before context assembly |
| `RAG-RANK-002` | Reranking | Cross-encoder batches all pairs in single forward pass |
| `RAG-RANK-003` | Reranking | LLM reranking not used in high-throughput pipelines |
| `RAG-RANK-004` | Reranking | Reranker evaluated on production-representative queries |
| `RAG-ASSEM-001` | Assembly | Token budget enforced |
| `RAG-ASSEM-002` | Assembly | `[Source N]` + title header on every passage |
| `RAG-ASSEM-003` | Assembly | Source metadata mapping in session state (chap8) |
| `RAG-ASSEM-004` | Assembly | Rank-order injection (most relevant first) |
| `RAG-ASSEM-005` | Assembly | Visible delimiters between passages |
| `RAG-AGENT-001` | Agentic RAG | Sufficiency checking as minimum viable agentic pattern |
| `RAG-AGENT-002` | Agentic RAG | Structured `gap` explanation in sufficiency result |
| `RAG-AGENT-003` | Agentic RAG | CRAG fallbacks pre-defined per failure type |
| `RAG-AGENT-004` | Agentic RAG | Self-RAG only on verified interruptible-generation runtimes |
| `RAG-AGENT-005` | Agentic RAG | Maximum retrieval iterations: 3 |
| `RAG-EMBED-001` | Embedding | Model selection validated with RAGAS on production queries |
| `RAG-EMBED-002` | Embedding | Multilingual model for multilingual corpora |
| `RAG-EMBED-003` | Embedding | Similarity metric locked at index creation |
| `RAG-MULTI-001` | Multi-agent | Fan-out dispatch to source agents (chap34 ADV-FAN-*) |
| `RAG-MULTI-002` | Multi-agent | Deduplication before context assembly |
| `RAG-MULTI-003` | Multi-agent | Conflict annotation with explicit markers |
| `RAG-MULTI-004` | Multi-agent | Trust boundaries via chap32 authorization model |
| `RAG-MULTI-005` | Multi-agent | Per-passage `sourceId` tracing to originating agent |
| `RAG-QUAL-001` | Quality | All four RAGAS dimensions measured pre-deployment |
| `RAG-QUAL-002` | Quality | Evaluation dataset from real production queries |
| `RAG-QUAL-003` | Quality | Metric thresholds declared in configuration |
| `RAG-QUAL-004` | Quality | RAGAS re-run on every significant pipeline change |
| `RAG-QUAL-005` | Quality | Diagnostic mapping applied before making changes to failing pipeline |

---

## Section 35.12 — Part 7 Synthesis: The Complete Integration Stack

How the four chapters of Part 7 compose into a unified integration architecture:

| Chapter | Layer | What It Provides to chap35 |
|---|---|---|
| **chap32** (Tool Abstraction Layer) | Foundation | `knowledge.retrieve` is a registered tool contract with dispatch pipeline, authorization check, timeout, and middleware stack |
| **chap33** (Enterprise Systems Integration) | Backend adapters | Enterprise knowledge stores (SharePoint, SQL, object stores) are backend adapters behind chap32 tool contracts |
| **chap34** (Advanced Tool Patterns) | Composition layer | Fan-out (parallel dense + sparse retrieval), chaining (query transform → retrieve → rerank → assemble), middleware caching for frequent queries |
| **chap35** (RAG Pipelines) | Retrieval intelligence | Multi-stage pipeline that transforms queries, retrieves from hybrid indexes, reranks, assembles context, and supports adaptive agentic retrieval |

### Volume 1 → Volume 2 Implementation Handoff

| Volume 1 Component | Volume 2 Java Implementation |
|---|---|
| `knowledge.retrieve` tool contract | Java `RagToolHandler` implementing chap32 `ToolHandler` interface — Vol 2, Ch. 3 |
| `QueryRewriter`, `QueryDecomposer`, `HydeQueryBuilder` | Java components in `rag.transform` package — Vol 2, Ch. 3 |
| `HierarchicalChunkBuilder` + `ParentExpansionRetriever` | Java components in `rag.index` package — Vol 2, Ch. 3 |
| `HybridRetriever` + RRF | Java `HybridRetrievalService` with `VectorStore` + `Bm25Index` adapters — Vol 2, Ch. 3 |
| `CrossEncoderReranker` | Java `RerankerService` wrapping JNI/ONNX cross-encoder — Vol 2, Ch. 3 |
| `ContextAssembler` + `AttributedContext` | Java `ContextAssemblyService` with tokenizer integration — Vol 2, Ch. 3 |
| Agentic RAG loop (sufficiency check, CRAG) | LangGraph graph nodes: `retrieveNode` → `assessSufficiencyNode` → conditional edge — Vol 2, Ch. 4 |
| `RetrievalOrchestrator` (multi-agent) | LangGraph supervisor pattern over source-specialized subgraphs — Vol 2, Ch. 4 |
| `RagasEvaluator` | Java integration test harness + CI quality gate — Vol 2, Ch. 12 |

---

> **Volume 1 closes here.**
>
> Across seven parts and thirty-five chapters, Volume 1 has established the complete conceptual, architectural, and integration foundation for production AI agent systems: the agent loop and its failure modes, the memory subsystem across five types with a full write-store-retrieve-expire lifecycle, multi-agent coordination and protocol, nine frameworks evaluated on a ten-axis rubric, and a four-layer integration stack from tool contracts to production RAG pipelines.
>
> Every layer — the contract model, the registry, the dispatch pipeline, the middleware stack, the RAG pipeline — is specified precisely enough to be implemented. **Volume 2 performs that implementation.**
