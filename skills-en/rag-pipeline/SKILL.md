---
name: rag-pipeline
description: Use when building or tuning a RAG retrieval pipeline. Covers chunking strategies and parameters, dense embeddings (BGE-M3, cosine similarity, ANNOY/HNSW), sparse embeddings (TF-IDF to BM25), three hybrid stages (parallel recall, RRF fusion, cross-encoder reranking), recall@k/MRR/nDCG metrics, and when sparse or dense retrieval wins.
---

# RAG Retrieval Pipelines

Translated from the [Chinese original](../../skills/rag-pipeline/SKILL.md).

## When to Use

- Build knowledge-base retrieval from scratch: chunk → index → retrieve → inject for generation.
- Choose dense, sparse, or hybrid retrieval, and ANNOY versus HNSW ANN indexing.
- Diagnose inadequate retrieval and tune chunk size, fusion, or reranking.
- Establish retrieval metrics and acceptance criteria.
- Explain failed recall for particular queries: semantic similarity versus exact matching.
- Supply underlying retrieval for memory systems or knowledge organization.

## Core Principles

- RAG has three core steps: **retrieve relevant passages → inject into context → generate with an LLM grounded in context**. Retrieval sets the answer ceiling; generation determines how fully it is realized. Diagnose retrieval before changing the prompt.
- RAG exists because training data has a cutoff while knowledge bases can be updated anytime. External breadth and freshness compensate for model limitations.
- Two system components: a **retriever** finding relevant passages and a **generator**, usually an LLM generating from those passages as context. Most tuning targets the retriever.
- Chunking is essential for two reasons: embedding models have input limits, and one vector for a whole document mixes topics and dilutes semantics; retrieval should inject only relevant sections, while large chunks bring irrelevant content, waste window space, and dilute attention.
- Dense understands meaning and sparse understands wording, with different blind spots: sparse precisely matches “HTTP-403” where dense may return generic “server errors”; dense finds “cat” for “kitty” where sparse cannot. No single strategy is reliable everywhere; hybrid is the production default.
- Chunking inherently severs original context: “the company's revenue rose 3%” leaves which company and quarter unknown. Generate a contextual prefix for each chunk at indexing time (`book/chapter3.md`, “RAG Techniques: Contextual Retrieval”), or mitigate with reranking.
- The three hybrid stages are distinct and progressive: **parallel retrieval** (independent dense/sparse recall) → **result fusion** (unified candidate pool) → **neural reranking** (fine ranking). Neither fusion nor reranking replaces the other.
- Raw scores are not directly comparable: cosine similarity (0-1) and BM25 (0 to tens) have very different scales and distributions. Fusion must discard raw scores.
- Choose or combine routes by query type:

  | Query type | Dense | Sparse | Conclusion |
  | --- | --- | --- | --- |
  | Synonyms/paraphrases (kitty vs cat) | Wins | Loses | Prefer dense |
  | Exact codes/identifiers (HTTP-403) | May overgeneralize | Wins | Prefer sparse |
  | Technical terms, personal names | Average | Wins | Prefer sparse |
  | Multilingual/cross-language | Wins | Loses | Prefer dense |
  | Conceptual/descriptive questions | Wins | Average | Prefer dense |

  Hybrid retrieval + reranking is the production default covering all types.

## Practical Patterns

- Chunking choices:
  - **Fixed-size:** split by token count, such as 512, with 50-100 token overlap to avoid cutting critical sentences. Simple and predictable but ignores structure, potentially splitting paragraphs, code, or tables. Use only as a baseline.
  - **Recursive/structure-aware:** fall back through natural boundaries: section headings → paragraphs → sentences. Preferred for Markdown/HTML and the most common production default.
  - **Semantic:** compare adjacent sentence embeddings and split at semantic cliffs to keep one topic per chunk. Higher quality costs additional embedding computation.
  - Starting parameters: **256-1024 tokens per chunk, 10%-20% overlap**, then tune with measured quality.
  - No free lunch: too small means incomplete, contextless, ambiguous information; too large mixes topics, dilutes embeddings, and injects irrelevant content.
- Dense route: use context-aware models such as BGE-M3 (768+ dimensions; static Word2Vec-style vectors cannot handle polysemy, and early BERT-style 512-token limits do not suit long text). Cosine similarity compares vector **direction**, or meaning, rather than length, correctly matching content of differing document lengths.
- Embeddings support semantic arithmetic: “king” - “male” + “female” ≈ “queen.” Semantic relations are encoded as linearly computable operations. The same word has different vectors by context (“Apple Inc.” versus “two jin of apples”), moving from word-level to context-level representations.
- ANN index choices:
  - **ANNOY (trees):** fast builds and low memory, but no incremental updates: a full rebuild is required. Suits relatively static datasets.
  - **HNSW (graphs):** supports incremental insertion and very high query accuracy, but slower builds and higher memory. Periodic rebuilds after long incremental operation are recommended to maintain accuracy. Suits dynamic, real-time indexing.
  - Index strategy matters as much as embedding choice for performance, cost, and maintainability.
