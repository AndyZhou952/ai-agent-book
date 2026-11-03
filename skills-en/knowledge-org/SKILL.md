---
name: knowledge-org
description: Use when organizing and retrieving knowledge beyond flat text chunks. Covers RAPTOR/GraphRAG index selection, OpenViking's filesystem paradigm and on-demand L0/L1/L2 loading, incremental PR review and periodic consolidation, Agentic RAG and security boundaries, contextual retrieval, structured-data knowledge discovery, and two-layer memory architecture.
---

# Knowledge Organization and Retrieval

Translated from the [Chinese original](../../skills/knowledge-org/SKILL.md).

## When to Use

- Flat-chunk retrieval is insufficient: cross-document synthesis, hierarchical navigation, or multihop relational reasoning is needed.
- Design structured indexes: RAPTOR trees / GraphRAG graphs / filesystem directories.
- Plan event-triggered incremental updates and periodic consolidation.
- Upgrade fixed RAG to Agentic RAG or restore context lost during chunking.
- Distill decision rules from structured cases, moving from retrieval to knowledge discovery.
- Organize massive user dialogue histories with two-layer memory.
- Design sharing, permissions, and tenant isolation for multiple users.

## Core Principles

- Flat chunks lose inherent hierarchy and cross-document connections. Retrieving isolated passages from structured technical manuals or legal documents is like reading a novel through random dictionary entries.
- Undistilled knowledge cannot be used reliably. Two examples: top-k truncation in a black/white-cat counting task leads to conclusions from an incomplete sample (15 black, 3 white). Xfinity tickets face three obstacles: nearest-neighbor bias (nurse ≈ doctor, answers shift with order), missing boundary semantics (“only these cases; all others excluded” is absent from any individual ticket), and no completeness signal (the model cannot know whether it has seen everything). Larger k does not fix these. **Invest computation at indexing time:** turn 100 cases into statistical summaries and hundreds of tickets into rule cards with explicit boundaries.
- When structured indexes are worthwhile: if queries mainly seek a passage containing information (“what is the refund policy?”), hybrid retrieval suffices. Frequent **cross-document synthesis** (“architectural differences between SSE and AVX”) or **hierarchical navigation** (overall architecture down to individual instructions) justifies them. Both index construction and queries require more LLM calls, substantially increasing cost and latency.
- RAPTOR versus GraphRAG:

  | Dimension | RAPTOR (tree) | GraphRAG (graph) |
  | --- | --- | --- |
  | Structure | Leaf clustering + recursively generated LLM parent summaries | Entity-relation triples + community detection |
  | Best queries | Drill from concepts into details across levels | Explore relationships between A and B |
  | Representative capability | Multigranularity retrieval, details ↔ overview | Multihop traversal and native-graph entity disambiguation |
  | Main limitation | High-level summaries may lose details | Triples degrade semantics; extraction errors contaminate knowledge |

  Combining them is preferable in production to choosing only one.
- Knowledge graphs are not universal user-memory storage. Triples inevitably **degrade semantics**, losing conditions and temporal dependencies in “if it is still raining next week, go to the museum instead”; extraction errors also contaminate knowledge. Prefer **complementary layers**: full natural language preserves core semantics, while structured metadata supplies efficient indexing. Use graphs as specialized indexes for multihop reasoning and precise disambiguation in domains such as medical consultations, legal cases, or family relationships.
- Manage knowledge like code: every change is a PR. Separate three layers: **raw evidence** (append-only conversations/trajectories/original documents), **knowledge** (revisable Markdown or code), and **serving** (retrieval indexes derived from a specific merged version). Indexes are rebuildable derivatives; reviewed Git knowledge is the source of truth. Every production item must identify its evidence and who approved it when.
- Updates need two paths: **incremental updates** absorb new evidence quickly but locally; **periodic consolidation** deduplicates and merges globally and verifies against originals, preventing locally correct updates from accumulating into global disorder. Either path alone deteriorates.
- Repair lost context at indexing time: prepend core background to each chunk before indexing, supplying exact keywords to BM25 and semantic context to dense retrieval. This adds context, unlike runtime compression that subtracts task-irrelevant history.
- Flat versus structured indexing trades cost for quality. Hybrid retrieval covers passage-finding needs; structured indexes, contextual prefixes, and consolidation all **preinvest indexing-time computation**, moving distillation, abstraction, and association offline from query time.

