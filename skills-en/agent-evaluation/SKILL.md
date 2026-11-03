---
name: agent-evaluation
description: Use when building or improving agent evaluation—designing Pass@k/Pass^k/cost metrics, reproducible environments, deterministic verifiers or LLM-as-a-Judge within their appropriate limits, rubrics, failure attribution and regression tasks, evaluation-driven model selection, statistical significance, observability, and internal evaluation infrastructure (ablations, A/B tests, feature flags, prompt regression), and turning benchmark reports into system improvements.
---

# Agent Evaluation Systems

Translated from the [Chinese original](../../skills/agent-evaluation/SKILL.md).

## When to Use
- Build an agent system's first evaluation suite or audit the trustworthiness of an existing one.
- Obtain evidence before choosing or switching models: higher public scores do not imply better performance on your tasks and can conceal regressions.
- Determine whether falling scores reflect agent degradation or a faulty evaluation system.
- Turn broad conclusions such as “success rate fell” into specific, fixable problems.
- Build internal production evaluation infrastructure: ablations, A/B tests, feature flags, and prompt regression.
- Design observability and production trace feedback so the dataset evolves with the product.

## Core Principles
- **Evaluate the combination of model and harness, not the model alone.** The same model performs very differently under different harnesses. Suspect the harness before the model when performance is poor.
- **Use model-swap experiments to distinguish two problems**: hold the harness fixed and change only the model. No score improvement suggests a harness bottleneck; large variations with model capability suggest a model bottleneck. Ablations instead disable harness components to identify important internal parts. The two methods complement each other; do not confuse them.
- **Check the evaluation system before changing the agent.** When scores fall, first inspect random failures from insufficient resources, verifier bugs rejecting correct results, and cases disconnected from production. These look numerically identical to model degradation; only replaying full traces distinguishes them.
- **Metric definitions depend on the business scenario.** The same 0.8 means one in five refund-service users does not receive a refund, but may be respectable in vulnerability discovery. State whether k means independent samples of one task or k consecutive tasks in a production pipeline.
- **Use programmatic assertions whenever a check can be expressed that way.** Reserve LLM judging for dimensions that truly cannot be checked mechanically. Deterministic verification is cheaper, more reproducible, and suitable for long-term CI.
- **A score difference should exceed noise, hold under paired analysis, and be reproducible before it justifies switching models or releasing changes.**
- **One round of evidence supports only a proportionate next step.** Passing 4/4 in a subset does not establish 100% system-wide performance. Do not deploy before the complete retest passes.

## Practical Patterns

### 1. Metric Design: Pass@k / Best@k / Pass^k
- **Pass@k** (at least one pass in k attempts) and **Best@k** (the best continuous score) measure capability ceilings. They suit scientific discovery, vulnerability research, and open-ended creation, where a human selects the best of k candidate trajectories.
- **Pass^k** (all k consecutive attempts pass, with no veto-triggering safety, compliance, hallucination, or similar failures) measures business reliability. It suits payments, refunds, permission changes, and production deployment. Relations: Pass@k = 1-(1-p)^k; Pass^k = p^k.
- At p=0.6 and k=5, Pass@5 ≈ 99%, while Pass^5 ≈ 7.8%. The first answers “can it occasionally do something remarkable?”; the second answers “can it deliver reliably?”
- Do not retry side-effecting operations until they succeed. Sample in a sandbox or reversible environment, and count every failure in reliability metrics.

### 2. Five Elements of an Evaluation Environment
- **Dataset**: bundle initial state, agent input, simulator behavioral specification, and acceptance criteria into one record per case.
- **Environment state**: make it resettable (`initialization_actions` is the reset script), combining realism—changes follow business logic—with control—every run starts at the same point.
- **Tool interfaces**: both sides' tools should be atomic operations. Overly high-level abstractions such as a “solve the user's problem” tool reduce evaluation to one function call because the tool absorbs planning and reasoning.
- **Scoring criteria**: layered checks plus explicit aggregation rules, such as `reward_basis`.
- **Execution protocol**: termination signals, turn limits, and simulated users ending the interaction when patience runs out. Poor communication efficiency itself counts as failure.
- Match environments to task form: stateless single-turn environments for Q&A; multistep tool environments for search and synthesis; stateful environments for database changes and state verification; sandboxes for running code and inspecting output. Parallel sampling and trajectory caching are standard. Tool failures should return clear errors rather than a single failure flag so the agent can adapt.

