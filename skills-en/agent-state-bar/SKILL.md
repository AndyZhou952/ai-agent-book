---
name: agent-state-bar
description: Use when an Agent loops indefinitely, miscounts tool calls, forgets TODOs or goals, or spends ever more reasoning tokens on long tasks. Covers the status-bar mechanism, why it is injected as a final user message for KV Cache reuse, replacement versus persistent append implementations and their cost models, and five techniques (timestamps, tool counters, TODOs, detailed errors, and system state).
---

# Agent Status Bars: Improving Trajectory Management with Metadata

Translated from the [Chinese original](../../skills/agent-state-bar/SKILL.md).

Prompt engineering supplies static instructions, while an executing Agent also needs dynamic awareness of its own state and task progress. An **Agent status bar** organizes runtime state such as progress, environment changes, and tool-call counts into a structured summary that the framework continually injects at the end of context. Like the time, battery, and signal always visible at the top of a phone, it lets the model glance at the current situation before every new response and make better decisions.

## When to Use

- An Agent repeatedly makes the same tool calls or loops indefinitely, for example continuing to call after exceeding a limit.
- The model cannot count calls already made or remaining TODOs and violates explicit constraints.
- Reasoning tokens per iteration keep growing as context length increases on long tasks.
- The Agent focuses on local subtasks and forgets the user's original request and core constraints.
- A small model needs task adherence approaching that of frontier models.
- Design how state messages are injected and updated.

## Core Principles

- **The status bar is not the conversation's main content:** it is neither user content, model output, nor a tool result. It is an automatically generated framework summary injected at the **very end** of context, immediately before new model tokens.
- **The theoretical basis is that in-context learning is retrieval rather than reasoning:** models find information well but struggle to proactively aggregate and count during one forward pass. Facts such as “how many calls have been made” are scattered across raw records; rescanning and counting each decision consumes reasoning tokens and is error-prone.
- **It turns implicit state into explicit knowledge:** raw trajectories contain much redundancy and few key state facts. At very low extra token cost, the status bar presents information otherwise requiring thousands of tokens to scan.
- **It explicitly directs attention:** early goals and constraints can be buried by later tool output. Structured metadata at the end lies closer to new tokens and receives greater attention: a form of forced attention guidance.
- **Measured benefits:** with precomputed status information, smaller open-source models can approach frontier-model accuracy; reasoning tokens, latency, and cost per iteration fall by roughly an order of magnitude. Without it, reasoning grows **continuously** with context length; with it, it remains **approximately constant**.
- **A status bar is a context-compression technique:** code deterministically maintains conclusions about the trajectory, complementing LLM-driven compression.
- **Nonintrusive:** metadata is human-readable and inspectable. No fine-tuning is needed; it works directly with any language model.

## Practical Patterns

### 1. Injection Position: A Final User-Role Message

```text
messages: [
  { role: "system",    content: "You are a customer service assistant..." }  ← Fixed, already in KV Cache
  { role: "user",      content: "Help me cancel my Xfinity plan" }
  { role: "assistant", content: null, tool_calls: [...] }   ← Turn 1 decision
  { role: "tool",      content: "Call log..." }
  { role: "assistant", content: null, tool_calls: [...] }   ← Turn 2 decision
  { role: "tool",      content: "Call log..." },
  { role: "user",      content: "Can you call them again to follow up?" },
  { role: "user",      content: "<agent_status>
      Current State:
      - phone_call invoked 3 times (Xfinity: 3/3 max)
      - Current time: 2025-09-14 10:30:45
      - TODO: [1] Cancel plan (in_progress)
    </agent_status>" }                                        ← Framework-injected status bar
]
```

- **Why a user role rather than modifying system:** changing the system message breaks prefix caching. Here user is only a technical API-protocol choice, **not a claim that the content came from the end user**. The Harness borrows this message slot for automatically generated system state.
- Wrap it in `<agent_status>` tags so the model can recognize its special nature.
- Appending rather than modifying leaves all previously cached content unaffected.

### 2. Three Components of a Status Bar

- **Task planning:** a TODO list breaks complex multistep work into clear steps. At the trajectory's end it continually reminds the model of progress and subsequent goals, aligning actions with the overall plan rather than local subtasks alone.
- **Side-channel event information:** attach metadata such as exact time, location, and elapsed time since the last Agent response to each event, usually appended with that event.
- **Observation summary of the environment's current state:** system time, working directory, unusual-operation warnings such as “this tool has been repeated N times,” and conversion of implicit state to explicit observations. Update as work proceeds.

### 3. Two Update Implementations and Cache Costs

