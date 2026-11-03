---
name: agent-evolution
description: Use when enabling agents to keep learning from execution experience or building self-evolution loops—extracting learning signals through three-layer trajectory verification; choosing experience knowledge bases, prompts/skills, programs/harnesses, or model parameters; automating prompt optimization; compiling trajectories into programs; gating self-modification; sleep-time learning; and long-term layered evaluation. Triggers include continuous evolution, self-evolution, experiential learning, trajectory verifiers, self-modification, and evolution loops.
---

# Continuous Agent Evolution

Translated from the [Chinese original](../../skills/agent-evolution/SKILL.md).

## When to Use

- A deployed agent repeatedly makes similar mistakes and needs durable capability from execution experience.
- Build trajectory verifiers/evaluators that identify what failed, why, and where the evidence is.
- Decide whether a defect's lessons belong in a knowledge base, prompt/skill, program, or parameters.
- Perform targeted evolution such as automatic prompt optimization, extracting rules from failed traces, or turning browser traces into workflows.
- Design verified releases for agents modifying their own harnesses, tools, or retry strategies.
- Build online-execution/offline-evolution loops, or organize, retire, and roll back memories/skills.
- Evaluate whether a system truly evolves, rather than merely storing or appending feedback.

## Core Principles

- **Saving experiences is not learning from them.** Learning happens through active evaluation, comparison, generalization, and verification, not when logs hit disk. A hundred traces in a vector store are retrievable cases, not automatic cross-case comparison.
- **Evaluate before updating.** Without knowing whether the task finished and which step caused success/failure, reflection is guesswork. Evaluate outcomes (was it done?), processes (was it done permissibly?), and quality (was the interaction comfortable?).
- **Three-layer verification, bottom first.** Outcome verifiers read environmental truth—tests, databases, tool results—and deserve first priority over self-reports. Process verifiers use code against policies, permissions, and action sequences. Only quality needs LLM rubrics. Correct outcomes do not imply correct processes: deleting failing tests also makes tests pass.
- **Learning signals require four elements**: success/partial success/failure; per-dimension conclusions; evidence locations for each conclusion (conversation turn/tool call); and failure-type labels. A single score cannot guide updates. Abstain from scoring insufficient evidence and exclude low-confidence learning cases.
- **Choose storage by how the capability is represented** (Table 9-2):
  - Experience knowledge base—facts, experiential patterns, exceptions, and sources; fast updates and traceability, but dependent on retrieval and correct application.
  - Prompt/skill—judgment principles, contexts, and exceptions expressible in natural language; explainable and scopeable, but prone to growth, conflicts, and being ignored.
  - Program/harness—deterministically parseable, executable, verifiable behavior and high-risk hard constraints; testable, stable, cheap at runtime, but costly to develop and maintain.
  - Model parameters—high-dimensional perception, generation style, and implicit strategies; strong generalization and low inference overhead, but expensive updates and regression checks.
- **One capability can span several representations**: facts in knowledge bases, exception-handling principles in skills, unavoidable permission gates in code, high-dimensional recognition in parameters. Routing only proposes an update; it does not qualify it for release.
- **Experience knowledge versus domain knowledge**: Chapter 3's domain knowledge describes users and the world, such as “this airline requires special-meal orders 24 hours ahead.” Experience knowledge derives “under these conditions, act this way” from actions and outcomes, such as “check special-meal deadlines before booking.” They share storage and retrieval techniques but differ in provenance and verification goals.
- **Every change needs a minimal diff, rollback, and independent verification.** Candidate versions must improve the triggering failure's boundary set without regressing normally working retention tasks. Verifiers must be independent of extractors.
- **Separate online execution from offline evolution.** Online loops complete tasks and append immutable evidence without directly rewriting the production agent. Offline loops aggregate traces, propose changes, and release after gates. Versioned experience stores and datasets connect the loops. A lucky success or malicious input must not immediately rewrite durable capabilities.
- **Three safety boundaries**: separate evidence from instructions (web/tool outputs and their LLM summaries are untrusted evidence; summarization does not sanitize them); separate candidates from production capabilities (new artifacts enter a staging area that cannot serve real traffic, then pass sandbox, permission, supply-chain scanning, and regression gates); prohibit self-modification of safety mechanisms (agents cannot edit verifiers, tests, release gates, audit logs, or stable backups, which would let lower thresholds disguise regressions as progress).
- **Ability to update a harness differs from ability to benefit from it.** The former creates useful changes from traces; the latter means task agents activate and follow them in the right contexts. End-to-end scores cannot isolate updater quality; evaluate layers separately.

