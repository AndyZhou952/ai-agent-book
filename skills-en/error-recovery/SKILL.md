---
name: error-recovery
description: Use when diagnosing or hardening agent recovery—infinite loops, interrupted streams, context overflow, rate limits/overload, cross-provider continuation, retry and circuit-breaker design, or cascading failures inside error handling. Covers four failure layers, detection, graduated recovery, termination limits, and stream continuation.
---

# Failure and Error Recovery

Translated from the [Chinese original](../../skills/error-recovery/SKILL.md).

## When to Use
- An agent repeats a tool without progress or enters an infinite loop/death spiral.
- A stream breaks during reasoning, body text, or tool arguments and needs continuation.
- Model APIs rate-limit, overload, time out, or hit output limits, requiring retries or fallback.
- Context overflows, compression fails, or trajectory structure breaks (calls lack paired results).
- The main model remains unavailable and another provider must continue the trajectory.
- Design retries, circuit-breaker thresholds, and error-exposure boundaries.
- Let the model correct hallucinated tool calls or malformed parameters.

## Core Principles

**Failure taxonomy: four layers.** Classify before handling; recovery differs completely by layer:

- **API**: rate limits (429), overload, request timeouts, connection breaks, output-limit truncation. Infrastructure noise unrelated to task content—retryable.
- **Tool**: hallucinated calls (nonexistent tools), malformed parameters, execution exceptions, and the most dangerous case: repeated identical errors with unchanged retries.
- **Context**: window overflow, compression failure, malformed trajectory structure.
- **Control flow**: infinite loops (same actions without progress) and death spirals (recovery logic calls an LLM, fails again, and cascades).

**Detection: classify, then count.** First ask not "Should we retry?" but "Is retry worthwhile?" Retryable errors (rate limits, overload, network jitter) justify retries. Nonretryable errors (invalid parameters, insufficient permissions, missing tools) will have the same outcome after ten thousand unchanged retries; inputs or strategy must change. A production harness maintains an "error → recovery strategy" map rather than blindly retrying. Detect **patterns** as well as individual errors: fingerprint "tool name + arguments"; repeated fingerprints clearly signal a no-progress loop. Maintain separate consecutive-failure counters per recovery path for circuit breaking.

**Liveness and integrity monitoring.** The most dangerous streaming failure is not disconnection (which immediately errors) but a **silent stall**: a connection exists but data stops. SDK timeouts often cover connection establishment only, not transmission. Use an independent idle watchdog: if no output arrives within the configured period, kill the hanging stream and retry. More generally, **every long-lived connection needs a liveness signal, not just a connection timeout**. Integrity monitoring targets trajectory structure: automatically repair missing call-result pairs before context injection rather than exposing structural errors to the model/user. Product mode may tolerate placeholder repair; training-data collection must reject repair because synthetic placeholders contaminate data.

**Recovery: escalate in stages, transparently at each stage.** Do not escalate if a lower level suffices:

