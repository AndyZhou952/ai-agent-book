---
name: bad-case-to-dpo
description: Use when turning production failures or failed trajectories into training data—from failure attribution and trajectory-prefix extraction through DPO preference pairs and LoRA training to boundary-set and retention-set validation. Also use to choose SFT, DPO, RL, or prior harness/tool/tokenizer repairs, and to extract evidence-backed structured learning signals from execution traces. Triggers include failure cases, failure attribution, DPO, preference pairs, trajectory prefixes, rejection sampling, overcorrection, boundary sets, retention sets, premature completion, reward hacking, LoRA fine-tuning, and training/evaluation isolation.
---

# From Failure Cases to Training Data

Translated from the [Chinese original](../../skills/bad-case-to-dpo/SKILL.md).

## When to Use

- Production failures appear through user corrections, downvotes, or post-hoc audits, and should become post-training input.
- Decide whether a failure category needs harness/tool/tokenizer repairs or warrants training.
- Construct DPO preference pairs, trajectory-prefix regression tasks, or decision-boundary demonstrations.
- Decide whether a failure belongs in SFT data, DPO pairs, PRM negative labels, or RLVP penalty rules.
- Validate a training fix without “fixing one case and breaking many others.”
- Extract learnable signals from execution traces, rather than keeping only aggregate scores.

## Core Principles

- **Attribute the first erroneous step, rather than the entire trajectory.** In premature coding-agent completion, reading and editing code may be correct; the mistake is the decision boundary where it concludes without evidence. Attribution must identify that step so data can target it.
- **Do not directly use failed trajectories as correct SFT demonstrations.** Use them to build preference pairs, identify task-coverage gaps, or add diagnosis and repair before training on them.
- **Judgments must use mechanisms the model cannot write.** Hidden tests, state assertions, or external termination hooks decide completion. Tests must inspect real files or environment state, rather than whether the model said “done”; otherwise it learns verbal promises to verify without actual verification.
- **Boundary and retention sets are both essential.** Boundary sets contain unfinished tasks and test whether the model continues verification rather than claiming completion. Retention sets contain finished tasks and test whether it still concludes normally. Optimizing only boundaries causes **overcorrection**: never daring to finish, verifying indefinitely, and making latency and cost collapse. This is the parameter-level version of “changes must preserve existing behavior.”
- **Strictly isolate training and evaluation.** Use different tasks, parameters, and template combinations. Reference solutions, hidden tests, and verifier feedback must not leak to the model. Enforce isolation with unit tests, rather than relying on memory.
- **Rule out non-model causes first.** Compare each layer: raw file bytes → tool result → harness serialization → model context → output tokens → decoded string → JSON/tool-call parsing → tool matching. Label a model-capability problem only when its context matches the original string exactly and **model output is the first point of divergence**. Otherwise fix tools/harness/tokenizer, then reassess what remains.
- **Small-data DPO must mix in general-task data.** When few cases exist, use a small mixing proportion in general tasks for LoRA fine-tuning. Avoid learning a new overfit rule such as “always verify before every conclusion,” and reduce catastrophic forgetting.
- **Learning signals need evidence.** A total score cannot identify what to change. At minimum, retain task success/partial success/failure, per-dimension conclusions, evidence locations for each conclusion (conversation turn or tool call), and failure-type labels. Verifiers must be allowed to **abstain from scoring** when evidence is insufficient. Exclude low-confidence cases from learning rather than encoding them as facts.

## Practical Patterns

### Full Pipeline: Failure Attribution → Dual-Set Validation

**① Collect and attribute.** Collect failed trajectories from three production signals: user corrections, downvotes, and post-hoc audits. Locate the **first erroneous step** and record its category. Trajectory-prefix regression tasks are cut at this decision boundary.

**② Route: should this be trained?** First ask whether prompts, tools, code constraints, or context management suffice. Then rule out tool/harness/tokenizer problems along the byte-processing chain. Do not train when weight changes are unnecessary.

