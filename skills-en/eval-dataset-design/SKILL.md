---
name: eval-dataset-design
description: Use when designing agent evaluation datasets or individual evaluation tasks—dissecting the four parts of a task, designing verifiers and acceptance criteria (including protection against superficial fixes), defining difficulty tiers and trap tasks, preventing training/evaluation leakage with parameterized instances and canaries, maintaining dataset quality, and choosing among public benchmarks, custom business datasets, and production trace feedback. Covers transferable design techniques from τ²-bench, SWE-bench, AndroidWorld, OSWorld, Terminal-Bench, and GAIA.
---

# Evaluation Dataset Design

Translated from the [Chinese original](../../skills/eval-dataset-design/SKILL.md).

## When to Use
- Design an evaluation dataset from scratch, or review whether existing task definitions hold up.
- Write an individual agent evaluation task: ticket, simulated-user specification, initial state, and acceptance criteria.
- Design a verifier: should it assert final state, actions, or information that must be communicated to the user?
- Improve the signal-to-noise ratio when models pass tasks by guessing answers.
- Prevent and detect leakage when an evaluation dataset may have contaminated training data.
- Decide where custom evaluation tasks should come from: public benchmarks, business datasets, or production failures.
- Maintain a deployed evaluation dataset by fixing environments, descriptions, verification logic, and initial states.

## Core Principles
- **One evaluation task has four parts**: the agent's ticket, the simulator's behavioral specification, initial resets of both sides' state, and success criteria. Without any one of them, the task cannot be run repeatedly.
- **Model the limits of the user's knowledge as separate fields rather than relying on prompt restrictions.** The agent cannot infer information the user does not know; it must obtain information through questions and guidance. Progressive disclosure requires separating `known_info` from `task_instructions`, rather than merely saying “do not reveal everything at once.”
- **An ungrounded user simulator reduces evaluation to two models confirming each other.** Every simulated-user answer about environmental state must be grounded in tool results, without fabricated outputs. Otherwise, a little prompting from the agent makes the user confirm that “the problem is solved.”
- **Verifiers must check facts a machine can independently verify, rather than trusting the agent's self-report.** An agent can easily write an “everything is complete” report without doing anything.
- **Strictly separate training and evaluation data.** Use at least one of these measures: answers cannot be retrieved directly from the internet; task instances are generated from parameters; embedded canaries make leakage detectable.
- **Acceptance criteria need verifiable boundaries to block superficial fixes.** “The network is restored” is unverifiable; “only an excellent speed-test rating counts as resolved; poor/fair/good are unacceptable” prevents masking symptoms without fixing the root cause.
- **High-quality evaluation datasets are refined through repairs, not simply written.** Current mainstream benchmarks resulted from repeated repairs after their initial versions exposed problems. Budget for both manual rejection before release and ongoing fixes afterward.
- **Complete every task yourself.** Then ask: does its description admit multiple reasonable interpretations, and which does the verifier accept? What is the cheapest way to fake success, and can the verifier block it?
- **Difficulty tiers keep an evaluation dataset useful over time**, with failures at each tier pointing toward different improvements: basic failures indicate tool-use problems, intermediate failures indicate multistep planning problems, and the highest tier indicates long-sequence reasoning problems.

## Practical Patterns