## Practical Patterns

### 1. Trajectory Verifiers: Sources of Learning Signals

- Three layers: outcomes read final state; processes check business rules, privacy, factual grounding, and promise–action consistency (did claimed operations actually occur?); quality uses per-item rubric scores with trace evidence.
- Return structured diagnostics, not totals: per-dimension conclusions + evidence turns + confidence, plus failure-type labels.
- Exclude low-confidence cases; embedding them as facts is more dangerous than discarding them.

### 2. Five-Step Experience Extraction Pipeline

1. Save immutable raw traces and environmental outcomes for audit.
2. Produce structured run analyses: task type, required capabilities, observed strategies, errors, exceptions.
3. Group comparable runs by task family; build evidence tables listing traces supporting and contradicting each draft lesson.
4. Promote lessons to formal documents only after support thresholds are met.
5. Test transfer on new tasks not used for extraction.

Formal experience documents state applicability, recommended strategies, prohibited practices, exceptions, evidence sources, and last verification time. Do not retell a full individual run. Transferable value comes from comparison: what success did, what failure lacked, and which versions/prerequisites invalidate a strategy. Reflexion-style natural-language reflections can help draft lessons, but are not themselves evidence.

### 3. Turning Experience into Prompts/Skills

- Trigger: multiple similar traces repeatedly expose one strategy error that can be described clearly in language.
- Form: source-backed minimal diffs (`old_str → new_str`), rather than rewriting the entire prompt every round, which loses details and collapses interacting conditions into overly abstract principles. ACE provides a reference: maintain context as stable-ID entries, with incremental updates and deterministic merging/deduplication.
- Release gates: nonempty patch, traceable sources, no retention regression, and real boundary improvement. Only passing changes enter canary release (`release_to_canary`); reject others. Write candidate patches to working files, not production prompts.
- Skill/harness boundary: skills interpret context, ask questions, and organize specifications. Harnesses veto high-risk writes without confirmation. The veto mechanism does not choose the solution for the model.
- Extraction inputs determine generalization quality: failure summaries and error text yield lessons; adding separate agent/user tool inventories allows responsibility boundaries to be inferred.

### 4. Turning Experience into Programs

- Six-stage browser-workflow lifecycle: capture traces (action arguments, URLs, element-location evidence; locators only help find elements again, not prove completion) → parameterize (replace literals with template variables) → define state checks (before/after actions and final-state checks that read real pages or backend state) → independently validate replay (reset sandbox/test sites to an independent initial state and replay fully; publish only if every predicate passes) → match and replay (retrieve by intent from a capability library and execute directly without stepwise LLM calls) → invalidate and relearn (predicate failures or schema changes fall back to full-agent mode).
- Self-modification follows software release procedures, not in-place replacement of a running process: isolated worktree from stable version → minimal coding-agent patch → static checks, unit tests, security scans, failure replay, old-task regression → canary-capable new version. Each request is a falsifiable change contract: failure evidence, inferred root cause, responsible component, proposed change, intended behavioral fix, potentially damaged behavior, and cases verifying both.
- Proposal generators need not only failures but also success constraints (properties to preserve) and prior rejections (avoiding reworded resubmissions). Together these bound the solution space.

### 5. Writing Experience into Parameters

- Decide by representation, not whether tasks are stable over time: can external symbols express the capability adequately? Use LoRA/continued fine-tuning for domain shifts, periodic preference training for rapidly changing styles.
- Source data from evaluated production traces: high-quality demonstrations for SFT, explicit preferences for paired data, and interactions with reliable environmental rewards for RL.

### 6. Dual Loops and Sleep-Time Learning

- Five offline consolidation steps: trigger (time/trace volume/capacity/error-frequency thresholds, with no high-priority online tasks) → orient (read formal knowledge, skill catalogs, and versions; understand immutable boundaries) → collect and consolidate (merge duplicates, mark conflicts/applicability, prefer local patches) → verify and approve (transfer/retention/safety sets, human approval for high-risk changes) → prune and index (expire/archive unused or refuted knowledge while preserving provenance and rollback versions).
- Regularly combat context rot: merge duplicate experience, move local rules from global prompts to domain skills, revalidate long-unused tools, delete refuted knowledge, and retrain LoRA from the original base.

