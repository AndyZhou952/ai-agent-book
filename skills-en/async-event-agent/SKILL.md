---
name: async-event-agent
description: >-
  Use when building asynchronous/event-driven agents, responding to external events (new email, webhooks,
  IM messages, timers, system alerts), or handling user interruptions and concurrent tasks during tool execution.
  Covers event loops and safe points, three event-trigger tool types, user communication and multichannel
  re-engagement, virtual identities and isolated environments, queued/canceling/parallel event strategies,
  urgency classification, and native asynchronous versus synchronous-interface compatibility routes.
  Keywords: asynchronous agent, event-driven, webhook, timer, heartbeat, task handle, interruption, steering, parallel tool execution.
---

# Asynchronous and Event-Driven Agents

Translated from the [Chinese original](../../skills/async-event-agent/SKILL.md).

## When to Use

- Let agents **act without user input**: scheduled reports, periodic server checks, following up on unanswered emails, or calling during business hours.
- Connect **external event sources**: new email, API callbacks, IM/SMS, GitHub PR changes, payment-failure alerts.
- Support **interruptions during execution**: "Stop," "Change the budget," or "Check the weather first" during a long task.
- **Manage concurrent tasks**: run scripts in parallel, inspect whichever finishes first, cancel insufficient progress.
- Upgrade turn-based ReAct into a long-running **event-loop agent**.
- Evaluate native asynchronous model/interface support or build a compatibility layer where absent.
- Tasks span multiple turns between initiation and completion (phone calls, long approval workflows).
- Communicate asynchronously with a human-like presence (messages, images, files, urgency-based push reminders), without requiring users to open a particular conversation.

## Core Principles

- **Turns are an interaction convention, not a property of the environment.** The world does not wait for model reasoning; tasks often span multiple turns from initiation to completion.
- **Use an event loop, not polling.** Each iteration takes events from an input queue → appends them to the trajectory → calls the LLM once → executes its chosen tools → waits for the next batch. Do not repeatedly ask whether new messages exist.
- **Proactive service requires push delivery to the agent.** Of hooks (framework lifecycle events), cron (scheduling), and heartbeat (wake every N minutes), only the last two initiate autonomous activity, and both are **time-driven**. Third-party sources outside built-in channels (email, external callbacks) need real-time push; otherwise detection waits until the next cycle, an unacceptable delay for verification-code waits or three-party calls.
- **Safe points determine everything.** With synchronous interfaces, events are consumed only at loop boundaries (reasoning ends or a tool returns). Cancellation for urgent events effectively creates an earlier safe point; cancellation points must allow safe tool/reasoning termination.
- **Cancel ≠ undo.** A cancellation signal stops the executor but does not roll back completed actions. Represent unfinished results with explicit placeholders; never fabricate success.
- **Choose three strategies by urgency**:
  - **Canceling** (urgent: `user.interrupt`, `supervisor.instruction`, `agent.interrupt`, emergency alerts): stop the current operation → drain the queue → append all events, including the urgent one, to the trajectory together → immediately call the LLM again.
  - **Queued** (routine: `user.input`, `tool.result`, `timer.trigger`): enqueue without interruption; append **in batches** at the next safe point to reduce round trips.
  - **Parallel** (independent, lightweight queries requiring quick replies, such as "How is the weather today?"): run a separate reasoning session, append results to the main trajectory, and **explicitly mark them as parallel** to prevent confusion.
  - Hardcoded rules have limits; use a lightweight classification LLM as an event router.
- **Model all input as structured events**: source (who), channel (how), content (what/urgency), context (related task). This is a prerequisite for preventing tool results from being mistaken for user instructions and for preventing prompt injection. Unified modeling of heterogeneous triggers (webhooks, timers, email, database changes, file watchers) enables consistent handling.
- **Put asynchronous semantics in tool interfaces**: separate initiation from completion. `initiate_phone_call` immediately returns task ID/initial state, with progress through `phone_call_connected` / `phone_call_ended` events. Names/descriptions should convey this: "Returns an ID immediately; continue other work and receive a separate completion notification."
- **Choose among three routes by model capability**: native async (suspended tools, continued reasoning, mid-turn steering) > synchronous compatibility (placeholders + queues + new requests) > fully serial fallback. Using `asyncio` does not make the model asynchronous; the application still manages sources, tool lifecycles, and result attribution.
- **Reliable handling is harder than reception**: check whether delayed results reach the right task without fabrication when absent; whether the original task resumes after updates and "change plan" is distinguished from "stop"; whether multiple updates are followed together rather than only the latest.
- **Judge priority dynamically, not by static order**: like a secretary, decide what is urgent and whether to pause/switch midway. Long asynchronous tasks must not block user interaction, and interrupted tasks should resume naturally.
- **Introduce placeholders only upon interruption**: normally the LLM sees a standard synchronous trajectory. Only interruption inserts a "tool executing in background" placeholder result to repair call-result pairing; actual results later arrive as events carrying source and task ID.

## Practical Patterns