**③ Choose the training form.** Map Chapter 7 evaluation-dataset types to Chapter 8 training uses:

| Available material | Training use |
|---|---|
| End-to-end regression tasks with verifiers | RL rollout tasks and verifiable rewards (RLVR); sampling pools for rejection sampling (RFT) |
| Trajectory-prefix regression tasks | DPO pairs, decision-boundary SFT demonstrations, teacher states for on-policy distillation |
| Failure-attribution records with first-error step and category | Negative process-supervision labels (PRM), rule sources for RLVP path penalties |
| Multidimensional rubric scores and human gold sets | Vector reward dimensions, training/calibration data for generative reward models (GRM) |

**④ Extract prefixes and build pairs.** Cut the prefix at the “about to claim completion” boundary. The rejected continuation is premature stopping; chosen is “run tests, check acceptance criteria individually, then conclude.” Generate chosen with a teacher, then filter through rule verifiers (rejection sampling). With too few cases, augment across task types, missing verification items, and completion wording to produce hundreds of pairs. Store failure reasons, applicability conditions, and verifiers with examples for tracing and review.

**⑤ LoRA DPO training.** Mix a small proportion into general-task data against overfitting. Reference configuration: Qwen2.5-7B-Instruct + bf16 LoRA, 4 epochs, learning rate 3e-5, about 24GB VRAM on one GPU. Record configuration, data hashes, and timestamps in training receipts.

**⑥ Dual-set validation.** Ask for the next action separately on unfinished tasks and completed-task retention sets; classify it as “claim completion” or “continue verification”:

- **Premature-completion rate on unfinished tasks** should fall after training.
- **Normal-conclusion rate on completed retention tasks** should remain intact.
- **Overcorrection rate = 1 − normal-conclusion rate** should stay low.

Prefer **fixed-format candidate comparisons consistent with training prompts**: fix two candidate actions and compare model preference, directly measuring the decision boundary, rather than relying on open-ended generation. Also spot-check general capabilities to ensure the LoRA patch did not damage them.

**⑦ Record and trace.** Store training and evaluation separately. If data construction has both deterministic-template and teacher-model routes, retain evidence receipts for both.

### Reference Case: Premature Coding-Agent Completion (Experiment 8-17)

Companion data: 24 failed trajectory prefixes across four premature-stopping categories—claiming completion without tests; completing only some objectives; claiming completion despite unmet acceptance criteria; giving up after errors and declaring impossibility. These include reward-hacking variants such as deleting failing tests or adding skips. Strictly isolated holdouts contain 12 boundary cases + 8 retention cases.

Measured results: fixed-candidate comparison on unfinished tasks gave the base model **3/12 (25.0%)** correct, with **8/8 (100%)** on completed retention tasks. After LoRA DPO, results were **11/12 (91.7%)** and **8/8 (100%)**. Free generation is supplementary diagnosis: premature stopping 1/12→0/12, but normal conclusions 6/8→0/8. **Small-data DPO thus made open-ended answers overly cautious.** Primary conclusions use only candidate comparison; they cannot be extrapolated into overall production success gains.

An RL branch addresses the same problem: make “acceptance tests must pass before claiming completion” a verifiable reward. Tests are invisible to the model and run only when it claims completion: +1 on pass, −1 on failure.

### Different Treatment for Two Other Common Cases

- **Rule feedback, such as “use curly rather than straight quotation marks in Chinese”**: a desired description is not directly trainable. The same quotation mark has different roles in Chinese prose, original English, Markdown inline code, code blocks, comments, JSON, and paths. Use **scope-sensitive minimal edits**: first write an auditable skill, with positive examples of Chinese passages and comments that need conversion and negative examples covering original English, string literals, JSON, paths, and inline code. This serves as synthetic-data labeling guidance, post-training regression specification, and retraining input when rules change.
- **Byte-exact copying, such as `edit_file` `old_string` mismatches**: express copying as three verifiable tasks—verbatim repetition, selecting the exact target among similar equal-length strings, and copying an entire specified string into the `old_string` tool argument. Include spaces, newlines, backslashes, and Unicode combining characters that real edits easily damage. Independently audit the tokenizer after training to exclude tokenization artifacts: if the round-trip ceiling is 80.1%, measuring 80.1% cannot count as purely improved model capability.