### 7. Layered Evaluation of Evolution

- Metrics (Table 9-3): proposal effectiveness (acceptance and gains under independent verification), artifact activation (loaded in correct contexts), compliance success (action sequences and process verification), and gains on held-out tasks (success/quality/cost for tasks not used in evolution).
- At least five long-term dimensions: regressions from conflicting experience; generalization outside test scenarios; token efficiency; safety drift in rules/privacy/refusal boundaries; engineering quality including maintenance complexity, backward compatibility, and debugging burden.
- Open-ended research/strategy tasks have delayed feedback and nonunique answers, requiring different evidence structures: separate claims from evidence with auditable sources, keep negative results and stopping reasons, preserve search diversity rather than betting everything on the current top score, and involve humans at higher levels to define problems, review evaluation criteria, and decide when to stop.

## Common Pitfalls

- Treating user satisfaction or one aggregate score as a learning signal: satisfaction can rise with rule violations, requiring guardrail metrics.
- Encoding low-confidence conclusions as facts, or placing LLM summaries directly into skills as supposedly sanitized instructions.
- **Models may treat observed behavior as prescribed behavior**: frequent human escalation in traces can yield a “transfer after three failures” rule, even when escalation necessarily fails in that environment—encoding failure into policy. Independently verify extracted artifacts.
- Rewriting entire prompts/memories each round, losing details over successive revisions.
- Equating executed actions with task completion: replay without final-state checks only repeats mistakes faster.
- Publishing unverified versions directly, or allowing proposals to modify their own release gates. Authors cannot certify their own safety gates.
- Fixing only the current failure while ignoring retention regressions, generalization, token costs, and engineering quality.
- Appending without retirement: unbounded knowledge causes retrieval errors, conflicts, and catastrophic forgetting that offset learning gains.
- Judging updater quality from end-to-end scores while ignoring activation and compliance failures.
- Accepted proposal does not equal improved downstream capability: acceptance proves the update workflow works; still run on/off ablations under identical tasks and models.

## Companion Code

- `chapter9/trajectory-verifier/` — Experiment 9-1: three-layer trajectory verifier, comparing totals with evidence-backed multidimensional diagnostics.
- `chapter9/tau2-escalation-experience/` — Experiment 9-2: extracting escalation/tool-use rules from τ²-bench telecom failures, with three-arm comparison.
- `chapter9/prompt-auto-optimization/` — Experiment 9-3: airline-service failures → three-dimensional diagnosis → minimal prompt diff → release gates.
- `chapter9/ai-style-skill/` — Supplementary example: continuously extract a writing skill from user before/after corrections of “AI-like” prose, with an LLM judge and gold-set calibration.
- `chapter9/browser-use-rpa/` — Experiment 9-5: compile browser traces into candidate workflows with state predicates; independent replay precedes capability-library admission.
- `chapter9/self-modifying-agent/` — Experiment 9-6: failure-triggered self-modification of retry/circuit-breaker code, proposals verified in Docker sandboxes.
- `chapter9/harness-safety-gate/` — Experiment 9-7: user-feedback-triggered confirmation gates for high-risk operations, AST scans + isolated replay.
- `chapter9/hermes-self-evolution/` — Experiment 9-8: Hermes reads the book, autonomously chooses a topic, modifies itself, and receives independent reviewer scrutiny.
- `chapter9/self-evolution-eval/` — Experiment 9-9: four-stage longitudinal evaluation (learning/transfer/rule changes/retention), distinguishing storage, append-only memory, and memory with retirement.
- `chapter9/self-evolving-tools/` — Alita-style supplementary example: agents discover tools from scratch, sandbox-test, package, and store them for reuse.

## Further Reading

- `book/chapter9.md`, “Obtaining Learning Signals from Execution Traces”—three verification layers and four learning-signal elements.
- `book/chapter9.md`, “Four Methods for Continuous Agent Evolution”—boundaries and processes for knowledge, instructions, programs, and parameters.
- `book/chapter9.md`, “Building a Long-Running Continuous Evolution Loop”—dual loops, sleep-time learning, layered evaluation, and safety boundaries.
