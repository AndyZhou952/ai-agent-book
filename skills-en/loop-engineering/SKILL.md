---
name: loop-engineering
description: Use when building or repairing agent iteration loops—premature completion claims, stopping halfway, false success, or loops that cannot stop; designing verifiers and termination criteria; and implementing proposer–reviewer agent loops. Covers three premature-termination forms, the verifier as the loop bottleneck, LoopX's persistent control plane, LongHorizon-Harness's MEA loop, and minimal proposer–reviewer invariants (independent evidence and actionable repair conditions).
---

# Loop Engineering and Proposer–Reviewer Systems

Translated from the [Chinese original](../../skills/loop-engineering/SKILL.md).

## When to Use
- Agents stop halfway: claiming completion after coding without tests, or saying both requests are done after completing only one.
- Judge whether completion claims are trustworthy; design verifiers and termination conditions.
- Agents abandon whole tasks after one failure, or mistake an incomplete end-to-end workflow for success.
- Implement proposer–reviewer loops for presentations/web pages, code, video editing, security, or content review.
- Maintain long-running tasks across context refreshes and GUI/CLI interfaces.
- Stop runaway token use with budgets and iteration limits.

## Core Principles
- **Before verification, “complete” is a model's claim, not proof.** Loop engineering turns claims into proof by sustaining a cycle: find the next necessary action, execute, verify, record progress. Humans shift from prompt operators to loop designers.
- **The verifier, not the model, is the bottleneck.** Unreliable verification only lets faster loops label poor output complete faster.
- **Models may propose completion but cannot approve their own completion.** The central checkable invariant: only independently verified results enter persistent progress and consume quota.
- **Review must introduce new information.** Having an agent review its own generation is just asking it to think again. ICLR 2024 research found GPT-4 self-correction without external feedback changed more correct answers into wrong ones. Effective review reads execution feedback (test passes/failures), visual feedback (rendered screenshots), and tool feedback (external verification output).
- **Reviewers must inspect independent evidence**: rendered results, execution results, and external facts, rather than restating proposer explanations.
- **Rejections must be actionable and localized**: page, `issue_type`, severity, and a concrete repair condition, rather than “looks bad.”
- **Human gates block before execution.** Human approvals, wait states, and budget limits should prevent the next iteration, rather than repair consequences afterward.

## Practical Patterns

### 1. Identify the Form of Premature Termination
| Form | Behavior | Root cause |
|---|---|---|
| Incomplete work claimed complete | Claims all work done after partial work: code without tests, untried deployment, or only one of two requests completed | No verification checklist covering all acceptance items |
| Premature abandonment | One route fails, so the whole task is declared impossible: one rejected phone call despite remaining forms/email channels | Alternatives not enumerated; no replanning |
| False success | Believes success without closing the loop: a refund is verbally agreed, but the user still needs an app confirmation, while the agent reports completion | Intermediate acknowledgment mistaken for final state; no end-to-end verification |

All three share one cause: models narrate completion criteria instead of verifiers deciding them.

### 2. Minimal Proposer–Reviewer Loop
```python
candidate = proposer(task, constraints)
evidence = execute_or_render(candidate)       # tests, state, screenshot, facts
review = independent_reviewer(candidate, evidence)

while review.veto and budget_remaining:
    candidate = proposer.repair(candidate, review.findings)
    evidence = execute_or_render(candidate)
    review = independent_reviewer(candidate, evidence)

if review.pass:
    publish(candidate, evidence, review)      # Publish the evidence too
else:
    escalate_or_reject(review)                # Escalate at budget exhaustion; do not lower the bar
```
Three minimal invariants:
1. Reviewers read **independent evidence**—execution results, rendered screenshots, external facts—rather than only paraphrasing proposer explanations.
2. Rejections provide **localized repair conditions**: location, issue type, severity, suggestion.
3. Reviewers **cannot modify tests, evidence collectors, or release gates**, which would reduce independent verification to self-approval.