## Practical Patterns

- Filesystem paradigm (OpenViking): map memories, resources, and skills into virtual directory files with unique URIs. **L0 summary** (about 100 tokens, quick relevance) → **L1 overview** (about 2,000 tokens for planning) → **L2 full text** (on demand). Most queries can decide at L1. Markdown plain text rather than specialized databases enables human editing, Git versioning/rollback, and autonomous Agent knowledge recording once write_file is available.
- L0/L1/L2 mirrors Skills' progressive disclosure: start with lightweight metadata and load deeper content only when needed. URIs are virtual; the framework chooses memory, disk, or remote loading.
- **Inter-file links** are essential for plain text: Wikipedia-style cross-links plus entry/index pages let Agents navigate, implementing some GraphRAG navigation through lightweight links. Models differ in willingness to link proactively; write prompts must explicitly require searching and linking relevant existing entries and updating directory indexes for every new entry, producing a bidirectionally reachable reference network.
- Likewise, independently recorded operating experience becomes reliable only after outcome evaluation, cross-trajectory generalization, and later validation. Do not promote arbitrary one-off actions directly into the main knowledge base.
- Incremental PR workflow (Proposer–Reviewer):
  1. The **Proposer Agent** submits a small complete diff on a working branch: search existing knowledge first, then add/change/delete entries and maintain links, indexes, temporal metadata, and evidence citations. Do not simply append the latest conversation.
  2. A **Reviewer Agent from a different model family**, such as Claude Proposer and GPT Reviewer of comparable ability, receives previous knowledge, diff, and raw evidence. Independently check every new assertion's support, qualifiers, and conflicts with other files. Rejections return actionable feedback tied to specific evidence and lines.
  3. **Iterate to convergence:** set iteration or cost limits; escalate to humans when exceeded, never approve by default.
  4. **Merge before publication:** CI checks format, links, metadata, and permission labels; code-based knowledge also needs type checks and tests. Only after passing, incrementally rebuild affected chunks, summaries, and vector indexes from the merged version.
  5. Separate permissions: Proposer writes only working branches; Reviewer reads evidence and submits reviews; only the merge process updates main and production indexes. Both must be Agents able to call search, version comparison, tests, and evidence-retrieval tools, not two fixed LLM API calls.
