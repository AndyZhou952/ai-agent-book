---
name: kv-cache-design
description: Use when optimizing Agent inference latency and cost or diagnosing slower first tokens and lower cache hit rates. Covers prefix invariance, three strict rules, five cache-breaking context patterns, Chat Templates and historical chain-of-thought handling, and architectural constraints including cache boundaries, byte-aligned subagents, and frozen replacement strings.
---

# KV Cache-Friendly Context Design

Translated from the [Chinese original](../../skills/kv-cache-design/SKILL.md).

KV Cache stores previous tokens' key/value pairs (K/V), so the next turn computes only new portions. **The reusable token prefix must remain unchanged.** If the sequence differs at some position, KV states at the first different token and all subsequent positions must be recomputed; earlier positions are unaffected. The corresponding cross-request mechanism is Prompt Cache. One apparently harmless line of dynamic code can slow the entire inference process by an order of magnitude.

## When to Use

- Diagnose higher time to first token (TTFT), doubled inference bills, or low `cached_tokens` hit rates.
- Design Agent context layout: what goes in system and what is appended at the end.
- Decide how to derive subagents, load tool definitions, or restore sessions.
- Handle returning historical chain of thought (`reasoning_content` / thinking blocks) and moving trajectories across models.
- Evaluate any proposal for supplying dynamic information to context.

## Core Principles

- **Rule one: do not change system prompts or tool definitions once established.** Any change, even one extra space, can alter the token sequence and prevent reuse from the first changed token onward. Earlier changes mean more recomputation and billed tokens, usually with greater latency impact, measured at several times in practice.
- **Rule two: always append dynamic information at the end.** Add timestamps, user status, and TODO progress as new messages at the conversation's end, rather than changing the existing system prompt.
- **Rule three: use the standard API format; do not concatenate messages yourself.** Chat Templates translate structured messages into the fixed token sequences seen in training. Hand-built `USER: ... ASSISTANT: ...` text deviates from training format and weakens multistep reasoning.
- **Transformer layers are sequential:** changing token k leaves earlier states intact but affects representations from k onward, layer by layer. The cache can only be retained before the first different token.
- **Without caching, prefill attention computation grows quadratically with context length.** Caching avoids recomputing historical K/V projections, but each new token's attention still traverses all cached K/V, growing linearly. This explains slower long-context decoding and memory-bandwidth bottlenecks.
- **Reading Prompt Cache costs far less than initial computation**, approximately one tenth at the scale of Anthropic, DeepSeek, and GPT-5. Activation and billing differ substantially; check current documentation before use.
- **Caching is an architectural constraint, not an after-the-fact optimization.** Earlier design consideration lowers later engineering cost.

## Practical Patterns

### 1. Layered Context Layout

```text
[ system prompt ][ tool definitions ]   Static prefix, byte-for-byte stable; cached across requests/users/sessions
[ user / assistant / tool ... ]         Trajectory, append-only
[ status bar / new tool schema / dynamic information ]    Append at the end
```

- Each runtime condition (OS, current mode, user preference) placed before the cache boundary doubles cache-key variants. N binary conditions create 2^N combinations (3 conditions = 2x2x2 = 8 cache keys). **Put all dynamic elements after the boundary.**
- Prompt ordering is determined first by cache economics and then by semantic logic.
- Appending on-demand tool schemas is safe: causal attention means each token's KV depends only on earlier tokens, so additions do not change cached tokens' K/V. A new schema is computed once when first encountered, then joins the growing prefix and keeps hitting in subsequent turns.

### 2. Three Cache-Consistency Designs

- **Byte-aligned subagents:** when deriving a subagent or making a side query, its prompt, tool definitions, model configuration, message prefix, and thinking configuration must match the parent's byte for byte to hit the provider's Prompt Cache. Alignment is unnecessary when the framework intentionally uses a different context or prompt.
- **Freeze replacement strings at first appearance:** when replacing large tool output with a summary preview, persist the replacement string. Use exactly the same string after a session restart so restored messages match the cached byte stream.
- **Freeze thinking settings along with the prefix:** whether to return CoT and which fields to return are part of the prefix; changing them also invalidates caching.

### 3. Chat Templates and Historical Chain of Thought

