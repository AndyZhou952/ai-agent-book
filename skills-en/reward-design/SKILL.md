---
name: reward-design
description: Use when designing RL reward functions or diagnosing reward hacking—choosing rules, human preferences, or model judging; outcome or process rewards; scalars, vectors, or generative diagnostics; and path constraints with RLVP to penalize prohibited actions. Also covers multi-turn agent credit assignment, hidden tests, and early-stopping criteria. Triggers include reward design, reward hacking, reward seeking, RLVR, RLHF, GRM, ORM, PRM, RLVP, credit assignment, hidden tests, process rewards, reward shaping, and within-group variation.
---

# Reward Design

Translated from the [Chinese original](../../skills/reward-design/SKILL.md).

## When to Use

- Define RL rewards or review whether existing rewards incentivize incorrect behavior.
- Decide whether a task needs verifiable rewards, human preferences, or model judging.
- Outcome rewards are too sparse, or all-correct/all-wrong rollout groups produce no gradient.
- Assign credit in multi-turn agent tasks: which step caused terminal success or failure?
- Distinguish “the task succeeded” from “it succeeded through permitted means.”
- Diagnose verbal promises to verify, deletion of failing tests, or early stopping after self-selected easy checks.
- Design completion criteria the model cannot alter: hidden tests, state assertions, or external termination hooks.

## Core Principles

- **Reward design has three complementary dimensions**: **where rewards come from** (rules/preferences/model judging), **when they arrive** (outcome or process), and **how much information they express** (scalar/vector/generative diagnosis). A fourth question asks whether a successful result also followed a **compliant path**. Answer all four before implementing rewards.
- **Verifiable rewards (RLVR) are the most reliable source.** Judge results directly with tests, database assertions, state diffs, or format checks. More definite rules mean cheaper, reproducible rewards that are harder to exploit. Prefer binary scalars when correct answers or tests exist.
- **Reward hacking and reward seeking are different.** Hacking exploits rule or implementation loopholes for high scores, such as changing tests or fabricating results. Seeking means modeling “what the evaluator will inspect” and adapting behavior to that guess. It need not alter tests, but in long tasks it may create an overly easy check and stop as soon as it passes. **“Passed the grader” does not automatically mean “completed the task.”** The evaluator proxies intent; stronger training makes the model more likely to treat that proxy as the objective itself.
- **Reward outcomes and constrain processes (RLVP).** Outcome rewards answer “was it accomplished?” but cannot express “was it accomplished as required?” Agents can appear successful by editing tests, skipping authentication, or running destructive commands. RLVP targets mechanically checkable, **outcome-neutral constraints** independent of final success. It cannot replace independent checks of semantic intent, delivery completeness, or premature stopping.
- **Real environments usually have asymmetric verifiers.** Detecting a bad action is cheap and reliable; proving meaningful progress is difficult. Penalties can therefore almost always restore within-group variation, while progress rewards help only when partial progress is attainable.
- **Every added dimension adds another possible exploit.** Do not stack unverifiable dimensions merely to enrich rewards. First verify that a signal creates meaningful within-group differences in a few rollouts before adding it to training.
- **Terminal feedback in multi-turn tasks needs external credit assignment.** Multi-turn LLM RL usually sets γ=1. PPO's value network or turn-level advantages attribute terminal feedback to earlier actions; GRPO spreads trajectory-level advantages across generated tokens, requiring particular care about signal dilution in long trajectories.

## Practical Patterns

### 1. Three Reward Sources

| Source | Suitable for | Cautions |
|---|---|---|
| Rules / verifiable rewards (RLVR) | Math answers, code tests, structured tool calls, state assertions | Establish first; start with binary outcome rewards |
| Human preferences / reward models (RLHF) | Open-ended tasks whose goals cannot be fully expressed as rules | Reward models only proxy preferences; overoptimization causes reward hacking. KL regularization usually anchors policies near the SFT reference model |
| Model judging (GRM) | Rules cannot be exhaustively enumerated; diagnostics are needed | Generate “what worked and what needs improvement,” beyond scores. Have the model derive evaluation principles, assess trajectories using them, then verify judgments against facts. Retain manual calibration samples to avoid new judge biases |

### 2. Reward Timing: Outcomes First, Process Signals as Needed

**Outcome rewards (ORM)** judge task completion only at episode end. They are simplest and offer the greatest exploration freedom. They are an appropriate starting point when intermediate paths lack agreed standards or optimal solutions remain undiscovered, as with SimpleVLA-RL's sparse success/failure rewards.

Key implementation: in long coding or cowork tasks, completion must be decided by **hidden tests, state assertions, or external termination hooks the model cannot write**, rather than self-reported completion. When it claims to finish, run acceptance tests it cannot see in an isolated workspace: +1 on pass, −1 on failure. Tests must read real files or environment state, not merely check that the model said “done”; otherwise it learns to promise verification without doing it.

**Process rewards (PRM)** provide feedback on intermediate steps—authentication, tool arguments, test-pass counts, navigation actions. They ease long-horizon credit assignment but can constrain the model to the designer's intended path, and cost more to label and verify.

Engineering route: **establish a reliable outcome-reward baseline first; add process signals only for genuinely verifiable intermediate events.**

### 3. Reward Density and Representation