**Implementation one: replace each turn.** Before each API call, remove the previous state message and append the latest. Only one current status remains, but removing the old one invalidates caching from that position onward. This is the same mechanism as a dynamic timestamp; the difference is its position near the end, so invalidation covers only messages added since the last injection, usually one turn, while the earlier prefix remains reusable.

**Implementation two: persistent append.** Once injected, a status message stays in the trajectory permanently; each turn only appends a new one. Claude Code's `<system-reminder>` follows this pattern, retaining historical state in the transcript without deletion or modification. It is fully cache-friendly because the prefix stays stable, but stale status consumes tokens and requires the model to focus on the newest entry.

**Selection rules:**

- Small status, many messages between updates, bounded session length → **implementation two**: retaining old state is usually cheaper than repeatedly recomputing a long suffix.
- Large status, frequent updates, or long trajectories → **implementation one**: invalidate only the short suffix since the last injection and prevent stale-state accumulation.

**Approximate cost model:** each status is S tokens, the new suffix between updates is R tokens, N updates are expected, and cached input costs α times normal input:

```text
C_replace ≈ (N-1) * (1-α) * R
C_append ≈ α * S * N * (N-1) / 2
→ Prefer implementation two when α*S*N/2 < (1-α)*R; otherwise prefer implementation one
```

This estimate excludes context occupancy and ambiguity from stale state. Also consider provider cache billing and measured hit rates.

### 4. Five Status-Bar Techniques (Experiment 2-9)

- **Timestamp tracking:** prefix user messages and tool responses with `[2025-09-14 10:30:45]` (**not the system prompt**, which would break KV Cache). This supports temporal understanding, debugging, and auditing; time simulation helps interpret “yesterday's file” and “today's changes.”
- **Tool-call counter:** maintain a global dictionary of calls per tool; annotate responses with `Tool call #3 for 'read_file'`. Explicit counts trigger pattern recognition: check the path after the first failure, list the directory after the second, proactively stop and seek alternatives after the third. Its deeper value is implicit cost awareness.
- **TODO management:** provide `rewrite_todo_list` and `update_todo_status`. Each item has a unique identifier, content, status (pending / in_progress / completed / cancelled), and timestamp. This follows Manus's idea of manipulating attention through repetition. Experiments averaged **15 iterations** with TODOs versus **21** without, with frequent omitted subtasks in the latter.
- **Detailed errors:** four layers: error type and description, full argument JSON, stack trace, and targeted repair suggestions. For FileNotFoundError, suggest verifying the path, checking the working directory, and using absolute paths. This raised success at finding alternatives in error scenarios from **60% to 95%**, replacing blind retries with targeted analysis.
- **System-state awareness:** inject current time, working directory, OS, shell environment, and Python version. Working-directory tracking is especially important: update automatically after `cd`. OS information enables platform decisions such as `apt` on Linux and `brew` on macOS.

Individual techniques have limited effects; **their combination produces emergent benefits**. Timestamps + counters explain operation frequency and timing; TODOs + system state support environment-sensitive strategies; detailed errors + counters help the Agent both change strategy and understand repeated failures.

## Common Pitfalls

- **Putting timestamps in the system prompt:** directly breaks the KV Cache prefix. Inject as message prefixes or final status instead.
- **Asking an LLM to count all state in one batch:** models trust status almost unconditionally. If it says three calls, they accept three without recounting, although LLM counting is error-prone. **Maintain status with code wherever possible; if an LLM is necessary, extract records individually and aggregate with code.**
- **Ignoring status-bar poisoning:** models highly trust it. Summaries from externally contaminable sources, such as webpage fragments inserted directly into status, can exploit that trust.
- **Deleting raw records as soon as the status bar seems sufficient:** it is a **lossy projection**, precomputing only dimensions you anticipate. Counts and state tracking may retain only status to save tokens, but a single question involving an uncomputed dimension can cause accuracy to **collapse**. Check for uncovered query dimensions before deletion.
- **Accumulating state messages forever:** implementation two consumes tokens and creates ambiguity. Switch to implementation one if session length grows out of control.
- **Using a status bar instead of real task planning:** TODOs need tools supporting state transitions; “please track progress” in a prompt is insufficient.

## Companion Code

- `chapter2/system-hint/` — Experiment 2-9 (agent-status-bar framework): independently toggle timestamps, tool counters, TODOs, detailed errors, and system state. `python main.py --mode preview` compares model-visible context with and without status without an API key; `python run_experiment_2_8.py` runs the frozen controlled experiment.

## Further Reading

- `book/chapter2.md`, “Agent Status Bars: Improving Trajectory Management with Metadata”
