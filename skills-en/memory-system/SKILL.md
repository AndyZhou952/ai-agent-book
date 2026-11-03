---
name: memory-system
description: Use when designing, implementing, or evaluating Agent user-memory systems. Covers three evaluation levels, separating trajectories from long-term memory, choosing Simple Notes through Advanced JSON Cards or executable code, Mem0/Memobase tradeoffs, memory compression and consolidation, privacy redaction, and deciding between memory and RAG.
---

# User Memory Systems

Translated from the [Chinese original](../../skills/memory-system/SKILL.md).

## When to Use

- Design cross-session user memory for preferences, identity, and past interactions.
- Choose a memory format: Simple Notes / Enhanced Notes / JSON Cards / Advanced JSON Cards / executable code.
- Evaluate memory quality against the three-level framework or LoCoMo benchmark.
- Integrate, adapt, or compare memory frameworks such as Mem0 and Memobase.
- Handle conflicts, expiry, growth, compression, and privacy redaction.
- Decide whether information belongs in user memory or a shared knowledge base.
- Design memory compression and periodic consolidation.

## Core Principles

- The memory/RAG boundary: **one user's personal information** (preferences, identity, relationships, history) belongs in user memory; **collective knowledge shared by all users** (industry regulations, company procedures, professional documents) belongs in a RAG knowledge base. Their scales differ, but underlying techniques such as vector retrieval and knowledge compression overlap, as do conflicts, expiry, and inaccurate retrieval. RAG can also improve memory recall.
- Memory is not sentence-by-sentence archiving. After a session, use a dedicated LLM call to extract information satisfying all three rules: **selectivity** (discard short-lived details such as “search returned 3 options”), **abstraction** (generalize this session's window-seat request into a lasting preference), and **structure** (store facts in retrievable fields).
- Establish evaluation before design. Eight capabilities are personal-information retention, preference tracking, context switching, memory updates, multisession continuity, complex reasoning, temporal awareness, and conflict resolution. Use three engineering acceptance levels:
  - **L1 basic recall:** accurately store and retrieve directly supplied structured facts (“my membership number is 12345”).
  - **L2 multisession retrieval:** find all relevant information across entities and periods, and proactively clarify (“which car should I book for?” when the user owns two, rather than guessing).
  - **L3 proactive service:** combine old memories to anticipate needs, such as connecting an international flight booking to passport information stored months ago and warning of impending expiry.
  - **LoCoMo** is a public reference benchmark of very long conversations averaging roughly 300 turns and up to 35 sessions.
- Decide three orthogonal dimensions separately: **where** (trajectory / user long-term memory / business state), **how** (four formats), and **what** (episodic / semantic / procedural memory). They can be freely combined.
- A trajectory is an append-only raw record of one run for tracing, debugging, and auditing; runtime context can compress or reorganize it. User long-term memory is a cross-session persistent profile tied to user ID, repeatedly rewritten, merged, and retired. One is a transaction log; the other is a maintained file.
- Cognitive types guide what to store: episodic memory records events (which flight was booked); semantic memory stores distilled stable traits (the user is vegetarian); procedural memory learns workflows (search nonstop flights → confirm seat → use frequent-flyer number).
- Choose format by engineering needs, balancing simplicity and expressiveness; choose type by business needs for facts, events, or procedures. These are orthogonal decisions.
- Choose format by criticality and volume:
  - **Simple Notes:** minimal atomic facts, lowest overhead and O(1) operations, but lost relationships require reconstructing fragments for combined queries.
  - **Enhanced Notes:** complete contextual paragraphs with rich semantics, but redundant storage and updates requiring multiple paragraph rewrites.
  - **JSON Cards:** three nested levels, category → subcategory → key/value; predictable, extensible, and partially updatable, but single-category classification loses multiple dimensions. “Developing personal projects with Python on weekends” is simultaneously a time, technology, and activity preference.
  - **Advanced JSON Cards:** add backstory (why stored), person and relationship (for whom), and timestamps to facts; disambiguate identically named entities such as the user's dentist Dr. Zhang versus the father's cardiologist Dr. Zhang.
  - Rule of thumb: use Advanced JSON Cards for **small amounts of critical information**, such as preferences and important relationships; use Simple Notes for **large amounts of noncritical** conversational facts. Production systems usually mix paths by category.

  Four-format reference:

  | Format | Structure | Strength | Cost |
  | --- | --- | --- | --- |
  | Simple Notes | Minimal atomic facts | Very low overhead, O(1) operations | Lost relationships; combined queries reconstruct fragments |
  | Enhanced Notes | Complete contextual paragraphs | Narrative structure, rich semantics | Redundancy; attribute changes rewrite multiple paragraphs |
  | JSON Cards | Category → subcategory → key/value | Partial updates, predictable, extensible | Rigid classification loses multiple dimensions |
  | Advanced JSON Cards | Facts + backstory/person/relationship/timestamps | Name disambiguation and identity separation | High generation and maintenance cost |

- For deterministic reasoning such as aggregation, conflict detection, or constraint enforcement, upgrade memory to **executable code** (User as Code). Facts first enter an append-only log, from which typed state is periodically rebuilt; ordinary functions express rules, such as warning when a passport has less than 180 days' validity before an international departure. Representation and reasoning then share a verifiable medium.
- Bigger memory is not necessarily better. Three compression layers: filter by importance (access frequency, time decay, emotional intensity, uniqueness; mark below-threshold items compressible/deletable) → cluster similar memories into representative summaries (“the user often asks about weather, especially rainfall,” archiving details in secondary storage) → abstract episodic memories into semantic/procedural memory (“prefers value-for-money products and cares about user reviews” from repeated shopping conversations).
- Business state (“needs clarification” / “processing request” / “awaiting payment”) is a developer-defined abstraction of high-level task stages, parallel to trajectory and long-term memory and especially useful in event-driven architectures. Most systems can start with trajectory + long-term memory.

## Practical Patterns

- Standard lifecycle: read relevant memories → extract candidate facts in the background after the session → verify sources and policies → update (add/change/delete/merge). Keep extraction separate from the main dialogue loop so responses are not blocked.
- Memory accumulates through interaction rather than being built once. Maintain both incremental absorption and periodic consolidation, using the same rules as shared knowledge bases; see `book/chapter3.md`, “How Knowledge Should Be Updated.”
- Framework tradeoffs:
  - **Mem0:** v3 extracts facts with one LLM call and only ADDs, without write-time disambiguation. Retrieval blends semantic similarity, BM25 keywords, entity matching, and temporal ranking. It avoids history loss from erroneous UPDATE/DELETE and uses fewer LLM calls (LoCoMo 71.4→92.5; LongMemEval 67.8→94.4). Mem0-g graph memory is historical: OSS v3 removed external graph storage and `relations` returns; entity links only weight internal retrieval. v2 and the 2025 paper instead decide ADD/UPDATE/DELETE/NOOP at write time, keeping memory compact but making a mistaken update irreversible.
  - **Memobase:** focuses on user profiles: configurable Profile slots (topic → subtopic, such as basic_info → name and work → job title) plus chronological Event Memory (“when did we last discuss the budget?”). Buffered batching accumulates conversations until a size or time threshold, then extracts once, amortizing LLM cost and maintaining low query latency.
  - **Custom reference architecture:** episodic, semantic, and procedural long-term memory plus an explicit working-memory layer managing the current task and interacting bidirectionally with long-term memory. Working memory is a selectively activated dynamic subset, unlike the append-only trajectory. Choose by business needs rather than building everything.
- Once memories number in the thousands, improve recall with retrieval: RAG pipelines and structured indexes apply to memories too. The most advanced form is a **two-layer memory architecture**: a few critical facts structured as Advanced JSON Cards remain in context for a global overview, while massive raw dialogue is context-enriched and retrieved on demand for precise details. Resident facts alone lose detail; retrieval alone misses hidden cross-session relationships.
- Privacy: prefer local small models for redaction, such as Qwen3 0.6B under Ollama, runnable on CPU with 1.7b/4b upgrade options. Sending logs to the cloud for redaction defeats its purpose. Recognize structured IDs/bank numbers, semistructured addresses, and natural-language secrets (“my password is abc123”); output type/location/confidence through JSON Schema. LLM redaction recall exceeds 95%. For very high throughput, combine regex screening with deep LLM analysis.
- Frontier note on multimodal memory: hard-to-describe faces and voices have three approaches—raw data + text descriptions (retrieve an image, then inspect the original); embeddings (about 1 token per face, so 1000 tokens can hold 1000 faces, efficient but requiring residency); or Engram hash N-gram slots in a pretrained model (scalable but model-specific and potentially less precise than embeddings). Per-user fact-LoRA can directly repeat facts reasonably, but fails on indirect reasoning.
- Evaluation: approximately 20 cases per level. L1 cases usually have one session; L2/L3 cases span time and entities across sessions, totaling roughly 50 turns per case. The tested Agent generates memory from session one, then modifies it using that memory and the next session. It may access memory only, never reread original conversations. After memory generation, answer a new user question and score against a reference with LLM-as-a-judge.
- Technology by evaluation level: L1 needs reliable storage/retrieval; L2 needs retrieval covering all cross-session information and decisions about proactive clarification; L3 is hardest, requiring both a global overview (resident structured facts) and precise details (on-demand raw dialogue).
- Write engineering: extract in the background without blocking responses; retrieve similar memories before deciding add/change/delete/merge to prevent unlimited accumulation; preserve timestamps so “moved to Shanghai” and “lives in Beijing” can be ordered rather than overwrite each other.

## Common Pitfalls

- Storing short-lived details as long-term memory, expanding noise and diluting retrieval.
- Storing facts without context: without backstory explaining for whom and why, names such as Dr. Zhang become ambiguous.
- One format for everything: Simple Notes disconnect relationships, Enhanced Notes are redundant and hard to update, and JSON Cards lose multiple dimensions.
- Aggressive write-time UPDATE/DELETE irreversibly loses history after one mistake; ADD-only without consolidation accumulates contradictions indefinitely.
- Updating without consolidation: one fact spreads across entries, old and new coexist, and summaries drift from evidence. Verify against original evidence periodically rather than rewrite old summaries using each other.
- Keeping the latest statement automatically or guessing during conflicts: trace original sources and check whether statements hold under different times, entities, or conditions. Record applicability; retain an unconfirmed state when evidence is insufficient.
- Retrieving trajectories directly as long-term memory: append-only raw logs are noisy. Distill cross-session memory before storing it.
- Allowing raw sensitive information into LLM context and system logs.
- Redacting through cloud APIs: logs themselves may contain sensitive data, so sending them to the cloud defeats privacy protection.

## Companion Code

- `chapter3/user-memory/` — Four modes behind one interface (notes/enhanced notes/JSON cards/advanced JSON cards), with dialogue separated from background memory processing. Switch modes to compare artifacts from the same case.
- `chapter3/mem0/` — Mem0 v3 ADD-only extraction plus hybrid retrieval for LoCoMo long-context multisession scenarios.
- `chapter3/memobase/` — Real Memobase SDK Profile + Event demonstration plus a custom Memobase-style Agent with episodic/semantic/procedural/working memory, importance decay, and clustering.
- `chapter3/user-memory-evaluation/` — Experiment 3-1: three levels, 20 cases per level and 50+ turns, multidimensional LLM-as-Judge scoring, and an offline keyword-recall baseline.

## Further Reading

- `book/chapter3.md`, “User Memory Systems”