1. **Build the skeleton**: inbox queue → urgency-routing dispatcher → single-threaded event loop → trajectory store. See `chapter6/async-agent/`, `runtime.py::AgentRuntime` (`_dispatcher` routes urgency, `_handle_interrupt` cancels at safe points, `run_llm_turn` advances the trajectory).
2. **Connect event sources**: three trigger-tool types—timers (one-shot + repeating; repeating timers accommodate query-only external services), background-task monitoring (new output or keywords rather than repeatedly watching command lines and wasting tokens), and external event channels (real-time push). Define filters and sufficiently contextual payloads per source to avoid irrelevant-event compute waste.
3. **Define the urgency classifier**: enumerate event types/default strategies, then cover semantic boundaries with a lightweight LLM ("Stop immediately" → cancel; "Send the report in Chinese" → queue; "How is the weather?" → parallel).
4. **Five synchronous-compatibility rules**: persist assistant messages/call items immediately; record actual tool results only when tools return; if interrupted during execution, create placeholder results before new events and a new request; without steering, cancel unfinished generation and retain confirmed messages only; queue/batch all noninterrupting events.
5. **Integrate multiple updates**: explicitly label each unprocessed event (`[Unprocessed event 2/4] ...`), append a summary, and require responses to all of them to avoid last-event-only attention.
6. **Observe concurrent-task state**: every asynchronous call needs a task ID, progress queries, and cancellation by ID. Inject completion events with actual results (exit codes, artifact hashes), not merely "Done."
6. **Reach the user**: communication tools support asynchronous messages, read/unread state, and channels (IM/SMS/email/phone/push). Choose by urgency, user state, content, and preferences. Notifications also re-engage users after long tasks and support habits around recurring tasks.
7. **Identity and environment**: give agents separate virtual identities (dedicated accounts/storage/compute) in isolated VMs/containers or virtual phones. Pass **file paths**, not contents, through shared volumes (such as `/workspace/shared`) to avoid filling context. For required real-user-account login, use HITL (VNC/RDP for users to log in themselves, reusing sessions while tokens remain valid).
8. **Three acceptance layers**: timely event arrival → policy execution (cancellation really terminates subprocesses; complete batches) → correct model use (result attribution, no fabrication before completion, resumption of original tasks). Add state checkpoints to verify post-interruption recovery.

## Common Pitfalls

- **Treating "started" as "completed"**: placeholders mislead the model into using unavailable results. Require explicit states/result checks and evaluate fabrication of data not yet received.
- **Divided attention**: responding only to the final event silently drops earlier requirements.
- **Polling disguised as async**: checking every few seconds is slow and token-expensive; use push or repeating timers + incremental monitoring.
- **Plaintext concatenation of hidden reasoning**: preserve provider-managed reasoning via its continuation protocol; do not invent partial reasoning and feed it back.
- **Sending cancellation without checking**: an executor that ignores cancellation (like failing to check `ctx.Done()`) keeps running after "Stop."
- **Equal priority for all events**: urgent alerts and routine input in one queue either fail to stop urgent work or interrupt the main task over trivia.
- **Directly hosting personal user accounts**: errors/compromise expose the entire digital identity; separate identities + isolation improve safety/auditability.
- **Ignoring antibot mechanisms**: virtual environments on datacenter IPs are easily detected and need residential proxies; platform-wide ecosystem bans may also occur.
- **Inferring training causes from one failure**: undisclosed training cannot be proven; inspect system-level state, sources, and execution feedback first.
- **Losing relationships across conversation threads**: third-party messages' emotional effects, user roles across threads, and when to synthesize advice require context beyond simple routing.

## Companion Code

- `chapter6/async-agent/` — Experiment 6-2: Flux event-driven asynchronous framework, parallel tools, interruption/cancellation, checkpoint persistence, and a zero-dependency offline demo.
- `chapter6/agent-with-event-trigger/` — Experiment 6-1: FastAPI event-driven agent + 42 MCP tools, demonstrating the minimal "event arrives → agent handles → result" loop.
- `chapter6/async-agent/` provides both offline acceptance demos (parallel versus serial wall time, resume after cancellation, checkpoint persistence/restoration) and LLM scenario reproductions. Verify the runtime offline before using real models.
- `chapter6/astra-async-steering/` — Experiment 6-3: GPT-6 Astra native asynchronous calls/mid-turn steering, with five comparisons (`sync` / `async` / `steer_reasoning` / `async_steer` / `unsupported_steer`) and real API run records.
- Speech (finer-timescale interruptibility): `chapter6/end-to-end-speech/`, `chapter6/streaming-speech/`, `chapter6/controllable-tts/`, `chapter6/live-audio/`, `chapter6/phone-agent/`.
- Robotics (millisecond continuous observation/action chunking): `chapter6/gemini-xlerobot-navigation/`, `chapter6/rgb-sim2real-grasping/`, `chapter6/xlerobot-teleoperation/`.

## Further Reading

- `book/chapter6.md`: "Expanding Modalities and Trigger Timing."
- `book/chapter6.md`: "Asynchrony and Event-Driven Agents: When the World Comes Knocking."
- Related design document: `chapter6/async-agent/agent_framework_design.md` (Flux runtime design).
