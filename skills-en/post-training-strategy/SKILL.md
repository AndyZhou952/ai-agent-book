---
name: post-training-strategy
description: Use when post-training agent models or choosing a training route—deciding whether to build foundations with mid-training, establish protocols with SFT, or proceed to RL; constructing mid-training corpora and SFT data; building RL environments; choosing GRPO/PPO/DPO or distillation; and diagnosing near-zero pass@k, RL with unstable formatting, no within-group reward variation, training–inference numerical mismatch, or reduced generalization from overtraining. Triggers include post-training, mid-training, continued pretraining, SFT, RL, GRPO, PPO, DPO, rejection sampling, RFT, distillation, on-policy, RLVR, and LoRA.
---

# Model Post-Training Strategy

Translated from the [Chinese original](../../skills/post-training-strategy/SKILL.md).

## When to Use

- A model performs poorly on a task family: determine whether it lacks foundations, protocols, or strategy.
- Choose mid-training / SFT / RL for a project, or decide whether RL is warranted.
- Construct mid-training corpora, SFT demonstrations, or rejection-sampled data.
- Build RL training environments and choose algorithms and LoRA hyperparameters.
- Improve sample efficiency through CoT distillation, prompt distillation, on-policy distillation, or self-distillation.
- Diagnose RL non-convergence, absent within-group variation, or capability regressions after updates.

## Core Principles

- **The four stages are not three different “fine-tuning intensities”; they address foundations, protocols, and strategy.** Mid-training adds knowledge and basic capabilities, SFT establishes output formats and behavioral protocols, and RL reallocates probability mass and explores strategies beyond demonstrations. Each has an entry threshold; this is not a ritual pipeline.
- **Data and environments matter more than algorithms.** Success depends on whether mid-training corpora fill foundational gaps, demonstrations establish protocols, and simulation and rewards provide reliable trial-and-error feedback. Basic PPO/GRPO usage is sufficient; in many scenarios, getting the first two kinds of data right removes the need for RL.
- **`pass@k` is the most important routing metric.** Sample held-out tasks at a temperature close to training: `pass@k = 1-(1-p)^k`. Low `pass@1` but rapidly improving `pass@k` as k grows means correct strategies already exist in the distribution, giving RL/rejection sampling/distillation something to amplify. If `pass@k` remains near zero at reasonable k and temperature, the base model can scarcely generate successful trajectories. With only 0/1 rewards, every rollout in a GRPO group receives 0, eliminating within-group advantages.
- **“SFT memorizes, RL generalizes” is a tendency measured in controlled experiments, not a universal property.** SFT can generalize with diverse data and appropriate regularization; RL can overfit biased rewards or environments. Mechanistically, maximum-likelihood SFT tends toward mass covering, while RL with reverse KL tends toward mode seeking.
- **SFT and pretraining are the same mathematical task**—next-token prediction—differing only in data organization and loss masking (backpropagating only on answer tokens). This makes SFT highly sample-efficient, with effects from a few thousand examples, but also prone to memorizing question phrasing instead of acquiring knowledge when demonstration coverage is insufficient.
- **LoRA is the engineering default**: apply it to all major weight matrices, especially MLPs, which contain the largest share of parameters; attention-only adaptation loses performance. The optimal learning rate is about 10× that of full-parameter fine-tuning. Use medium-to-high rank (64–256) for SFT; RL supplies less information per update, so use low rank (8–32), or even 1.
- **On-policy is a numerical issue, not just a scheduling issue.** Floating-point differences between sampling and training engines can make pre-update probability ratios deviate from their expected value of 1, quietly turning nominally on-policy training into off-policy training. Sampler/trainer consistency deserves the same attention as reward curves.

## Practical Patterns

### 1. Route Selection: Diagnose the Gap Before Choosing a Method