- Sparse route: TF-IDF values words frequent in the current document but rare across the corpus (IDF = ln(N/DF)); raw term frequency grows linearly and lacks document-length correction. BM25 adds **term-frequency saturation** through k1 and **length normalization** through b, commonly k1=1.5 and b=0.75. Its IDF becomes ln((N-DF+0.5)/(DF+0.5)); DF > N/2 makes it negative, so implementations need a floor. An inverted index maps term → documents, like a book's terminology index.
- Tokenization is a prerequisite for sparse quality: correctly split numbers (404, 3.14, 2.0.1), codes (XK9-2B4-7Q1, API_KEY_123), terms (C++, .NET, Node.js), mixed case (JavaScript, PyTorch), email addresses, hexadecimal (#FF5733, 0x1234), and abbreviations (API, HTTP), and remove stopwords.
- Fusion and reranking:
  - **RRF (Reciprocal Rank Fusion):** score = Σ 1/(k+rank), commonly k=60 to smooth leading-rank differences. Discards raw scores and uses ranks only: simple and robust, but loses score-based relevance signals.
  - **Neural reranking:** fine-rank roughly the top 50 fused candidates with a cross-encoder, concatenating query and document for token-level interaction. This is much more accurate than initial bi-encoder screening, which separately encodes query and document then compares vectors. A bi-encoder resembles a recruiter's rapid résumé screening; a cross-encoder resembles an in-depth interview. bge-reranker-v2-m3 is an option.
  - Bi-encoders versus cross-encoders embody a structural speed/accuracy tradeoff: screening massive collections must be fast; fine ranking can afford slower processing for a few candidates.
- Metrics and acceptance, all calculated on annotated test queries:
  - **recall@k:** this book actually uses hit rate: any relevant document in the first k results counts as a hit. The standard academic definition is the proportion of relevant documents retrieved; note this difference across sources. This is closest to RAG's needs: once relevant documents enter context, the LLM can use them.
  - **MRR:** average the reciprocal rank of each query's first relevant document. Rank 1 scores 1; rank 10 scores 0.1. Measures whether relevant information appears early enough.
  - **nDCG:** combines all relevant documents' ranks and relevance grades to measure overall ranking quality.
  - **Retrieval failure rate:** the fraction of queries for which correct information is absent from the top-20 results.
- Report metrics separately for dense, sparse, fusion, and reranking to locate the stage losing recall.

## Common Pitfalls

- Tuning generation prompts before measuring retrieval. Without adequate recall@k, later work lacks a foundation.
- Chunks too small (ambiguous without context) or too large (mixed topics and irrelevant injected content).
- Choosing RRF or reranking as alternatives. Reranking does not merely recover scores discarded by RRF; it uses a stronger matching paradigm and helps regardless of the fusion method.
- Deploying only dense or only sparse, then reacting to complementary failures on semantic queries versus exact codes/identifiers/names/multilingual queries.
- Directly summing weighted dense and BM25 scores despite different scales, or frequent incremental updates to ANNOY causing repeated full rebuilds.
- Failing to inspect severed context after chunking, causing serious semantic loss in isolated embeddings.
- Embedding an entire document as one vector: multiple topics prevent precise representation of any one topic.
- Optimizing recall without ranking: a relevant document ranked 50th is in the candidate pool but not the final top-k context.
- Never rebuilding indexes: updated knowledge does not reach the vector store, so retrieval returns stale content.

## Companion Code

- `chapter3/retrieval-pipeline/` — Full dense + sparse + RRF + BGE-Reranker pipeline. test_client.py compares routes by query challenge (semantic similarity such as kitty/feline, exact names, multilingual queries, technical codes) and shows ranking changes after reranking.
- `chapter3/dense-embedding/` — BGE-M3 retrieval service with interchangeable ANNOY/HNSW. Offline CLI computes recall@k/precision@k/MRR and compares ANN with brute-force exact search for recall, build time, and query latency.
- `chapter3/sparse-embedding/` — BM25 and inverted index from scratch, with tokenization covering numbers, codes, technical terms, and mixed case. Logs expose tokenization, inverted-index hits, TF/IDF calculations, and final ranking.

## Further Reading

- `book/chapter3.md`, “RAG Fundamentals: Building an Agent's Knowledge Acquisition Pipeline”