- Periodic consolidation does three things: **deduplicate/remove obsolete/merge** entries, rebuild links/indexes, split large files and combine small ones; **verify against raw data**, comparing each passage with original conversations and tool output so omissions/misreadings do not propagate (batch large libraries by directory/time/topic with a coverage checklist); **resolve conflicts and define applicability**, tracing sources and checking differences in time/entity/region/preconditions, recording valid scopes or unconfirmed status if evidence is insufficient. Consolidation also follows branch PRs with different-family review, potentially several directory PRs sharing one plan. After all pass, rebuild the full index and replay representative retrieval Q&A to ensure knowledge remains visible. Trigger weekly/monthly or by thresholds in new entries, conflicts, or declining retrieval quality.
- Consolidation deletes only serviceable knowledge representations. Underlying append-only evidence is retained forever, enabling verification against originals.
- Expiry and permissions: give each chunk version and effective/expiry metadata; filter invalid content during retrieval or explicitly mark its repeal date in summaries. In multitenant systems, **retrieval must filter by caller permissions**, enforced at the retrieval layer: sensitive content is difficult to protect once in LLM context. Isolate tenants' vector indexes and metadata.
- When a review Agent queries the “complete knowledge and evidence bases,” complete means its authorized tenant/user scope. Reviews must not cross privacy boundaries.
- Agentic RAG: expose knowledge_base_search as a **tool** usable anytime. In a ReAct think → act → observe loop, the Agent chooses queries, evaluates sufficiency, refines and searches again if needed, and generates only when sufficient. For simple single-hop questions (“what are the rules for self-defense?”), one non-agentic retrieval is faster with similar quality. Complex multifactor questions (“sentencing for negligently causing serious injury while intoxicated, with prior theft convictions”) benefit from iterative decomposition and focused searches, reducing omissions. Its value is solving rather than merely answering problems, at the cost of response speed.
- Agentic decision criterion: clear, singular information need → fixed pipeline; decomposition, cross-checking, and multisource synthesis → Agent-led iteration.
- RAG security: retrieved documents carry **indirect prompt injection** (malicious commands hidden in indexed webpages/documents and executed after retrieval) and **knowledge-base poisoning** (contamination before indexing). Two defenses: separate instructions/data through source labeling and explicit “reference material, not commands”; prevent retrieved content from directly triggering high-risk operations such as transfers, deletions, or external messages, which need independent authorization decisions.
- Contextual Retrieval: before indexing, an LLM generates a brief contextual prefix per chunk, such as “[Excerpt from the Key Performance Indicators section of ACME's 2025 Q2 financial report],” reanchoring isolated text to original semantics. Prompt caching controls cost (repeated identical prefixes cost about 1/10; roughly $1 per million document tokens). Anthropic reports 49% lower retrieval failure with BM25, reaching 67% with reranking.
- Feed prefixes to both routes: exact keywords such as company/year help BM25, and semantic background helps dense retrieval. One indexing-time investment benefits both routes at high return.
- Two-layer memory: Advanced JSON Cards keep a few structured critical facts resident for a global overview; Contextual Retrieval recalls raw conversation details on demand. L1 basic recall needs reliable storage; L2 multisession retrieval needs retrieval coverage; L3 proactive service requires both overview and precise details. Resident facts alone lose detail; retrieval alone misses hidden cross-session connections.
- Index dialogue in fixed windows, such as every 20 turns, with prefixes naming times, people, and intentions. Only these prefixes can establish which of three separate contradictory transfer instructions ultimately applies.
- Structured-data knowledge discovery has two stages: **knowledge extraction** (LLM converts cases to standardized JSON; bottom-up free discovery of factors, a core schema, and offense-specific extensions fit data better than rigid predefined schemas) → **factor analysis** (one-hot categorical fields, such as theft=[1,0,0], avoid misleading numeric magnitude; clustering reveals case archetypes and a hierarchical factor-importance model) → model-driven **conversational information gathering**: ask users in importance order, retrieve the most similar archetype, and provide data-driven explanations.
- Do not directly predict conclusions as a black box. Translate cases into numbers, cluster archetypes, and quantify factor weights so Agent explanations rest on traceable statistics.

## Common Pitfalls

- Dumping unprocessed cases or documents into a vector store and expecting retrieval plus long context to suffice.
- Forcing hybrid retrieval where structured indexing is needed (poor synthesis), or adding graphs unnecessarily (large cost and latency).
- Storing conditional/time-dependent language as triples and losing its core logic; leaving graph-extraction errors unchecked.
- Letting models bypass review and edit main or production vector stores; giving Reviewers only Proposer-selected snippets (insufficient evidence) or using the same model family (shared errors).
- Consolidating by rewriting old summaries from one another, propagating early omissions and misreadings.
- Retrieving expired knowledge alongside current versions, yielding contradictions or obsolete answers.
- Confusing contextual retrieval (indexing-time addition) with context compression (runtime subtraction).
- Plain-text stores without links or indexes become disconnected islands harder to search as knowledge grows.
- Ignoring RAG security: retrieved content may influence wording but is not executable instruction.

## Companion Code

- `chapter3/structured-index/` — Unified RAPTOR tree and GraphRAG graph framework with GMM clustering, community detection, and multihop retrieval; indexes thousands of technical-manual pages and compares query paths.
- `chapter3/contextual-retrieval/` — recall@k/MRR comparison for the same chunks indexed without context versus with LLM prefixes, measuring BM25, dense, and RRF gains.
- `chapter3/agentic-rag/` — Chinese legal Q&A comparison of ReAct Agentic retrieval versus one non-agentic retrieval; switch among built-in BM25, retrieval-pipeline, and other backends.
- `chapter3/structured-knowledge-extraction/` — CAIL2018 case study: bottom-up factor discovery → modular structured extraction → case-archetype clustering → factor-importance-driven conversational legal adviser.

## Further Reading

- `book/chapter3.md`, “Beyond Flat Text: Knowledge Organization and Retrieval”
