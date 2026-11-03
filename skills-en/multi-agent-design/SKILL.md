---
name: multi-agent-design
description: Use when designing or reviewing multi-agent systems or deciding between one and multiple agents—choosing peer/manager/decentralized topologies, shared or isolated contexts, inter-agent communication and shared filesystems, and diagnosing concurrency conflicts, error cascades, homogeneous convergence, disputes, or runaway loops. Covers architecture selection, the new-information criterion for outperforming a single agent, four control-plane primitives, and four virtual-filesystem regions.
---

# Multi-Agent Collaboration Design

Translated from the [Chinese original](../../skills/multi-agent-design/SKILL.md).

## When to Use
- Build or review a multi-agent architecture: how many agents, and who directs whom?
- Decide with concrete criteria whether splitting a task across agents really helps.
- Choose shared or independent contexts between agents.
- Design communication through tool arguments/shared filesystems/message buses, and filesystem layouts.
- Decide whether role changes should replace system prompts or load skills.
- Diagnose overwritten files, distorted conclusions, agents independently duplicating work, disputes, or exploding subagent counts.

## Core Principles
- **The new-information criterion is the only substantive criterion.** Multiple agents outperform one if and only if collaboration introduces information unavailable to a single agent during generation. The same model rereading its output, or agents debating identical text, adds no information; at equal compute, a single agent matches or beats them. Reviews using test execution, rendered screenshots, or external tool output improve performance significantly. Research finding multi-agent systems ineffective and engineering experience finding them better are compatible: the former compares discussion of the same context; the latter includes external feedback loops.
- **Choose two dimensions before implementation.** First, shared versus isolated contexts (inherited versus process-isolated); second, peer/manager/decentralized topology. Together they define the architecture.
- **Benefits must cover costs.** Parallel exploration and repeated iteration consume several times, or even an order of magnitude, more tokens. Without large enough benefits, a tuned single agent is more economical.
- **More steps are not always better.** More steps/tool calls do not guarantee improvements; without budget awareness agents quickly saturate. Managers should allocate budgets by subtask complexity and guide agents to plan, implement, test, then improve.
- **Give the strongest model to the manager.** Plan-and-Act's empirical conclusion is that a weak planner is the critical bottleneck: bad plans give every subsequent action false premises. Do not distribute resources evenly across agents.
- **Agent failures are Byzantine.** They rarely stop outright, instead continuing with plausible but wrong conclusions that do not label themselves as errors. Cross-validation and majority voting are therefore necessary, not optional optimizations.
- **Prefer context isolation.** Keep only task descriptions, plans, call records, and file indexes in manager context. Store full artifacts in the filesystem and pass lightweight path strings instead of loading their contents into context.
- **Handoffs do not expose private trajectories.** Peer transfers should convey only explicit task packages and artifact references.

## Practical Patterns

### 1. Topology Decision Tree
| Task characteristics | Choice |
|---|---|
| 2–3 roles need mutual feedback and repeated quality improvement | Peer collaboration: drafter/critic, proposer/reviewer |
| Many subtasks need dynamic scheduling and have complex dependencies | Manager plans; subagents execute |
| Equal-responsibility roles decide autonomously whom to contact, or a manager cannot be a single point of failure | Decentralized handoffs / message-pool subscriptions |
| Open search spaces need breadth and diverse discovery paths | Parallel agents, trading higher token budgets for coverage |

Peer collaboration is simplest to implement: define the two roles, communication, and iteration termination conditions.

### 2. Shared Versus Isolated Context
- **Shared context**: every stage is a separate agent with its own system prompt and tools, but inherits the full preceding trajectory. Information is preserved, but context grows rapidly and history can distract the current agent.
- **Isolated context**: every agent has its own context and trajectory, without access to others' reasoning. This improves modularity, isolation, scalability, and real concurrency, at the cost of synchronization and debugging difficulty. Interface specifications and data formats become essential.
- **Only three communication mechanisms, all classical IPC**: tool-call arguments (synchronous messages), shared filesystems (shared memory), and message buses (asynchronous messages). Use point-to-point for few agents and fixed topology; use a bus for many asynchronously parallel agents. Point-to-point connections grow quadratically and require both parties online together.

### 3. Role Switching: Replace System Prompts or Load Skills?
| | `transfer_to_agent` / system-prompt replacement | Skill |
|---|---|---|
| Tool visibility | Only current-role tools exposed | Usually the full fixed toolset |
| Prefix caching | Each switch changes the request prefix, invalidating cache from the change onward | Static prefix remains; skill content appends to trajectory, retaining prefix reuse |
| Constraint strength | Strong: out-of-role tools are absent from schemas | Weak: skills are behavioral instructions; hard permissions still need harness gates |

Criterion: differences mainly in **knowledge, workflow, or writing style** → skills. Differences in **permissions, tool isolation, compliance boundaries, or runtime-enforced prohibitions** → independent agents or `transfer_to_agent`, with code restrictions in the harness.

### 4. Four Virtual-Filesystem Regions: Layout Checklist
| Region | Visibility | Lifecycle | Access | Concurrency control |
|---|---|---|---|---|
| Agent-private scratchpad | That agent only | Destroyed with instance | Read/write | Unnecessary |
| Shared workspace | All collaborating agents + user | Persists for task; needs durable storage | Read/write | Required: optimistic locks / worktrees |
| External mounts | Depends on external authorization | External source controls it | Mostly read-only; writes require care | External source handles it |
| Built-in system resources, such as skills | All agents | Stable across sessions | Read-only | Unnecessary |

