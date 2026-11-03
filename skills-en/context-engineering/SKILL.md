---
name: context-engineering
description: Use when designing Agent system prompts, context structures, or on-demand Skills. Covers API message roles and the static-prefix-plus-trajectory structure; system prompt structure, SOPs, executable business rules, and few-shot examples; progressive disclosure and SKILL.md authoring; context-layer defenses against prompt injection; and layout principles such as attention position preferences.
---

# Context Engineering: An Overview

Translated from the [Chinese original](../../skills/context-engineering/SKILL.md).

Context is everything an Agent actually sees at each decision point: system prompts, tool definitions, user messages, model responses, and tool execution results. Context engineering systematically designs, organizes, and supplies this information. Model intelligence is only the foundation; **context quality is the real key to Agent capability**. A moderately capable model with carefully organized context often outperforms a top model searching blindly with insufficient information.

## When to Use

- Design or refactor an Agent's system prompt, tool definitions, or message-list structure.
- Write or review a `SKILL.md`: metadata, routing description, and layered body.
- Diagnose unstable behavior, business-rule violations, repeated mistakes, or forgotten constraints.
- Decide which knowledge belongs in the system prompt and which should become an on-demand Skill.
- Assess security risks when external content such as webpages, documents, or email enters context.

## Core Principles

- **The API is stateless; the framework rebuilds context every turn.** The model does not remember the previous turn. The Agent framework must send the complete message history back each time.
- **Context = four message roles + the `tools` field.** `system` holds developer rules, usually in one message at the beginning; `user` holds end-user input; `assistant` holds model responses including `tool_calls`; `tool` holds framework execution results, associated with calls through `tool_call_id`. Tool definitions belong in the request's top-level `tools` field, not a message role.
- **Static prefix + dynamic trajectory.** The system prompt and tool definitions remain unchanged throughout; conversation history, or trajectory, grows with interaction. Subsequent techniques optimize this list's content and structure.
- **The model decides; the framework executes.** The model only requests tool calls. The Agent framework executes them. The cycle “request → tool_calls → execution → return results → request again” is the API implementation of the ReAct loop.
- **Three minimum information requirements:** code information (structure, responsibilities, conventions), process standards (branches, commits, review, CI/CD), and environment information (configuration, databases, secret management). The Agent will make mistakes wherever a category is missing.
- **The new-employee test:** if an intelligent new employee cannot tell what to do after reading your system prompt, neither can the Agent.
- **In-context learning is retrieval rather than reasoning:** models are good at finding existing content, but poor at inductive counting and aggregation during a forward pass. Position preferences, or Lost in the Middle, mean attention is highest at the beginning and end; put critical information at those ends.
- **Align with the model provider's training methods:** prefer interaction patterns the provider has specifically trained, such as Skills and tool_search. They perform best within their own ecosystem.

## Practical Patterns

### 1. Four Dimensions of System Prompts

- **Tone and style:** use explicit instructions rather than requests, such as `You MUST answer concisely with fewer than 4 lines`; when a task cannot be completed, `keep your response to 1-2 sentences` and do not explain why it cannot be done. Capitals such as `NEVER do X` draw more attention than `Please avoid doing X`, but overuse dilutes their effect. Reserve them for truly critical constraints.
- **Structure:** Markdown headings (`#`, `##`) provide human-readable hierarchy; XML tags provide precise machine-readable semantics (`<file_operation>`, `<network_request>`), with meaning in the tag names themselves. Combine both: Markdown organizes the logic, XML defines semantic boundaries.
- **Process rather than accumulated rules:** write a branching SOP (Step 1 Validation → Step 2 Classification → ... → Step 5 Verification). The model should always know its stage and handle exceptions accordingly, rather than search hundreds of disconnected rules.
- **Make business rules executable:** replace vague rules such as “choose an appropriate billing type for the task” with Boolean decisions, for example `NEVER use percentage_based_one_time for refunds and service cancellations. Use fixed_fee instead.` Map success-rate thresholds directly to behavior (>60% allows a refund; <30% rejects the task). Specify billing granularity ($0.05 per minute, round the total to whole dollars), and state that “savings” are calculated only from existing bills. **Product managers design prompts using production data; engineers encode them accurately without independently deciding business logic.**

### 2. Two Decision Points for Few-Shot Examples

- **Placement:** put examples in the system prompt to make them part of the static prefix for all requests; alternatively, construct a set of user/assistant messages in the first conversation turn, useful for choosing different example sets by session type.
- **Stability:** wherever placed, examples sit early in context and must remain byte-for-byte stable once chosen. Dynamically retrieving the “most relevant” examples per request rewrites the prefix each turn and continually invalidates the cache. Prepare a fixed set for each task category.
- **Quantity:** two or three carefully selected examples covering edge cases beat ten nearly identical ones, which consume context and dilute attention to the rules themselves.