### 1. Task Anatomy: A Four-Part Field Checklist
Use a real τ²-bench telecom task as a template for checking custom tasks:
- **ticket (the agent's work request)**: include only what the user would volunteer. A real user may initially say only “I cannot get online.” Clarifying the request enough to act on it is itself a required agent capability.
- **user_scenario (the simulator's behavioral specification)**: `known_info` defines what the user knows, such as name, phone number, and current country; it excludes the fault's cause. `task_instructions` specifies how information is disclosed and includes three constraints: emotional behavior (show dissatisfaction after the first failed fix), acceptance criteria (only excellent counts as resolved), and factual grounding (answers about device state must use tool results).
- **initial_state (initial reset)**: `initialization_actions` resets both sides to the same starting point, including user-side state (airplane mode, roaming toggle) and agent-side state (carrier configuration).
- **evaluation_criteria (success criteria)**: combine four checkable dimensions as needed, plus an aggregation rule.

### 2. Four Verification Dimensions and Aggregation Rules
- `env_assertions` checks **final state**: mobile data works, speed reaches 200 Mbps, and its rating is excellent.
- `actions` checks **whether key actions occurred**: which tools were called by which side.
- `communicate_info` / `nl_assertions` checks **whether required information was communicated to the user**, beyond environmental state alone.
- `reward_basis` is the aggregation rule. A binary reward based only on final state trades process detail for one number comparable across models: even a full-score trajectory can violate the “one tool call at a time” policy without detection. Production evaluation needs more: identifying what went wrong as well as whether the outcome was correct.

### 3. Three Verifier Design Patterns
- **Two-part verification (SWE-bench Verified)**: FAIL_TO_PASS (fails before the fix and passes afterward, proving the problem is resolved) plus PASS_TO_PASS (passes both before and after, proving no new defects). Checking only the former lets an agent cheat by deleting or changing assertions that obstruct passing; checking only the latter does not test the fix. Also exclude inherently unstable flaky tests.
- **Deep state inspection (OSWorld style)**: 134 independent evaluation functions with full system access inspect filesystem structure, process state, network connections, and application internals. Database tasks connect to the database to verify that SQL actually executed; browser tasks inspect DOM, cookies, and localStorage and send backend verification requests. This catches apparent completion with incorrect substance.
- **Unforgeable execution (Terminal-Bench style)**: success requires a real build and execution—for example, building a kernel from source and showing a custom printk in QEMU boot logs. The agent must carry out the full workflow rather than forge output.

### 4. Difficulty Tiers and Trap Tasks
- Label difficulty tiers and use them diagnostically. GAIA has 466 questions across three levels: Level 1 needs only one or two tools (humans 93.9%, early models 30.3%); Level 2 needs multistep reasoning; Level 3 needs complex combinations. Level 1 failures indicate basic tool use, Level 2 failures indicate multistep planning and information integration, and Level 3 failures indicate long-sequence reasoning and complexity management. Each calls for different improvements.
- **Trap tasks** test judgment under pressure and misleading information: a user claims “customer service already approved cancellation,” but policy does not allow it. Check whether the agent maintains the correct judgment.
- Cover the full easy-to-hard range so the dataset does not rapidly become obsolete as models improve.

### 5. Preventing Data Leakage
- **Multiple sources + private attachments + exact matching (GAIA style)**: answers require combining multiple sources, so no single web page directly supplies them. Some tasks include PDF/audio/image attachments unavailable online. Evaluate with exact string matching.
- **Parameterized template instantiation (AndroidWorld style)**: tasks are templates such as “change contact X's phone number to Y,” with random parameters each evaluation. Three benefits: replaying a memorized action sequence fails; one template generates nearly unlimited instances; holding some parameters constant while varying others precisely measures particular factors.
- **Canary identifiers (Terminal-Bench style)**: embed a canary GUID in the task text. If a model can produce content containing that GUID, the benchmark has entered its training data. This does not prevent leakage, but makes it detectable.

### 6. Quality Control and Long-Term Maintenance
- **Manual rejection before release**: SWE-bench Verified sampled 1699 of 2294 original tasks. Ninety-three Python-proficient developers checked each for clear descriptions, edge-case test coverage, stability, new errors introduced by reference patches, and reasonable difficulty. Only 500 passed: rejecting 71% improved signal-to-noise and reduced evaluation cost by about 80%. Complex agent tasks take minutes to hours, and evaluating a frontier model on an entire dataset often costs thousands of dollars in tokens. Cost control determines iteration speed.
- **Ongoing repairs after release**: in the 15 months after OSWorld's release, over 300 issues emerged across four categories: environment issues (anti-scraping, CAPTCHA, dynamic content; address with pinned versions and offline backups), ambiguous descriptions (rewrite), overly strict or loose verification logic (establish correct human baselines, then adjust conditions), and incomplete initial states (add completeness checks). These repair methods are transferable.
- **Five typical redesigns from an initial to a mature version (τ-bench → τ²-bench)**: vague, guessable task instructions → separate known_info/task_instructions; imprecise success conditions → verifiable boundaries; mechanical simulated users → emotions, patience limits, and factual grounding; only agents can change the environment → dual control with user-side tools; static tasks → parameterized bulk instance generation, improving both coverage and leakage resistance.

### 7. Three Dataset Sources and Their Evolution
- **Public benchmarks**: use for rough model screening and borrowing design techniques—verification depth, parameterization, leakage prevention, and maintenance. Generally avoid using them for product decisions because their task distributions differ from the business's.
- **Custom business datasets**: cover the real task distribution and guide model selection and harness design. A mature benchmark can serve as a skeleton with domain data and tools replaced.
- **Production trace feedback**: failures identified through explicit user corrections, downvotes, or post-hoc rule/LLM audits become regression cases after failure attribution. These have the highest cost and accuracy because they come directly from real user problems.
- Evolution: start with public benchmarks and a small handwritten business dataset. After some production use, feedback-derived cases become the majority. A failure pattern exposed today becomes tomorrow's regression protection.

## Common Pitfalls
- Broad instructions let models pass by guessing a plausible workflow without clarifying the request.
- Unbounded success conditions such as “the network is restored” admit superficial symptom suppression.
- Checking FAIL_TO_PASS without PASS_TO_PASS allows assertion-deletion cheating; retaining flaky tests makes results irreproducible.
- Ungrounded user simulators reduce evaluation to mutual confirmation; no patience limit means poor communication efficiency is not counted as failure.
- Models memorize static task text, and without canaries leakage goes unnoticed.
- Verifiers are too strict (reject correct results) or too loose (accept apparent completion with incorrect substance); incomplete initial states make runs start differently.
- `reward_basis` checks only final state, missing process violations such as policy breaches or omitted communication, with no separate process checks.
- Descriptions have multiple reasonable interpretations but verifiers accept only one, creating disagreement between task developers and human executors.
- Once built, the dataset is never maintained: the same few dozen handwritten tasks remain after launch, disconnected from real users' task distribution.

## Companion Code
- `chapter7/tau2-bench-eval/` — Full dissection of the four τ²-bench telecom task parts (ticket / user_scenario / initial_state / evaluation_criteria), plus dual-control environment run records.
- `chapter7/public-health-reporting-eval/` — Verifier design through structured six-point scoring of five deterministic tasks: tool selection, arguments, answers including numerical tolerance, exact sets of evidence rows, and penalties for unsupported claims.
- `chapter7/android-world/` — T3A failure traces and attribution notes demonstrating parameterized instances, final-state verification, and generating trajectory-prefix regression tasks from failure traces.

## Further Reading
- `book/chapter7.md`, “Anatomy of an Evaluation Task: The Telecom Domain in τ²-bench.”
- `book/chapter7.md`, “Designing Evaluation Datasets.”