### 3. Verification Spectrum and Automated Evaluation
- The horizontal axis is **mechanical verifiability**: deterministic assertions → checklists → rubric + LLM-as-a-Judge → paired comparison. Moving right does not mean abandoning the methods on the left.
- Deterministic verification identifies correctness but does not explain why. Production needs “what failed and what should change next.” Use LLM judging when open-ended tasks—reports, complaints, creative work—lack a unique final state and manual assessment cannot scale.
- **Four rubric criteria**: expert guidance (capture key facts and reasoning rather than fluency); comprehensive coverage (facts, logic, completeness, safety, and explicit pitfalls); importance weighting (required/important/optional/trap, with vetoes such as zeroing the score for hallucinations); self-contained standards (replace “shows deep understanding” with “cites at least two authoritative theories and explains them accurately”).
- Give each dimension objectively verifiable scoring levels, examples, and boundary cases. Explicitly guard against reward gaming through hallucinations, sycophancy, keyword stuffing, and evasion of difficult questions.
- Prevent **length bias**: explicitly penalize verbosity and set length limits for comparable tasks; keep candidate lengths similar before paired comparisons; regularly audit score–length correlations. If high scores nearly always accompany long answers, judging is already biased.
- **Same-family model problem**: when the agent and judge share a model family, the agent may exploit the judge's preferences and blind spots. Goodhart's law: once a measure becomes an optimization target, it ceases to be a good measure. Use a different model family for judging and refine rubrics through disagreements.

### 4. Failure Attribution: Turning a Zero into a Fixable Problem
- Attribute the **first** trajectory error that caused the task to deviate. Later errors are often cascading effects; the last error message is not necessarily the root cause.
- Three sources of problem cases: explicit user corrections; downvotes or negative feedback; post-hoc state checks, rule verifiers, or LLM review discovering prohibited actions.
- Two tools locate **silent failures**, where no tool errors occur and the agent announces completion: factual-anchor comparison (align agent statements with tool results step by step and find the first divergence); trajectory-prefix bisection (handoff to a human after step k; if recoverable, the error lies after k). Searching error-message keywords is no substitute.
- Structure attribution records as JSON/YAML with task name, first-error step, category, responsible party, original evidence, and confidence. Distinguish root cause from effects: three retries are consequences, not three causes. When categories overlap, choose the earliest cause that explains subsequent failures.
- **Rule filtering followed by LLM localization** is cheaper and more accurate than sending every trace to an LLM. First flag mismatches between completion claims and executed commands, diffs touching test assertions or skip markers, and diffs changing public APIs/schemas without migrations.
- Watch for “correct action, incorrect communication”: tools and environment state are right, but the user receives incorrect information. Final-state-only evaluation masks this entirely (about one-third of failures in the τ²-bench baseline). Attribute causes at the correct layer: a missing observation channel is not “the model cannot do OCR.”
- Store attribution records with task goals, environment state, agent version, toolset version, and full traces for regression testing.

### 5. Two Levels of Regression Tasks
- **End-to-end regression**: run from initial to final state; verify final state, required output, and safety conditions. Closest to production results, but difficult for localizing failed steps.
- **Trajectory-prefix regression**: freeze context, conversation, tool outputs, and environment state immediately before the first error; require only the next observable action or few actions. Cheap and isolates a policy problem. For highly reliable production agents, it is often more important than end-to-end measures.
- Define prefix-regression answers as an **acceptable action set**—“read repository rules first,” “ask the user first,” “refuse a dangerous operation”—plus forbidden actions, rather than one canonical answer.
- Generate tasks by error category: missing workflow → end-to-end tasks with planning documents and test acceptance criteria; tool-call error → truncate the failing prefix and turn it into a boundary task; communication errors → assert answer content itself, beyond environment state.