### 3. Agent Skills: Three Layers of Progressive Disclosure

- **Layer 1, metadata:** the YAML frontmatter's `name` + `description` in `SKILL.md` are visible before the body loads and guide routing. Write `description` as a routing condition, not a feature introduction: define when to use and not use it, with typical counterexamples. A broad description such as `help with backend` triggers on any backend task and inevitably misroutes.
- **Layer 2, core workflow:** load the complete `SKILL.md` only when the model decides it needs the Skill. Slash commands are intercepted and expanded locally by the client; autonomous model activation requires another ReAct round trip, with the body appended as a user message.
- **Layer 3, details:** file references lead to subsidiary documents such as `html2pptx.md` and `reference.md`, which the Agent explores selectively as needed.
- **Context cost:** all Harnesses keep a small catalog resident and load full bodies on demand. Skill bodies are appended to the trajectory where invoked, leaving the established prefix unchanged and preserving KV Cache reuse.
- **Four parts of SKILL.md** (Baoyu's “Illustrated Guide to Skills”): role and audience; only 3–5 core principles with positive and negative examples; a prohibition list documenting frequent errors and unauthorized actions with legitimate exceptions; references containing glossaries, templates, and sample writing. Express rules as “scope + action + exception + verification,” rather than an ever-growing banned-word list.
- Skills can also bundle executable tools and templates, remaining self-contained and independently version-controlled.

### 4. A Context Layout Decision Skeleton

```python
stable_prefix = system_message            # Do not change once established
stable_tools  = core_tool_schemas         # Fixed order
trajectory    = load_message_history(session)
status_message = make_status_message(derive_current_state(trajectory))

if estimated_tokens(stable_prefix, trajectory, status_message) > budget:
    trajectory = compress_old_evidence(
        trajectory, preserve=[decisions, constraints, failures, citations]
    )

request.messages = [stable_prefix] + trajectory + [status_message]
request.tools = stable_tools
```

### 5. Context-Layer Defenses Against Prompt Injection

The first defense helps the model distinguish instructions from data:

- **Source labels:** wrap external content in `<external_content source="webpage">...</external_content>` and label its source before insertion.
- **Structured roles:** strictly pass information through system/user/assistant/tool roles. Never mix tool results into user messages, which removes the model's basis for identifying sources.
- **Input sanitization:** filtering common injection phrases such as “ignore previous instructions” is only supplementary and is easily bypassed through wording variations.
- **Treat Skills and status bars as injection surfaces too:** a Skill institutionalizes loading external content as instructions. Review unknown-source Skills before installing them. Models highly trust status information; summaries derived from externally contaminable data can exploit that trust.

## Common Pitfalls

- **Putting dynamic information such as timestamps or user balances into the system prompt:** rewrites the prefix each turn and invalidates the entire cache; see `kv-cache-design`.
- **Unstructured collections of rules:** experiment 2-4 retained all rules but shuffled their organization and removed heading hierarchy. Task success fell by more than 30%, with frequent violations of critical business rules.
- **Removing tool descriptions:** keeping function signatures but removing descriptive text increased tool-call errors by 45%.
- **Overinvesting in tone:** style substantially changes expression but has relatively limited impact on task completion.
- **Retrieving examples dynamically per request:** continually invalidates the cache and makes behavior irreproducible across example sets.
- **One system prompt for every scenario:** wastes tokens on irrelevant information and dilutes attention. Split into on-demand Skills.
- **Giving the model discretion over business rules:** models excel at following complex instructions and extracting information from long context. Vague rules yield different classifications for the same task at different times.
- **Treating context defenses as foolproof:** they only reduce attack success. Permission controls, sandbox isolation, and independent review of high-risk operations are a separate layer.

## Companion Code

- `chapter1/context/` — Context-aware Agent supporting multiple providers; ablation experiment 1-1 compares full / no history / no reasoning / no tool calls / no tool results.
- `chapter2/prompt-engineering/` — Experiment 2-4: a Tau-Bench-based prompt-engineering ablation framework quantifying tone/style, instruction organization, and tool descriptions against task completion.

## Further Reading

- `book/chapter2.md`, “Context: The Key to an Agent's Capability Ceiling”
- `book/chapter2.md`, “How Agents Call Large Models: Understanding API Context Structure”
- `book/chapter2.md`, “Prompt Engineering: Optimizing System Prompts”
- `book/chapter2.md`, “Dynamic Prompts and Agent Skills”