| Observation | Main gap | Preferred method | Gate for the next stage |
|---|---|---|---|
| Cannot handle domain concepts, language, or basic operations; `pass@k` stays near 0 under reasonable sampling | Knowledge and capabilities lie outside the base model's effective support | **Mid-training**; RAG for dynamic facts | Improved domain holdouts, acceptable general-capability retention, and verifiably correct or partially correct target-task trajectories begin appearing |
| Occasionally succeeds, but formatting, tool schemas, tone, or fixed procedures are unstable | Behavioral protocol is not established | **SFT** or constrained decoding | Stable parsing success; verifiers can reliably score critical actions and output protocols |
| Nonzero success and reliable rewards, but good strategies have low probability, or long-horizon decisions/OOD generalization are weak | Probability-mass allocation and policy optimization | **RL** | Rewards align with real goals; rollout groups have enough reward variation; independent test performance improves during training |
| Only a few stable demonstrations and no interactive environment | Imitable data exists, online feedback does not | **SFT/RFT/offline preference optimization** | Establish baselines and evaluation first, then decide whether building an RL environment is worthwhile |

Decision sequence: **①** first rule out solutions without weight changes (do not train if prompts, tools, code constraints, or context management suffice; use RAG for facts requiring frequent updates, citations, or deletion) → **②** measure capability support on target holdouts (not only greedy `pass@1`, but fixed-configuration `pass@k`, partial-progress rates, format-parsing rates, and manual failure audits) → **③** use SFT to establish protocols, rather than force a knowledge base into parameters → **④** use RL only where exploration has room to help.

### 2. Mid-Training: Building Foundations