### 6. Evaluation-Driven Model Selection
- Latency has two stages: prefill determines time to first token (TTFT = queueing + prefill); decode determines output speed and reasoning duration (2000 reasoning tokens at 50 tokens/s take 40 seconds). Inspect p95 tail latency rather than averages.
- Measure reasoning-token usage and corresponding benefits on your workload; do not infer them from public leaderboards.
- Calculate **average cost per task** and cost–performance ratio. A cheap model with low success can cost more through retries. Price input, output, and cached tokens separately.
- Choose metrics by scenario: everyday work uses Pass@1; critical operations use Pass^k; exploration uses Pass@k / Best@k.
- **Budget–capability curves**: beyond success rates, report performance against wall-clock time, tokens, tool calls, or compute budget. A lead at short budgets does not establish long-run capability (RE-Bench: at a 2-hour budget, the best agent was about 4× human experts; humans overtook it at 8 hours).
- **Behavioral strategy is another selection dimension**: under one neutral harness, measure the “action threshold”—tool calls and files read before the first edit. Behavior changing with the model while persisting for the same model across harnesses supports a model effect; the reverse supports a harness effect. Separate tool-step counts, parallel calls, and model latency. Tool-step counts are not quality scores.
- Heterogeneous combinations—small models for simple work, stronger models for complex work—need evaluation proving overall benefits exceed added complexity and checking for scenario-specific regressions.
- Make cost optimizations independently switchable: stable KV-cache prefixes, context compression, and tiered model choice. **Also test them together on full tasks**: savings cannot simply be added (28.3% and 17.5% together saved only 30%).
- Monitor costs in real time by task type/model/user. Track tokens and charges, cap per-task costs, and automatically stop agents stuck in loops.

### 7. Statistical Significance
- Standard error SE(p) ≈ √(p(1-p)/n). With 100 cases and 70% success, a 95% confidence interval is about ±9 percentage points. “New model 73%, old model 70%” does not justify switching.
- Use **paired analysis** for two configurations on the same tasks: record per-task winners, then use McNemar's test or paired bootstrap rather than subtracting two independent success rates.
- Run 3–5 random seeds per configuration and report means and variation. A single run can only screen directions.
- If expected gains are only 2–3 percentage points and the dataset has a few dozen tasks, expand the sample first: standard error shrinks with 1/√n.
- Account for **multiple comparisons** when testing several hypotheses: tighten significance thresholds or independently rerun positive findings.

### 8. Observability
- The data foundation is a **trace/span tree**: one trace per task; one span per LLM call, tool call, or retrieval, recording input/output, start/end times, token usage, and errors. Parent–child links form the execution tree.
- Use standards such as OpenTelemetry and OpenInference semantic conventions to decouple collection from analysis and avoid platform lock-in. Collect asynchronously in batches without delaying responses.
- Four key benefits: diagnosis through replay rather than guesses; continuous optimization by locating unreliable tools and empty retrievals; cost management by finding abnormal high-cost cases; and a foundation for post-training data.
- The most valuable destination is **feedback into evaluation**: select failed and suspicious production traces → remove private data and secrets → turn them into new evaluation and regression cases. The dataset is a living product asset, not a one-time static collection.

### 9. Internal Evaluation Infrastructure for Production
- **Ablations**: make every major feature independently disableable; run ablations regularly, such as before major releases, to find “feature debt”—features once useful but now unnecessary as models improve. Inject ablation switches very early in startup; they cannot be retrofitted afterward.
- **A/B tests**: use multiple arms rather than only two to reveal dose–response relationships and optima. **Separate mechanism metrics from goal metrics**: shorter plan files are a mechanism, lower session costs the goal. Overly short plans can cause more edit–check cycles and cost more. Set guardrails so satisfaction, action counts, and error rates do not worsen. Record baseline statistics—sample sizes, percentiles, correlations—or significance cannot be assessed.
- **Two-layer feature flags**: compile-time flags physically remove code and provide clean ablations; runtime flags are delivered from the server and cached locally, favoring slightly stale values over blocking startup. Log each feature exposure at most once per session. Feature flags are first-class architecture for experiments, gradual releases, and emergency circuit breaking.
- **Prompt sensitivity**: system prompts must render deterministically—identical configuration inputs always produce identical text. Keep versioned snapshots; regression-test every prompt change like code in CI.
- **Privacy-aware analytics**: analytics interfaces accept only specially typed wrappers, whose type names serve as audit clues. Design privacy constraints from day one. If analytics cannot collect data safely, evaluation has no foundation.