### Extracting Learning Signals from Traces: Three Verification Layers

Evaluating a trajectory answers three questions in sequence: **was the task accomplished, was it accomplished through permitted means, and was the user comfortable with the interaction?**

- **Bottom-layer outcome verifier** reads tests, database state, and tool results. It reflects real environment state, is more reliable than model self-description, and deserves first priority.
- **Middle-layer process verifier** checks business rules, permissions, and action sequences, distinguishing achieved results from permitted paths. Deleting failing tests can make tests pass; verbal promises can temporarily satisfy users. Policy repositories, permission tables, and action traces can be represented precisely, so code can judge this layer too.
- **Top-layer quality verifier** checks patience, compliant alternatives, and natural, concise wording. Use LLM-as-a-Judge with predefined rubrics, requiring per-item scores and trajectory evidence citations.

Output form determines whether feedback is learnable. A total score only ranks one run. Only with all four elements—outcome judgment, per-dimension conclusions, evidence locations, and failure labels—can later stages decide whether to update knowledge, prompts, programs, or model parameters.

## Common Pitfalls

- **Training directly on failures as correct demonstrations**, encoding mistakes in parameters.
- **Optimizing only boundary sets without retention**, producing a model that never dares to conclude.
- **Sharing tasks or parameters between training and evaluation**, losing independence. Enforce isolation through tests.
- **Using model self-reports for completion**, teaching verbal claims of verification instead of action.
- **Misreporting system-layer corruption as post-training benefit**: when direct-repetition probes work but tool calls still fail, fix the harness or tool protocol.
- **Mistaking tokenizer/serialization corruption for poor copying ability**: independently audit tokenizer round trips before training.
- **Too few cases without augmentation**: a handful of pairs cannot define a decision boundary. Augmentation that changes only wording, but not task types or missing checks, is also insufficient.
- **Small-data DPO without general data**, introducing “always do Y when X occurs” overfitting and catastrophic forgetting.
- **Low-confidence conclusions enter learning**, embedding false experience and prompt injections that amplify in later tasks.
- **Using open-ended generation as primary acceptance**: it can become overly conservative under small-data DPO; fixed candidate comparison matches training prompts.

## Companion Code

- `chapter8/premature-completion-dpo/` — The book's only training experiment starting from production failures: 24 failure cases, boundary 12 + retention 8 holdouts, and hidden tests. `python demo.py` gives an offline end-to-end demo; `python -m pytest -q test_pipeline.py` runs mechanism unit tests; `python build_preference_data.py` generates pairs (`--teacher` uses teacher-model rejection sampling); `python train_dpo.py --smoke` checks data/tokenizer forward passes; `python train_dpo.py` trains single-GPU LoRA DPO; `python evaluate.py --decision-score` evaluates dual-set candidate comparisons; `python train_grpo_optional.py` is the optional RL branch.
- `chapter8/curly-quote-sft/` — Another supervision target for rule feedback: write a `SKILL.md` specification, then structured synthetic data for scope-sensitive SFT.
- `chapter8/exact-copy-sft/` — Byte-exact copying SFT with `tokenizer_audit.py` to rule out tokenization artifacts.
- `chapter8/cot-distillation/` — Rejection-sampling pipeline: generate candidates → verify and filter → retain only correct trajectories.

## Further Reading

- `book/chapter8.md`, “From Failure Cases to Post-Training.”
- `book/chapter8.md`, “SFT Data Synthesis: From Demonstrations to Trainable Trajectories.”
- `book/chapter8.md`, “Reward Design: Turning Task Goals into Learning Signals.”
- `book/chapter9.md`, “Obtaining Learning Signals from Execution Traces.”