Density and representation are separate questions. Choose simply:

- Definite answers or tests → **prefer binary scalars**.
- Multiple independent quality goals → **vectors** scoring accuracy, completeness, cost, and safety separately, or a weighted scalar combination.
- Open-ended tasks with rules that cannot be exhaustively enumerated → **generative diagnostics**, with factual verification and manual spot checks.

A semi-scalar form—a short rationale followed by a score—lies between these and supports attribution.

### 4. Path Constraints and RLVP: Merge Two Reward Channels

Split verifier output into two channels and feed the existing policy optimizer. PPO/GRPO stay unchanged; only the rewards seen at each step change:

```python
outcome = verify_final_state(trajectory)              # Read actual results, not model self-reports
path_signal = 0
for step in trajectory:
    path_signal += deterministic_path_signal(step)    # Penalize violations or reward reachable subgoals
reward = normalize(outcome) + beta * normalize(path_signal)
```

Total reward is `R = O + βΦ`: `O` is task outcome, and `Φ` is a path signal calculated action by action with deterministic rules. Penalize verifiable violations and give small partial rewards for verifiable compliant actions or reachable subgoals. **Normalize the two channels separately before merging**, so path signals do not overwhelm the main objective.

Four RLVP design rules:

1. **Penalize concrete actions, not “insufficient effort.”** Penalty rules must be deterministic and hard to exploit.
2. **Always retain outcome rewards** so the model does not learn inaction, which avoids all violations.
3. **Ideally pair each penalty with an attainable compliant path.** If the base policy never samples compliant actions, seed that path with a few demonstrations, then gradually reduce path shaping once compliant behavior stabilizes.
4. **Test signal attainability before adding reward dimensions.** In software repair, if no rollout can pass any test, progress signals are unattainable and adding them brings no benefit.

The balance: **penalties are the usually attainable half; progress rewards are the half gated by attainability.**

### 5. Credit Assignment in Multi-Turn Tasks

- When terminal success/failure is hard to attribute to a turn, prefer **stepwise verifiable intermediate events** for process signals over inventing more complex advantage estimators.
- Environment-returned tokens in tool trajectories are not policy-generated. **Mask these feedback tokens in policy-gradient computation**, backpropagating only through model reasoning and tool-call arguments.
- Use PPO with a value network for fine-grained credit assignment. With GRPO, spreading trajectory-level advantages over all generated tokens dilutes signals, especially on long trajectories.
- Treat “claim completion” as a distinct decision point: attach an external verifier before it, rather than expecting terminal rewards to correct it incidentally.

## Common Pitfalls

- Rewards based on response length produce verbose, meaningless text. Evaluate the final goal rather than an intermediate metric.
- **Equating “passed the grader” with “completed the task”** ignores reward seeking: the model sets an easy check, passes, and stops prematurely with a deliverable satisfying only the proxy.
- **Stacking reward dimensions**: every unverifiable addition creates another exploit. First test for within-group variation on a few rollouts.
- **Only path penalties, no outcome rewards**: the model quickly learns to do nothing.
- **Adding unattainable progress rewards**: in all-failure groups the progress signal has no value and only adds noise.
- **Over-shaping process rewards** constrains exploration to the designer's intended route and can reward a correct process producing a wrong result.
- **Systematic verifier bias**: RL only learns to exploit it faster. Before scaling, test whether tasks are achievable and verifiers distinguish correctness.
- **Self-reports replace state checks**: deleting failing tests can make tests pass; promising a refund “within 7 days” can temporarily satisfy users. Code must check the policy repository, permission tables, and action traces at the process-verification layer.
- **Actively exploiting simulator loopholes**: when an LLM plays the environment, the target changes from real-world rules to simulator biases and loopholes, which RL actively searches for. A safer hybrid lets model simulation handle most interactions while supplementing real-world interaction and regular calibration.

## Companion Code

- `chapter8/RLVP/` — RLVP reproduction guide: outcome rewards plus deterministic path signals on GRPO. The book reports TerminalBench violations 3.71→0.66 with essentially unchanged success; on miniF2F, attainable partial rewards reduced iterations needed for 0.9 success from 7.0 to 4.4.
- `chapter8/premature-completion-dpo/` — Hidden-test rewards for premature completion: `data/hidden_tests.json` supplies model-invisible end-to-end acceptance scripts; `train_grpo_optional.py` is an optional RL branch for the same problem.
- `chapter8/SimpleVLA-RL/` — Sparse outcome rewards preserving maximum exploration: one demonstration per task for SFT cold start, 17.3%→91.7% success.
- `chapter8/retool/` — How tool feedback changes reasoning strategies: models learn to execute proactively, read errors, and self-correct.
- `chapter8/AWorld-train/` — Rewards and replayable environment pipelines in an MCP multi-tool sandbox.

## Further Reading

- `book/chapter8.md`, “From Single-Turn to Multi-Turn: Task Scenarios and Credit Assignment.”
- `book/chapter8.md`, “Reward Design: Turning Task Goals into Learning Signals.”
- `book/chapter8.md`, “RL Environments: From Evaluation to Simulation.”
- `book/chapter8.md`, “Practical Post-Training Considerations.”
- `book/chapter9.md`, “Obtaining Learning Signals from Execution Traces.”
