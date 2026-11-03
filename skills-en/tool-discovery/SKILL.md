---
name: tool-discovery
description: Use when tool counts grow and definitions fill agent context—designing discovery and progressive disclosure, implementing discover_tools and two-level semantic routing, choosing indexing/retrieval/active discovery/skills, handling KV cache disruption from dynamic loading, or evaluating token cost and selection accuracy. Includes a five-step loading workflow, tool-count thresholds, and key experimental results.
---

# Tool Discovery and Progressive Loading

Translated from the [Chinese original](../../skills/tool-discovery/SKILL.md).

## When to Use
- More MCP servers make tool schemas consume thousands to tens of thousands of system-prompt tokens.
- A library grows to hundreds of tools and the model starts choosing incorrectly or overlooking tools.
- Design a disclosure layer that exposes an index and loads definitions on demand.
- Implement active discovery: the agent declares a capability gap → semantic matching → dynamic schema injection.
- Make architectural tradeoffs among tool count, token cost, and accuracy.
- Diagnose falling prompt-cache hit rates after dynamic tool loading.
- Give small models (around 4B parameters) usable access to hundreds of tools.

## Core Principles
- **Scale itself hurts correctness.** Beyond 100 tools, even leading models tend to make selection errors. Listing everything also consumes many tokens, and every toolset change invalidates the KV cache.
- **Three increasingly on-demand answers**: ① hierarchical organization + on-demand loading (prepare definitions without injecting all of them); ② active discovery (the agent identifies a gap, declares its need, and the system matches/injects); ③ skills (treat tools as references to consult, rather than formal definitions).
- **Disclosure strategy and capability form are independent decisions.** A server with hundreds of MCP tools can expose only an index; twenty or so skills may still need hierarchical retrieval. Do not default to keeping everything in context.
- **Expose only an index first.** Cursor syncs tool descriptions to a folder; agents initially see a name index and look up definitions as needed. A/B testing showed a **46.9% reduction** in total tokens on MCP-related tasks.
- **A proxy tool of about 200 tokens** (pi-mcp-adapter) can support the full "search → inspect definition → invoke" discovery flow. MCP servers can even start lazily on first use. Using MCP as a protocol and exposing all definitions at session start are independent decisions.
- **One-time retrieval has inherent limits.** Retrieval preselection matches the initial query once; "Debug the file" may actually require a cross-domain chain of file access, code analysis, and command execution. All needs cannot be predicted at task start.
- **Matching must be hierarchical.** Group tools by server (similar to apps), identify a server first, then match tools within it. This shrinks the space from thousands of tools to tens of servers × tens of tools each, reducing compute and cross-domain semantic confusion.
- **Schemas stay where first inserted; the static prefix only grows and is never revised.** Append full schemas to the end of context and keep them at their first injection positions so they remain cacheable as ordinary history. "Definitions must be at the very beginning" is no longer an iron rule.

## Practical Patterns

### 1. Choose a Strategy by Scale and Scenario
| Tool scale | Recommended approach |
|---|---|
| A dozen or so | Inject everything; no extra mechanism needed |
| Tens | Expose only an index; look up definitions on demand |
| Hundreds | Retrieval preselection (inject top-k) |
| Hundreds to thousands | Active discovery (meta-tool + two-level semantic routing), or progressive skill disclosure |
| Thousands | Skills / Skill Hub; dedicated tools require a separate index layer |

Organize hierarchically by information-source type and explain groups explicitly in the system prompt so the LLM can locate them: **search tools** (active lookup: web/knowledge-base/file search), **reading tools** (extract from known locations: pages, documents, DB queries), **parsing tools** (unstructured data: OCR, video analysis, audio transcription), and **query tools** (structured sources: weather, stocks, public database APIs).

### 2. Five-Step Progressive Tool Loading (Core Mechanism)
1. **Index**: inject only a thin startup index—tool/server `name` + `description` (hundreds of tokens), without full schemas.
2. **Declare a need**: during reasoning, the agent identifies a capability gap and generates a structured request, such as `<tool_request>server: GitHub for repository operations; tool: search repositories by keyword</tool_request>`. Keep only a few basic tools (`web_search`, `code_interpreter`) and a `discover_tools` meta-tool in the system prompt.
3. **Semantic matching**: use an offline-built embedding index with incremental updates for **server-level → tool-level** routing; return 3-5 candidates with full schemas.
4. **Inject schemas**: **append schemas to the end of context** (as ordinary user/assistant messages) and keep them at those original trajectory positions. Maintain only a short tool-name list in the status bar. Later turns cache them as ordinary history.
5. **Invoke**: use the injected tools and reuse loaded tools for similar needs without loading again.