- A Chat Template is the envelope format: API messages are the letter's content; the template specifies boundaries and roles through special tokens such as `system`, `user`, `assistant`, and `tool`. Model families such as Qwen, Llama, and Gemma use different formats. Servers convert automatically; developers need not write templates manually, but should know they exist.
- **The actual cost of deviating from the standard:** Qwen3 retains historical reasoning inside `<think>` to preserve multistep continuity, but its Chat Template assumes a new user query changes the topic and clears previous thinking. Mislabeling tool results as user messages triggers this clearing, like taking away scratch paper halfway through a calculation.
- **Providers differ greatly in returning historical thinking; check documentation before migration:**
  - DeepSeek R1: strip all historical thinking and return only `content`, not `reasoning_content`; historical CoT never appears in training input.
  - DeepSeek V4: the reverse. Whenever a request carries `tools`, every assistant message between two user messages must return `reasoning_content` unchanged, even if that turn did not call tools; otherwise the API returns 400. Kimi K2 and GLM-5 use the same protocol.
  - Claude: during tool-call loops, return signed thinking blocks unchanged. After new user input, the server ignores thinking blocks preceding the last user input.
- **In multiround conversations these differences affect token savings; when another provider must continue a partially completed trajectory, they become actual interface errors.**

### 4. Correct and Incorrect Patterns

| Pattern | Consequence | Correct approach |
| --- | --- | --- |
| Dynamic system prompt (timestamp) | Cache invalidated from the timestamp onward; TTFT rises from 0.5s to 3-5s | Append time as a user message or fetch it with a tool when needed |
| Dynamic user profile (balance/quota) | Rewrites the prefix each turn, invalidating the cache | Retrieve as needed through a dedicated state-management mechanism |
| Dynamically sorted tool definitions | Invalidates everything from the first changed tool; each definition can span hundreds of tokens | Fix the order; experiments show almost no effect on tool selection and significant performance gains |
| Sliding-window history | Breaks prefix consistency and loses key tool results | Use compression/status bars; see `context-compression` |
| Text formatting (USER:/ASSISTANT:) | Deviates from training format: repeats completed actions, ignores tool results, outputs text when tools are needed | Use standard structured messages |

### 5. Implementation Checklist

- [ ] The system prompt and tools order are code constants without runtime interpolation.
- [ ] Time, user state, TODOs, and other dynamic information are appended at the end.
- [ ] Tool order is deterministic (registration or lexical order), never usage-frequency order.
- [ ] Compression/summarization happens only between API calls and leaves system and tools unchanged.
- [ ] Session restoration rebuilds a message sequence matching the cached byte stream.

## Common Pitfalls

- **Adding `Current time: {{now}}` to the system prompt so the Agent knows the time:** the most frequent mistake. A customer-service Agent handling 100,000 conversations daily saw TTFT rise from 0.5 seconds to 3-5 seconds and its monthly inference bill nearly double after adding one timestamp line.
- **Dynamically sorting tools by usage frequency:** subtle but highly destructive, with no model-capability benefit.
- **Controlling context through sliding windows:** with a 10-turn window, key results from turn 2 are gone by turn 15. Inference on truncated conversation raises error rates. Experimental Agents often looped, repeating the same tool calls because they had forgotten the results.
- **Passing tool results as ordinary user messages:** breaks Chat Template roles, triggers CoT clearing, and removes the model's basis for distinguishing instructions from data.
- **Rebuilding the whole message list each turn:** even identical intended content invalidates caching if reconstruction introduces byte differences through timestamps, random ordering, or serialization order.
- **Assuming appending is free:** catalogs and Skill bodies must be processed on their first request; reuse only becomes possible in subsequent requests with a stable prefix.
- **Treating prefix rules as immutable and abandoning optimization:** research such as Models Take Notes at Prefill shows that models write field conclusions into downstream KV states during prefill, while the fields' own tokens often contribute less than 1%. Explicit CoT can enable editing with about 1% of the compute, or RoPE repositioning can combine cache blocks (on vLLM, p90 TTFT reductions of tens to hundreds of times, approximately 98.5% hit rate, and logit cosine similarities of 0.90-0.999 across 12 models). This remains research; **production systems should still follow the three rules above by default.**

## Companion Code

- `chapter2/kv-cache/` — Experiment 2-3: KV Cache comparison for a ReAct Agent in correct mode and five antipatterns (dynamic_system / shuffled_tools / dynamic_profile / sliding_window / text_format), measuring TTFT, hit rate, and token usage; supports offline `--report` comparison and `--cache-price-ratio` cost estimates.

## Further Reading

- `book/chapter2.md`, “KV Cache-Friendly Context Design”