Isolated scratchpads prevent temporary-file overwrites and keep manager context small: subagents submit only final artifacts to shared space. External mounts need explicit permission handling, weak-consistency handling, and read-only access on demand. Mixing regions that should be isolated causes many concurrency conflicts and information leaks.

### 5. Four Control-Plane Primitives
- **Messaging**: point-to-point or bus messages need structured envelopes: sender ID, target, type (`task_assigned`/`status_update`/`result`/`terminate`), and JSON payload. A common envelope makes chains traceable, essential for multi-agent debugging.
- **Status queries**: do not use polling `get_status`. Prefer messages asking about progress or a shared convention such as `progress.md`, updated after each completed item. No modification for N minutes indicates a stall and triggers timeout recovery. Persistent JSONL traces help inspect the whole run but are poor primary communication: tens of thousands of tokens still need distillation.
- **Termination**: prefer graceful `terminate` signals acknowledged after resource cleanup at safe points, with forced termination as fallback. Cascade termination down creation relationships to prevent orphan agents. Deliberately long-lived background agents should start a new lifecycle tree.
- **Resources and scheduling**: set step/token budgets at launch and stop at limits; strong models for difficult tasks, cheap models for mechanical work; cap concurrency against API exhaustion; permit preemption for high-priority arrivals.

### 6. Two Manager Coordination Forms
- **Sequential**: call specialist agents in order, with linear control flow for clear dependencies.
- **Parallel**: agents work simultaneously, with a message bus for real-time monitoring; the manager makes global decisions on success/failure.
- **Settlement point**: define “first **verified** success,” not first claimed success. Validate results with hidden checks, then claim the winner using idempotent locking/transactions and cancel other workers. Two near-simultaneous success events must not trigger two aggregations.
- Advanced: managers can write agent workflows as code for deterministic runtimes, rather than remaining in the loop at every step.

### 7. Minimal Decentralized Handoff Protocol
```python
handoff = {task_id, sender, recipient, goal, constraints,
           accepted_facts, artifact_refs, remaining_budget, visited_agents}
```
Recipients read packages and references and retrieve evidence as needed. The runtime preserves budgets, visitation chains, and cycle detection; agents cannot delete them. Reject recipients already in `visited_agents` to prevent A→B→A loops. Stop and report when budgets are exhausted.

### 8. Real Diversity in Peer Collaboration
Multiple instances do not imply multiple approaches. Similar models, contexts, and scaffolds make similar choices. Deliberately vary **models, contexts, tools, visible evidence, or responsibilities**, and have agents form independent judgments before aggregating.

## Common Pitfalls
- **Shared-filesystem conflicts**: simple last-writer-wins overwrites need optimistic locking—record versions when reading, check on write, reread and retry on conflict. Cross-file semantic conflicts are subtler: agent A renumbers images while B cites old numbers, causing no file conflict but incorrect logic. Independent Git branches/worktrees are standard for concurrent code changes, deferring conflicts to merge points.
- **Error cascades**: semantic handoffs re-encode information lossily at each retelling. Break the chain through independent cross-validation using only raw evidence and final conclusions, without preceding reasoning.
- **Homogeneous convergence**: common-cause failures. Reviews from identical models and similar contexts are not automatically independent evidence. Vary models/context/data sources and use namespaces, quotas, and rate limits to prevent identical decisions overwhelming shared resources simultaneously.
- **Disputes**: agents with conflicting goals interpret each other's actions as deliberate obstruction. Predefine goal priorities, resource ownership, and permission boundaries in the runtime. If verifiable rules cannot resolve conflicts, pause for human arbitration.
- **Runaway loops**: the opposite of premature termination. Uncontrolled agents spawn thousands of subagents and waste tokens. Give highly autonomous agents separate API keys.
- **Understanding debt and cognitive surrender**: faster delivery makes engineers' understanding lag implementation; relying on delegation can undermine independent review. Reasoning can be outsourced; understanding cannot.

## Companion Code
- `chapter10/multi-role-transfer/` — Controlled comparison of system-prompt replacement and skill loading over the same shared trajectory.
- `chapter10/staged-system-prompt/` — Archived staged system-prompt/toolset switching experiment: requirements→implementation→review loop.
- `chapter10/book-translation/` — Four-agent manager architecture (Glossary/Translation/Proofreading/Manager), verifying manager context does not grow with book length.
- `chapter10/parallel-web-research/` — Dynamic launch of N homogeneous workers, a message bus, and cascading cancellation/resource cleanup after the first verified match.
- `chapter10/autonomous-phone-registration/` — Phone + computer point-to-point collaboration, with the model deciding whether to spawn a phone agent.
- `chapter10/talkact-reproduction/` — Fixed-topology fast/slow dual-agent comparison against a single-model baseline.
- `chapter10/voice-werewolf/` — Multi-agent voice Werewolf: private/public memory isolation and a code-driven read-only judge.
- `chapter10/generative-agents/` — Reproduction of Stanford's 25-character AI town, with controlled reflection and information-diffusion experiments.

## Further Reading
- `book/chapter10.md`, “A Classification Framework for Multi-Agent Collaboration.”
- `book/chapter10.md`, “When Multiple Agents Truly Outperform One.”
- `book/chapter10.md`, “Multi-Agent Collaboration with Shared Context.”
- `book/chapter10.md`, “Multi-Agent Collaboration without Shared Context.”
- `book/chapter10.md`, “Failure Modes of Multi-Agent Collaboration.”