If candidate similarities at both routing levels fall below threshold, **explicitly return "not found"** so the agent can reformulate, implement the capability manually with basic tools, or create a new tool.

### 3. Measurable Comparison of Three Strategies
| Strategy | Injected content | Suitable for |
|---|---|---|
| Full injection (all-tools) | All N schemas; tokens grow linearly with catalog size | Small-toolset baseline |
| Retrieval preselection (retrieval) | Semantic top-k for the initial query | Hundreds of tools, predictable needs |
| Active discovery (active) | Agent iteratively declares needs; context grows on demand | Thousands of tools, multistep cross-domain tasks |

Measured with top-k=5: full injection used 3,857 tokens for 35 tools, rising to 40,258 for 400 tools; retrieval stayed around 540-550 tokens with 100% recall. On-demand selection changes "Which tool?" into "Which reference entry?", without cost expanding alongside the ecosystem.

### 4. Dynamic Loading and KV Cache
- **Root cause**: putting all definitions in the static prefix invalidates the entire cached segment whenever a new tool loads.
- **Solution**: append schemas at context end, keep the static prefix stable, and preserve schemas at their original trajectory positions for later cache hits as ordinary history.
- **Native API support**: OpenAI `tool_search` + `defer_loading` (preserve `tool_search_output` items at their original positions in later requests; no need to reload the same tool); Anthropic `tool_reference` (blocks expand inline at their original history positions; official docs explicitly describe continued cache hits on later turns); Codex CLI enables `tool_search` by default.
- **Only two cases actually require recomputation**: prompt-cache TTL expiration (the whole prefix is recomputed, not a tool-definition-specific cost), and changing/removing/reordering loaded tools (invalidation from the change point onward).
- **Cost**: post-training must teach the model to understand definitions distributed throughout context.

### 5. Skills: Turn Discovery into On-Demand Reference Lookup
No embedding index, retrieval meta-tool, or infrastructure such as `tool_search`/`tool_reference` is required. At startup, the agent sees only a thin catalog (`name` + `description` per skill). When the current context needs a capability, it reads the corresponding sub-skill and follows references to scripts or subdocuments. Like a reference book, it is consulted through an index rather than read cover to cover. Dedicated tools need a separate infrastructure layer to achieve the same progressive disclosure—that is why those mechanisms exist.

### 6. Configuration for Small Models
Experiment 4-1 (Qwen3-4B with 120+ tools):
- Control: inject all schemas into the system prompt at once (over 50K tokens) → severe instruction-following degradation, choosing Web Search instead of Yahoo Finance for stock prices or "forgetting" a tool and failing.
- Experimental: keep only `web_search`, `code_interpreter`, and `discover_tools` in the system prompt. `discover_tools` takes natural-language needs and returns 3-5 embedding-similarity candidates with full schemas. Append definitions to conversation history, update the status-bar name list, and prompt the model to call `discover_tools` proactively for capability gaps.

## Common Pitfalls
- Listing everything after tool counts exceed a hundred, causing wrong/missed choices and wasting context and tokens.
- Building an index without retrieval or hierarchy until the index is as long as the full list.
- Matching only the initial query with retrieval preselection and leaving later capability gaps unhandled.
- Inserting new schemas near context start or reordering loaded tools, invalidating the cache from the change point.
- Returning a low-confidence "best guess" when both routing levels are below threshold rather than clearly reporting not found.
- Building embedding indexes, incremental updates, KV-cache handling, and special training for weak models without evaluating the lighter skills route.

## Companion Code
- `chapter4/active-tool-discovery/` — compares full injection / retrieval preselection / active discovery over 126 cross-domain tools; `python demo.py --offline` exercises the mechanism; `run_exact_experiment.py` is the formal experiment entrypoint.
- `chapter4/active-tool-selection/` — MCP-Zero-style teaching implementation: structured `<tool_request>`, server-level → tool-level semantic routing; `demo_comparison.py --offline` outputs recall/token/scale curves.

## Further Reading
- `book/chapter4.md`: "Too Many Tools: Hierarchical Organization and Active Tool Discovery."
