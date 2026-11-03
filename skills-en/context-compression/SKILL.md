---
name: context-compression
description: Use when context grows excessively, tool results overflow the window, or Agent decisions deteriorate over turns. Covers three motivations for compression (length/cost, reasoning quality, context anxiety), compression as retrieval preparation, measurements of six strategies, five production compression layers, four design principles, and subagent isolation as an alternative.
---

# Context Compression Strategies

Translated from the [Chinese original](../../skills/context-compression/SKILL.md).

Compression addresses the opposite problem: **how to subtract from context**—when to compress, how to compress, and why to do it even before the window is full. The central insight is that **in-context learning is fundamentally retrieval rather than reasoning**. Compression turns conclusions requiring thought into directly retrievable knowledge.

## When to Use

- Tool results routinely span tens of thousands of characters and fill a 128K window within a few interactions.
- On long tasks, an Agent cannot find key information despite spare window capacity, or repeatedly revisits resolved issues.
- The model wraps up or draws conclusions prematurely before completing the task.
- Design triggers, compression targets, and retention policies, or investigate side effects.
- Weigh delegating large tasks to subagents against doing the work in the main Agent and compressing afterward.

## Core Principles

- **Three motivations at different levels:** (1) length and cost constraints: finite windows, higher token cost, and latency; (2) better reasoning: summarized knowledge is easier to use than scattered raw results from a dozen searches that require repeated retrieval across tens of thousands of tokens; (3) less **Context Anxiety**: a model expecting to exhaust its window may finish before the task is done, so compressing well before exhaustion can improve decisions.
- **The context window is half a retrieval engine:** retrieval is powerful, like built-in RAG on each forward pass, but the distillation layer is missing. Context is never automatically counted, indexed, or summarized in place. Conclusions about it must be recalculated from raw records at cost growing with content volume N.
- **Compression and status bars are two sides of one coin:** status bars **add** precomputed conclusions through deterministic code; compression **replaces** bulky raw records with conclusions distilled through an extra LLM call.
- **Context Rot ≠ overflow:** overflow means it no longer fits; rot means it fits but cannot be found. Rot is subtler: the Agent appears to work while decisions degrade. Attention is spread over more tokens, and irrelevant content dominating context noticeably harms quality.
- **Distill explicitly rather than expect automatic learning:** actively supply dense, structured knowledge rather than leaving the model to retrieve passively from huge information volumes.
- **Compression most readily loses** early architecture decisions, reasons behind constraints, and failed approaches. Regularly document progress rather than scatter everything through execution history.
- **Isolation beats compression:** compression is a lossy remedy after information enters context; isolation keeps bulky intermediate information out of the main context altogether.

## Practical Patterns

### 1. Measurements of Six Strategies (Experiment 2-10)

Task: identify and track the career status of OpenAI cofounders; Kimi K3, with a native window of roughly 1M, deliberately limited to a 128K budget to trigger compression.

| Strategy | Method | Iterations | Tokens | Compression ratio | Issue |
| --- | --- | --- | --- | --- | --- |
| No compression | Retain complete raw results | Overflow failure at 5 | ~165,000 | — | 7 searches total about 367,000 characters (52,000/search average), exhausting 128K after a few searches |
| Individual summaries | Independently summarize each result in 2-3 paragraphs | 12 | 276,608 | 10.9% | Fragmented information; multiple pages repeat the same events |
| Combined summary | One integrated summary of all results | 10 | 93,449 | 4.3% | Overlong input must be truncated, risking information at the end |
| **Context-aware** | Include query intent and accumulated information in the compression prompt | 7 | 40,157 | ~3.0% | Best balance |
| Context-aware with citations | Intelligent compression plus a source URL for every fact | 7 | — | — | Lossy content, lossless index; original sources remain traceable |
| Adaptive windowing | Threshold trigger + batch compression + duplicate protection | — | 174,601 | — | Initially retains all raw information for maximum flexibility |

> Compression ratio means compressed size / original size; a smaller number means stronger compression. Context-aware compression reduced token use by over 75%.

**Context-aware prompt writing:** specify `Given the search query: {query}` and `Current context: {context}` to guide targeted summaries. In experiments, compressing roughly 150K characters to 2K still preserved founder names, role changes, and other information needed by subsequent tasks.

### 2. Three Adaptive-Windowing Mechanisms (Recommended Default)

- **Threshold trigger:** monitor context utilization and activate compression only above 80% of the window, such as 102,400 / 128K prompt tokens.
- **Batch compression:** compress all unmarked tool results at once when triggered, rather than piecemeal each turn.
- **Duplicate protection:** add `[COMPRESSED]` to ensure compressed content is never processed again.