### 3. Evidence Collection: Channels for New Information
- **Code**: run tests and compilation; return pass/fail and errors.
- **Frontends / PPT / slides**: actually render PNGs for vision-model review. Screenshots reveal layout information unavailable to the proposer while coding: overflow, crowding, image dimensions.
- **Video**: sample key frames for review.
- **Factual content**: verify through external search, interpreters, and data sources.
- Negative control: remove tool verification while retaining only model self-assessment, and most gains disappear.

### 4. LoopX: Move Loops from Chat History into a Persistent Control Plane
```text
LoopX decides → Agent executes → Independent verifier proves → LoopX commits
```
- Goals and boundaries explain why; gates and todos decide what can happen now; evidence and quotas decide whether to continue; handoffs let another iteration or agent resume.
- LoopX manages continuity across iterations rather than replacing the agent runtime.
- Verification failures trigger repairs or replanning. Only independently verified results enter persistent progress and consume quota.

### 5. LongHorizon-Harness: MEA Loops for Long-Horizon Tasks
Reframe long execution as **task-state management**, using Manage–Execute–Audit:
- **Manager**: generate the next **bounded** subtask from the original goal, verified progress, failure evidence, and remaining work.
- **Executor**: change the environment through GUI or CLI in fresh context.
- **Auditor**: inspect real results read-only. Only audited progress enters the next task state; failures remain evidence for recovery and replanning.

The value is separating continuity from ever-growing execution histories. Context can refresh and UI operations can fail, yet the next round resumes from the latest **verified** state. In controlled comparisons with the same model and execution backend and only the outer loop changed, the paper reports WeaveBench PassRate 51.8%→80.7%, OSWorld 2.0 binary completion 2.8%→8.3%, and Terminal-Bench 2.1 69.7%→77.2%. Costs vary: the first two consumed 2.3× total tokens and 3.6× output tokens respectively; the third used 24% fewer. Deployments must handle stale state and bound recovery with iteration, time, and cost budgets.

### 6. Budgets, Termination, and Runaway Prevention
- Set iteration/token/time limits per round; escalate or reject when exhausted, rather than passing by lowering standards.
- Verifiers determine termination, not model self-reports.
- With parallel workers, settle on the “first **verified** success”; claim the winner with an idempotent lock, then cancel others.
- Give highly autonomous agents separate API keys to limit token costs from explosive subagent growth.

### 7. Extending to More Review Scenarios
Apply the same pattern to security review (proposer drafts actions, reviewer checks compliance and risk), content moderation (draft replies, review business rules and wording), and code review (write code, inspect security and best practices). A planner–generator–evaluator three-agent variant first agrees on per-round completion criteria, then has the evaluator operate the real app and submit defect reports.

## Common Pitfalls
- **Self-approval**: generators define acceptance, execute verification, and approve results, making independent verification ceremonial.
- **Review without new information**: rereading one's own output can reduce accuracy; at equal compute, multi-agent debate matches a single agent.
- **Vague rejection**: “looks bad” cannot locate repairs and causes empty iterations.
- **Evidence and candidate out of sync**: reviewers inspect prior-round evidence or self-reports rather than actual execution of the current candidate.
- **Failure to continue and failure to stop are opposite extremes**: absent verification causes false completion; absent budgets and iteration caps causes runaway loops. Both need explicit termination conditions and quotas.
- **Understanding debt**: faster loop delivery makes human understanding lag implementation. Thinking can be outsourced; understanding cannot. Reviewers and gates keep humans able to understand and direct the system.

## Companion Code
- `chapter5/paper-to-ppt/` — A proposer writes Slidev; a reviewer really renders every page to PNG and uses a vision LLM for structured feedback (`page`/`issue_type`/`severity`/`suggestion` + overall score and pass). Dual-agent peak context is 24,186 versus 92,601 tokens for single-agent self-review.
- `chapter5/video-edit/` — Two-stage vision localization (10s coarse scan → 1s fine scan) packaged as a subagent; proposer creates Blender bpy / ffmpeg editing scripts, reviewer samples key frames for iterative review.

## Further Reading
- `book/chapter10.md`, “Peer Collaboration” → “Loop Engineering.”
- `book/chapter10.md`, “Peer Collaboration” → “The Proposer–Reviewer Paradigm.”