### 10. The Improvement Loop from Benchmark Reports to Systems
- Start reports by finding failure concentrations across tasks and capabilities, then replay traces to distinguish failures in **observation, reasoning, action, or verification**. Do not focus solely on aggregate scores; 88% overall can easily hide a concentrated failure cluster.
- Change one variable per round, building a hypothesis chain: more prompting does not help → changing the input channel helps but costs too much → trimming inputs preserves success while lowering cost. Detailed prompts cannot restore information the agent never saw; more input is not always better either.
- Passing a small subset only qualifies the change for broader testing. Before discussing deployment, use the standard environment and all tasks × multiple seeds, verifying no success-rate regression and that tokens and latency stay within limits.

## Common Pitfalls
- Changing agent code after scores fall without inspecting evaluation resources, verifier bugs, or outdated cases.
- Making product decisions directly from public benchmark scores: two points on GAIA have no necessary relationship to refund success.
- Treating the last error as the root cause, or attributing it to the wrong layer—for example, calling a missing observation channel “cannot do OCR,” then switching models or training OCR.
- Checking only final state and missing “correct action, incorrect communication.”
- Unverifiable rubric abstractions such as “shows deep understanding”; no veto criteria, allowing keyword-stuffing reward games.
- Never auditing score–length correlations and remaining unaware of length bias.
- Using the same model family as both contestant and judge, inflating scores under Goodhart's law.
- Adding savings from multiple context optimizations or selecting models from a single run.
- Failing to swap candidate order in paired comparisons, letting position bias systematically favor the first candidate.
- Reporting a passed small subset as overall performance, beyond what the evidence supports.

## Companion Code
- `chapter7/tau2-bench-eval/` — Five-task τ²-bench telecom evaluation in a dual-control environment: resets, saved trajectories, binary rewards, and failure analysis (wrong line selected, recharge omitted).
- `chapter7/android-world/` — T3A run records, failure attribution notes, and the five-stage diagnose→hypothesize→experiment→decide→iterate improvement loop.
- `chapter7/public-health-reporting-eval/` — Deterministic six-point scoring on synthetic data: tool selection, arguments, answers, evidence rows, and penalties for unsupported claims.
- `chapter7/agent-cost-analysis/` — Full cost breakdown of an eight-turn refund task, with 2×2 A/B quantification of KV-cache-friendly design and context compression.
- `chapter7/model-benchmark/` — Multidimensional campaigns: throughput/TTFT/tail latency, 168-hour availability, rate-limit ramping, and same-model comparisons across providers.
- `chapter7/model-action-threshold/` — Action thresholds across models under a fixed coding harness: tool calls and files read before the first edit.
- `chapter7/elo-leaderboard/` — Elo/Bradley-Terry paired rankings and historical snapshots from real Chatbot Arena votes.

## Further Reading
- `book/chapter7.md`, “Evaluation Metrics: Defining Success.”
- `book/chapter7.md`, “Evaluation Environments.”
- `book/chapter7.md`, “Automated Evaluation Methods.”
- `book/chapter7.md`, “Evaluation-Driven Model Selection.”
- `book/chapter7.md`, “Statistical Significance of Evaluation Results.”
- `book/chapter7.md`, “Agent Observability.”
- `book/chapter7.md`, “From Benchmark Reports to System Improvements.”
- `book/chapter7.md`, “From External to Internal Evaluation: Evaluation Infrastructure for Production Agents.”