### 3. Coexisting with KV Cache

Compression does not change context during one API call. The framework preprocesses messages **between API calls**:

1. **Never change System Prompt or Tool Definitions:** the static prefix remains cached.
2. **Compress tool results in conversation history:** cache after the replacement becomes invalid, while earlier cache remains valid.
3. **Make the tradeoff deliberately:** without compression, overflow fails; compression loses some caching but bounds length and increases information density. **Frequency matters: frequent compression repeatedly breaks caching. Batch near the threshold rather than compress every turn.**

### 4. Five Production Compression Layers (Claude Code Reference)

Information has different shelf lives; match strategy to expected lifespan, ordered from lower to higher cost:

1. **Tool-result budget control:** save large output to disk and show only a summary preview. Freeze replacement decisions to preserve cache consistency.
2. **Delete noise directly:** remove low-value content, such as search results of which only a few lines were used, without summarizing. Summarizing noise wastes tokens.
3. **API-level microcompression:** context-editing capabilities tell the server to remove specified tool results from the prefix while local messages stay unchanged. This needs no local implementation, but still invalidates cache after the removal point and requires a rebuild. Use near overflow when that cost is unavoidable, not frequently.
4. **Archival summaries:** create structured summaries per turn, like git log retaining individual records rather than git squash merging them. Preserve conversational logic.
5. **Full compression:** complete LLM-driven compression as a last resort, in two stages (session memory first, full context if needed), with a **consecutive-failure circuit breaker**. Production data shows many sessions trapped in repeated compression failures; a breaker prevents continued spending.

### 5. Four Compression Design Principles

- **Information value is uneven:** critical decision points (such as a personnel list) > supporting evidence (news details) > redundant noise (navigation and footer ads).
- **Semantic completeness:** do not reduce `Sutskever left OpenAI in May 2024` to `Sutskever left`. Time and company are indispensable.
- **Task relevance:** the same material should yield different compression for “find the founders' names” and “understand personal backgrounds.” Retrieval needs breadth, analysis needs depth, and creative work needs inspiration triggers. Ideally, Agents adapt strategy by task type.
- **Compression is understanding:** effective compression requires deep semantic understanding. Its module should approach the main model's ability, forming a recursive model-calling-model architecture. Explicit results are inspectable and reusable across sessions.

### 6. Four Information Categories That Must Survive

Every strategy must explicitly preserve **decisions, constraints, failures (failed approaches), and citations (source references)**.

### 7. Isolation Beats Compression: Subagent Context Isolation

Compare the same task, “find the function handling payment callbacks in the codebase”:

- **Main Agent searches itself:** raw code from a dozen files can add tens of thousands of tokens; most becomes permanent noise after locating the target and must later be compressed away.
- **Delegate to a search subagent:** main context gains only two messages, a task description and a conclusion (“`handle_callback` is in `src/payment/callbacks.py`, with two other call sites”). Tens of thousands of intermediate tokens disappear with the subagent's context.

The cost is that a subagent cannot see the main Agent's full context; **task descriptions must be self-contained with clear goals**. Claude Code's Task tool and retrieval subagents in Deep Research systems implement this pattern in production.

## Common Pitfalls

- **Compressing every turn:** repeatedly breaks the KV Cache prefix. Use threshold-triggered batch compression.
- **Summarizing noise:** summarizing results used for only a few lines wastes tokens; delete them.
- **Losing times or subjects:** incomplete semantics can be worse than no compression because the model reasons from incorrect facts.
- **Using a much weaker compression model:** compression is understanding; large capability gaps lose critical information.
- **Treating compression only as a length solution:** it can improve reasoning and reduce context anxiety even with ample window space.
- **Having the model count and aggregate:** attention excels at retrieval; aggregation requires traversing records and maintaining counts. Precompute conclusions into context.
- **Keeping progress only in execution history:** architecture decisions, reasons for constraints, and failed paths are easily lost. Write them to documents regularly.
- **Vague subagent assignments:** isolation demands self-contained task descriptions, or conclusion quality cannot be assured.

## Companion Code

- `chapter2/context-compression/` — Experiment 2-10 implements and compares 6 strategies (`no_compression` / `individual` / `combined` / `context_aware` / `citations` / `windowed`), reporting success, time, tokens, compression ratio, and overflow counts. Run one with `python experiment.py -s context_aware`; use `-m moonshot-v1-128k` with `CONTEXT_WINDOW_SIZE=128K` to reproduce overflow.

## Further Reading

- `book/chapter2.md`, “Context Compression Strategies”