Five data-construction steps: **derive data needs from the failure distribution** (break evaluation down by topic/language/document type/code pattern/context length; address knowledge and capability gaps, without misdiagnosing formatting errors as missing knowledge) → **construct dense target corpora** (raw documents build terminology–fact associations; repositories teach structure and dependencies; textbook derivations and synthetic paraphrases make implicit relations explicit; deduplicate, filter quality, and check evaluation contamination) → **mix by capability** (natural long texts + CoT for atomic long-context abilities + agent execution traces; the latter two can be distilled from stronger open models) → **use two forms of replay at each stage** (original short texts and general data retain language knowledge; old tasks at increased lengths test whether the same capability persists in longer windows; ideally general data comes from the base model's original pretraining corpus, otherwise use open corpora such as FineWeb-2) → **stop according to multidimensional gates** (track held-out domain tasks, general capabilities, original instruction following, and target-task `pass@1`/`pass@k` together; domain improvement with general decline means overly aggressive mixing or learning rates; lower loss with unchanged `pass@k` means data misses required capabilities or SFT is needed to access acquired knowledge).

After mid-training, use LongBench v2, IFEval, and end-to-end agent evaluation to verify that **basic long-context capabilities remain intact across context lengths**: position and retrieval, relations and reasoning, aggregation and statistics, instruction following, long-chain reasoning, and atomic agent abilities. A robust recipe: mid-training absorbs knowledge and capabilities → small-scale SFT establishes output formats → RL only after nonzero success. Validate data mixtures at small scale before expanding training budgets.

### 3. SFT: Establishing Protocols

Three data routes, often combined: **human expert demonstrations** (highest quality ceiling, expensive and slow; suitable seed data for defining formats and style) → **teacher-model generation** (a strong model generates input–output pairs in bulk, filtered before distillation to a student) → **rejection sampling** (sample several candidates per problem, retain correct examples with a verifier, then train the model on them: RFT, a highly cost-effective construction method for verifiable tasks).

Construction: define the task distribution and output schema → generate candidates in bulk → quality-filter with rules, format checks, and manual spot checks → deduplicate, balance mixtures, and ensure diversity. **Thousands to tens of thousands of high-quality examples usually suffice to establish output formats**. Carefully refining ten thousand clean examples is better than accumulating a hundred thousand dirty ones: SFT may faithfully encode every noisy detail in its parameters.

Pipeline: **production data → task blueprints → synthetic tasks → multiple candidate trajectories → task verification and trajectory verification → SFT data**. Use deterministic code for conditions expressible as unit tests, database assertions, or state diffs. Supplement open-ended communication quality with model evaluators calibrated by manual sampling. Deduplicate and split training data by task template/customer/time period; evaluation must use non-overlapping task types. Reference solutions, hidden tests, and verifier feedback must not leak to the model.

### 4. RL: Four Steps for One Parameter Update

For GRPO, replicate one SWE-bench task across 16 isolated sandboxes. **① Rollout**: the policy independently solves it 16 times, each including the full “read code → edit files → run tests → submit” workflow. **② Calculate rewards**: a verifier applies patches and runs tests in a clean environment; 4 passes receive 1, 12 failures receive 0. **③ Calculate relative advantages**: with group mean 4/16, the 4 successful trajectories receive positive advantages and the 12 failures negative advantages. **④ Gradient-descent update**: increase probabilities of the model's choices in positive-advantage trajectories and decrease those in negative ones. All-success or all-failure groups have no within-group variation, so the gradient vanishes.

PPO and GRPO differ only in “what they compare against”: GRPO directly compares multiple rollouts of the same question and needs no value model; PPO trains a value model to estimate step by step “how well it usually does,” better suiting long trajectories requiring fine-grained credit assignment. DPO learns previously collected preference pairs instead of having the current policy generate online rollouts.

**Environment engineering sequence**: task blueprints → resettable simulator → deterministic verifier → training/evaluation isolation → calibration with a small amount of real interaction. Training and evaluation can share generators and verification code, but not task instances. Keep test cases, hidden state, and reference solutions on the verifier side. Before scaling sampling, use a few rollouts to check whether tasks are achievable and verifiers distinguish correct from incorrect. Deterministic verifiers' CPU throughput can become the bottleneck; throughput depends on parallel verifier workers, not adding GPUs.

A crucial tool-trajectory detail: **environment-returned tokens are not generated by the policy and must be masked when computing policy gradients**. Backpropagate only through the model's reasoning and tool-call arguments; otherwise it learns to predict sandbox outputs.

### 5. Distillation: Turning One Rollout into Dense Supervision

Poor sample efficiency stems from sparse feedback: one rollout ends with a single success/failure scalar. Distillation lets the same trajectory contribute many gradients.

- **On-policy distillation**: the student chooses where to go; at states it has reached, the teacher supplies a full next-token distribution, minimizing their KL divergence. A length-T rollout gives about T sets of token-level supervision. On math tasks, matching performance takes about 1/10 the steps of pure RL. The student must at least reach meaningful states the teacher can correct, and the teacher policy must not lie too far outside the student's effective support.
- **On-policy self-distillation (OPSD)**: without a stronger teacher, one model plays two roles. The teacher sees privileged information—reference answers, verified solutions, or domain documents—while the student sees only the question and aligns token distributions to the teacher along its own sampled trajectories. Construct privileged information only on the training side; never leak it to the deployed agent. This does not create knowledge from nothing: if the model cannot explain the process even with the answer, no extra signal exists. Naive OPSD may lose the original reasoning style, requiring retention sets/style regularization.
- **CoT distillation / prompt distillation**: the former transfers a strong teacher's full reasoning trajectories and can recover 70%–80% of its capability at the same parameter count. The latter compresses “long prompt + reasoning teacher” into “short prompt + non-reasoning student,” giving 20–30× faster responses. Distillation inherits teachers' systematic errors and verbose reasoning habits.

### 6. Pre-Deployment Checklist

- [ ] Measure `pass@1` and fixed-configuration `pass@k` on independent holdouts, and manually audit failures.
- [ ] Stage transitions have measurable gates, rather than treating “mid-training → SFT → RL” as ritual.
- [ ] Every stage has independent retention sets and early-stopping criteria, separately tracking domain/general/instruction-following abilities.
- [ ] Before RL, rewards produce meaningful within-group variation in a few rollouts.
- [ ] Compare sampler/trainer token log probabilities before updates; monitor probability-ratio means/percentiles/maxima, approximate KL, clipped fractions, and policy staleness.
- [ ] Synchronize weights plus LoRA adapters, tokenizer, chat template, model revision, and positional-encoding configuration.
- [ ] Validate key hypotheses at small scale before expanding training budgets.

## Common Pitfalls

- **Forcing a knowledge base into SFT**, or putting all knowledge in parameters. A few facts can be learned with demonstrations, but large interconnected knowledge belongs in mid-training. Facts needing dynamic updates, citations, permissions, or deletion belong in RAG.
- **Introducing RL before formatting stabilizes.** If the model cannot reliably generate JSON needed to compute rewards, training signals become sparse or distorted. Establish formatting thresholds through small evaluations; use SFT or constrained decoding if necessary.
- **Applying RL with near-zero `pass@k`.** All-failure rollouts offer no positive trajectories and no within-group advantages, usually wasting sampling budgets.
- **Equating nominal with effective context windows.** Positional encoding supporting 128K does not mean retrieval, reasoning, and planning still work at 128K. Pass capability gates at the current length before expanding; inspect regressions in a capability × length matrix.
- **Ignoring simulation fidelity.** Oversimplified simulations or unrealistic responses produce policies that fail immediately in production. High-fidelity environments may cost more to build than training itself. When a real environment cannot be built, use a model to synthesize tool responses and dynamics, but **the simulator's world knowledge is the training ceiling**. Its systematic biases transfer to the policy, requiring real interactions and regular calibration.
- **Overtraining reduces generalization.** Falling training loss with worsening validation means memorization. Mid-training can forget general capabilities, SFT can overfit demonstrations, and RL can overfit rewards and task distributions. All need independent retention sets and early stopping.
- **Value-function collapse and insufficient exploration.** Inaccurate PPO values bias advantages and cause highly oscillating curves. Low temperatures or inadequate randomness trap agents in local optima.
- **Treating training–inference numerical mismatch as minor noise.** Small log-probability errors are amplified through exponentiation, long-prefix accumulation, clipping, and advantage weighting. Symptoms include importance-ratio spikes, many clipped tokens, abrupt gradient or response-length changes, and simultaneous reward/entropy collapse.
- **Underestimating RL compute costs.** Moving a task that works under SFT to RL can require 10–100× the training time. SFT may suffice when test and training distributions are highly aligned.
- **Poor training data quality.** Mid-training absorbs false associations; SFT directly learns demonstration noise; systematically biased RL rewards amplify policies in the wrong direction.

## Companion Code

- `chapter8/continued-pretraining/` — Adding a language through continued pretraining (Mistral 7B + Korean Wikipedia, 80/20 mixing to mitigate forgetting, LoRA + training embed/lm_head), demonstrating separate mid-training→SFT responsibilities.
- `chapter8/curly-quote-sft/` — Scope-sensitive SFT: distill feedback into a `SKILL.md` specification, then train Qwen3-8B LoRA on structured synthetic data.
- `chapter8/exact-copy-sft/` — Byte-exact copying SFT for special strings, including tokenizer audits to rule out tokenization artifacts.
- `chapter8/cot-distillation/` — Three CoT-distillation stages: auditable teacher-trace collection filtered by rule verification → real student parameter updates → paired base/student/teacher evaluation.
- `chapter8/SimpleVLA-RL/` — Open exploration under sparse outcome rewards: one-demonstration SFT cold start + GRPO, success 17.3%→91.7%, discovering actions absent from demonstrations.
- `chapter8/retool/` — SFT warm-up followed by PPO with interleaved text–code reasoning and sandbox feedback, AIME 2024 about 25%→67.0%.
- `chapter8/AWorld-train/` — A resettable, replayable multi-tool training pipeline in an MCP multi-tool sandbox.
- `chapter8/RLVP/` — Reward outcomes, penalize paths: outcome rewards plus deterministic path signals on GRPO.
- `chapter8/AdaptThink/` — RL for a “when not to think” metapolicy, including constrained objectives and importance-sampling cold-start handling.

## Further Reading

- `book/chapter8.md`, “From Pretraining to RL: A Four-Stage Overview.”
- `book/chapter8.md`, “Mid-Training: Adding Knowledge and Foundational Capabilities.”
- `book/chapter8.md`, “SFT (Supervised Fine-Tuning)” and “SFT Data Synthesis: From Demonstrations to Trainable Trajectories.”
- `book/chapter8.md`, “When to Choose Mid-Training, SFT, and RL.”
- `book/chapter8.md`, “RL Algorithms: From 16 Rollouts to One Parameter Update.”
- `book/chapter8.md`, “RL Environments: From Evaluation to Simulation.”
- `book/chapter8.md`, “Distillation: Improving Sample Efficiency.”
- `book/chapter8.md`, “Practical Post-Training Considerations.”
