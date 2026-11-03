---
name: coding-agent-harness
description: Use when building coding agents or agent guardrails—autonomous code reading/writing, testing, and self-correction; adding constraints, verification, and correction; defining execution boundaries and acceptance baselines; assessing task suitability; or improving harness prompts and tools. Covers four harness components, four task quadrants, four design principles, and industry practice.
---

# Coding Agents and Harness Engineering

Translated from the [Chinese original](../../skills/coding-agent-harness/SKILL.md).

## When to Use
- Build a coding agent from scratch: select tools, define workflow, connect test feedback.
- Improve an existing agent's reliability with guardrails, acceptance baselines, execution boundaries, and rollback.
- Assess whether a task suits autonomous agent execution using four quadrants.
- Address an agent that "runs but is unreliable": drifts off course, takes destructive shortcuts, or declares completion early.
- Optimize the harness itself (prompts, tool middleware, self-verification loops) rather than changing models.

## Core Principles

**Agent = Model + Harness; Harness = context management + tool interfaces + constraints + verification + correction.**
The harness is the runtime and governance layer "inside the agent boundary but outside the model": tool definitions, call adapters, and sandbox permissions belong to the harness; files/processes inside the sandbox, external databases, and users belong to the environment. As model capabilities converge, the harness becomes the competitive differentiator.

**Four harness components for coding agents** (turning abstract mechanisms into engineering components):

| Component | Question answered | Implementation |
| --- | --- | --- |
| Acceptance baseline | What counts as done? | Test suites, CI pipelines, code-review standards |
| Execution boundaries | What may or may not be touched? | Module boundaries, dependency rules, permission controls |
| Feedback signals | Automated correctness judgments | Linter output, test results, type-check errors |
| Rollback mechanisms | How do we recover? | Git version control, sandbox isolation, snapshot rollback |

**Four task quadrants (goal clarity × verification automation)**—the harness aims to move tasks toward "clear goals + automatic verification":

- Clear goal + automatic verification = **best region** (fixing a bug with a test case).
- Clear goal + human verification = **throughput-limited** by human review speed.
- Vague goal + automatic verification = **efficiently going off course** (optimizing "code quality" through linter scores).
- Vague goal + human verification = **hard to start** ("Make the UI look better").

Coding naturally lies at the heart of this framework: tests provide acceptance criteria, linters/type checks provide immediate verification, and Git provides versioning/rollback. Coding agents are most mature not because code models are strongest, but because decades of software-engineering infrastructure already form a harness.

**Four transferable design principles**:

1. **Constraints before guidance**—enforce rules in code when possible, rather than recommending them in documents. Linter rules, types, and CI make actions impossible; "Please follow..." in a prompt only advises against them.
2. **Automate verification**—human review is an unscalable bottleneck; tests/checks/monitoring offer far better returns than adding people.
3. **Make feedback fast and structured**—more detailed errors, closer to when they occur, help agents correct more efficiently.
4. **Make rollback reliable**—a safety net enables experimentation; branches, sandboxes, and snapshots make errors reversible.

**Constraints govern actions, not just outcomes.** Acceptance baselines judge results; execution boundaries govern process. Dropping and recreating a database can "fix" a failure while losing data; deleting and rewriting everything can pass compilation while removing the implementation. Agents can find such destructive shortcuts around metrics (everyday reward hacking), so `rm -rf`, production-data deletion, and overwriting unread files need dedicated checks and approval.

**Fault boundaries for parallel calls**: a failed tool's failure propagates only within the same parallel batch (cascading aborts of dependent calls); do not cancel independent calls, propagate it to the parent operation, or abort the whole task. Each tool declares whether it supports concurrency (default: no, a fail-safe default).

**Industry experience**:
- Large-scale code migrations rely on three things: knowledge in the repository itself (what agents cannot see effectively does not exist), constraints encoded in linter/CI rather than documents, and end-to-end automated verification/correction.
- LangChain improved Terminal Bench 2.0 from 52.8% to 66.5% by optimizing only the harness (prompts, tool middleware, self-verification loops), then used agents to analyze failed trajectories and improve it, moving harness engineering from experience-driven to data-driven.
- Anthropic separates two roles: an initialization agent decomposes the task list; an execution agent progresses incrementally and leaves clear handoff artifacts, addressing "trying too much at once" and "claiming completion too early."

## Practical Patterns

**Minimum checklist for a coding agent**:

1. **Tools**: seven basic operations—file read/write/edit, directory glob/ls, content grep, command bash. Use intuitive names, parameter examples, and explicit boundaries (error-proof design).
2. **Workflow**: understand the project (read docs, build a conceptual framework) → design → implement → **immediately write and run tests** → analyze/localize/fix failures until all tests pass. Define completion as "tests passed," not "code written"; skipping tests and reporting done is the most common coding-agent shortcut.
3. **Acceptance baseline**: connect existing tests, linter, type checks, and CI; if absent, add a minimal runnable baseline first.
4. **Execution boundaries**: least privilege by default (fail-safe defaults; capabilities disabled unless explicitly enabled); approve dangerous commands.
5. **Rollback**: create a Git branch or snapshot before every task; execute irreversible actions inside a sandbox.
6. **Feedback loop**: feed linter/test/compiler errors into context in structured form, including line numbers, paths, and error types.
7. **Continuous improvement**: collect failed trajectories and let agents analyze them to inform harness changes.

**Context and environment injection** (the harness context layer): read large files by line ranges and prefix each line with its number. For long command output, retain the head (error context) and tail (error summary), stating that full output is saved to disk. Before each reasoning step, dynamically inject working directory, Git branch, recent commits, and unstaged changes through a status bar (append dynamically; do not hardcode them into the system prompt and break the KV cache). Maintain persistent terminal sessions to retain cd/environment-variable state.

## Common Pitfalls
- Prompt guidance without code constraints—the model may ignore "Please follow..." advice.
- Granting autonomy without acceptance baselines: efficient drift or destructive shortcuts (recreate databases, delete and rewrite everything).
- Reporting "complete" after writing code without running tests.
- Putting the whole repository into context, which is costly and unnecessary.
- Aborting an entire parallel batch or task because one tool fails.
- Changing models without fixing the harness: benchmark gains often come from the harness.
- Hardcoding environment information into the static system prompt.

## Companion Code
- `chapter5/coding-agent/` — production-grade coding agent in pure Python: 16 tools, patch application, test/lint acceptance feedback, and failure paths.
- `chapter1/context/` — context-aware agent and ablations quantifying history/reasoning/tool calls/tool results.
- `chapter1/web-search-agent/` — autonomous ReAct search agent using Kimi Formula API, demonstrating a minimal "model as agent" loop.
- `chapter1/search-codegen/` — Responses API deep-research agent orchestrating hosted web_search + code_interpreter.
- `chapter1/learning-from-experience/` — comparison of LLM in-context learning and traditional RL to understand where agent experience comes from.
- `chapter1/image-gen-workflow/` — workflow versus native-generation routes: when to use a fixed harness and when to grant model autonomy.

## Further Reading
- `book/chapter5.md`: "Coding Agent" → "Harness Engineering in Coding Agents."
- `book/chapter1.md`: "Harness Engineering: Competitive Strength Beyond the Model."