1. **Silent retry**—default for retryable errors. Exponential backoff plus random jitter avoids synchronized client retries and renewed congestion; respect server wait hints. Distinguish foreground/background: retry the main loop, but abandon auxiliary calls such as title generation or input suggestions, whose retries can consume main-path quotas and cause "retry amplification."
2. **Fallback and continuation**—if retries fail, change the request. At output limits, silently raise the limit and resend first; if still insufficient, append a meta-instruction to continue from the break. Fall back to another model when the main model overloads (strip the old model's private-format blocks first or the new one cannot parse history); temporarily revert expensive modes to standard mode under rate limits.
3. **Expose to the user**—only after automatic options are exhausted, with the recovery actions already attempted.

**Tool errors take another path: keep the session alive and make errors model input.** Return a structured "tool not found" error for hallucinated calls, input constraints for validation failures, and programmatically repair malformed parameters before execution (for example, a string instead of an object). Errors enter context as ordinary tool results for next-turn self-correction; more specific feedback improves success.

**The error-handling boundary is the full recovery loop, not one request.** Until failure is confirmed unrecoverable, withhold intermediate errors from consumers (users or downstream event subscribers). Successful recovery remains invisible; expose errors together only after all strategies fail.

**Cross-provider takeover: portable text, nonportable attestations.** Continuing a trajectory with another model is obstructed less by endpoint differences than provider-specific history. Calls/results differ structurally but have the same semantics and can be rerendered. Reasoning is harder: it combines readable text and provider attestations proving authorship. Another provider can read the text, but the attestation becomes invalid. Attestations may attach to calls rather than reasoning (Google's thoughtSignature), so "delete all reasoning and it is safe" can fail with strict validators. Design for the strictest endpoint: rewrite historical calls as prose; the model no longer treats them as genuine past calls, but can continue. Therefore **store trajectories in a neutral format**: split reasoning into portable-text and nonportable-attestation slots, store only call names/arguments, regenerate IDs for the target provider when rendering requests, discard attestations on switching, and carry text as ordinary content. Neutral trajectories also support evaluation replay, training-sample construction, and experience extraction.

**Termination: every recovery path needs a limit.** Recovery itself can fail: abandon compression after several consecutive failures, fall back to asking humans after repeated permission-classification failures, and cap continuation rounds. Derive thresholds from production data, not guesswork. Claude Code's "3 consecutive failures" compression threshold came from real-session statistics: some sessions failed over three thousand times in a row; this single retry category wasted about 250,000 API calls globally each day. Three was the empirical turning point between "most failures already recover" and "further retries are largely hopeless."

**Death-spiral protection.** Error handling that calls an LLM can trigger further failures and cascades. Real example: context overflow triggered an "automatically commit on exit" hook; the hook called an LLM for a commit message, overflowed again, and retriggered itself. Two defenses: disable all model-calling side effects on error paths (even at the cost of auxiliary functions such as automatic memory extraction), and use a recursion-depth counter to detect/break remaining cascades. Add global termination conditions: maximum iterations, session budget, and human escalation after consecutive failures exceed threshold.

**Three streaming breakpoints and three recovery methods.** Breaks occur during reasoning, body text, or call arguments. Methods: ① discard partial output and resend the entire turn (most expensive, most robust); ② pass partial output as the last assistant message and request continuation (natively supported by some providers; otherwise explicitly mark the message to continue; if unsupported, use the next method); ③ append a meta-instruction to continue from the breakpoint. Partial calls cannot be returned in native structure: convert them to text, complete them, then reparse/validate the concatenation. If any tools already ran through streamed early execution, deduplicate by call fingerprint before continuation to avoid repeated side effects. Joins may introduce spaces or repeated characters—valid parameters are not necessarily semantically correct.

## Practical Patterns

**Build an error → strategy map** (a fundamental improvement over "retry on error"):

1. Catch exceptions → classify as API/tool/context/control flow.
2. Retryable? → silent exponential-backoff + jitter retries (retry foreground; abandon background).
3. Can changing the request help? → raise output limit / append continuation meta-instruction / use backup model / standard mode.
4. Tool errors → structured results back into context for next-turn correction.
5. Persistent failure → consult that path's consecutive-failure counter and trip the breaker above threshold (derived from production logs).
6. Everything failed → expose to the user together with attempted recovery actions.

**Streaming recovery checklist**: record breakpoint type (reasoning/body/arguments); convert partial calls to text + fingerprint deduplication; reparse/validate concatenated parameters; compare token cost and recovery success across the three methods to choose a default.

**Takeover checklist**: neutral trajectory storage (text/attestations separate); render per target provider; discard attestations; rewrite historical calls as prose for providers with mandatory attestation checks; monitor repeated-call fingerprints after switching.

## Common Pitfalls
- Blind retries: invalid parameters or permissions never improve with unchanged retries.
- Connection timeout alone misses silent stalls; use an idle watchdog.
- Retrying auxiliary background calls amplifies retries and consumes main-path quotas.
- Exposing intermediate recovery errors to users/downstream subscribers.
- No circuit-breaker cap: thousands of consecutive failures waste enormous API volume.
- LLM calls inside error handling create death spirals/cascades.
- Assuming deleting reasoning makes cross-provider takeover safe; attestations can attach to calls.
- Continuing partial outputs without fingerprint deduplication repeats executed tools' side effects.
- Repairing missing training-data messages with placeholders contaminates training data.

## Companion Code
- `chapter5/provider-failover/` — Experiments 5-1/5-2: neutral-format cross-provider takeover (direct/stripped/neutral three-arm comparison) and continuation at three streaming breakpoints.
- `chapter5/log-diagnosis/` — full diagnostic loop: read trajectories, identify causes, generate regression tests, verify with real replay, and create an Issue.
- `chapter5/adaptive-log-parser/` — self-healing loop: give failed parse samples to an agent to generate code, test, and hot-update rather than reporting failure.

## Further Reading
- `book/chapter5.md`: "Failure and Error Recovery."
- `book/chapter5.md`: "Coding Agent" → "Implementation Techniques" (parallel calls, streamed execution, cascading aborts).
- `book/chapter1.md`: "Harness Engineering: Competitive Strength Beyond the Model" (correction and the principle of not exposing intermediate states).
