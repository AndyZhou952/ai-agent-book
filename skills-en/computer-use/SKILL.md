---
name: computer-use
description: >-
  Use when building GUI automation agents that inspect screens and operate desktop or browser software
  with mouse and keyboard. Covers perception-reasoning-action loops, action spaces (GUI/bash/file editing),
  three grounding routes (structured element indexes, Set-of-Mark, direct coordinate prediction), resolution
  scaling, keyframe observation interfaces, world models, and mobile ecosystem barriers.
  Keywords: Computer Use, GUI automation, screenshot, grounding, browser-use, Set-of-Mark, visual localization, coordinate scaling, OSWorld.
---

# Computer Use: GUI Automation Agents

Translated from the [Chinese original](../../skills/computer-use/SKILL.md).

## When to Use

- Let agents operate software without APIs: fill forms, change settings, complete multistep web workflows.
- Evaluate/implement a "screenshot → decision → action" loop and select models, grounding, and frameworks.
- Diagnose inaccurate localization, coordinate offsets, or wrong clicks through grounding/scaling pipelines.
- An agent observes only every 3–5 seconds, missing notifications, videos, popups, or other between-frame events.
- Extend computer use to mobile (Android accessibility services) or assess platform-ban risk.
- Design confirmations/safety boundaries before irreversible actions (delete, purchase, submit).
- Reduce action counts/waits for practical usefulness, not just correctness (compare OSWorld-Human baselines).
- No programming API exists and Selenium-like scripts are difficult to maintain as pages change.
- Run agents on containerized virtual desktops and isolate irreversible effects from real environments.

## Core Principles

- **The core is a perception-reasoning-action loop**: screenshot → multimodal model reasoning + one action → execution layer performs it (mouse move/click/type) → wait for UI response and capture again.
- **Distinguish understanding an interface from completing a task**: screenshot Q&A measures the former (multimodal understanding); the latter needs closed-loop understanding/actions, handling loads, state changes, mistakes, and irreversible consequences. After every action, **reconfirm that reality still matches the plan**.
- **Three action-space categories**: GUI (mouse move/click/drag/scroll, keyboard text/shortcuts/holds, screenshots/waits), persistent bash sessions (retain environment across calls; sentinel strings detect completion), and string-matching file edits (more precise than full-file overwrite, reversible).
- **Action spaces are not provider-private protocols**: if the harness translates screenshots, action constraints, and results into the target model's messages/structured outputs, Claude, open-weight vision models, and self-hosted endpoints can drive the same loop.
- **Prefer turning grounding into multiple choice**: number elements; the model returns an ID; the system calculates center coordinates. Choosing is easier than filling blanks—"Click [123]" instead of "Click (350, 464)." Choose among three routes by interface:
  - **Structured element index** (DOM / Accessibility Tree, browser-use): first choice when structure is available; most precise/stable, without segmentation false positives/negatives.
  - **Set-of-Mark visual annotations**: segment candidate regions and overlay numbers; no DOM required, suitable for native desktops, games, Canvas/WebGL.
  - **Direct coordinate prediction** (SeeClick, Claude computer use): GUI screenshot-coordinate-trained models avoid annotation, but are less accurate on small elements/dense interfaces.
- **Coordinate prediction requires resolution management**: coordinate understanding depends on training resolution; mismatch causes systematic offsets. Match target resolutions by **aspect ratio** and scale both ways (for example, a 2560×1440 16:9 screen uses FWXGA 1366×768; predicted (683,384) maps back to about (1280,720)). Never distort with nonuniform scaling.
- **A static screen is a dangerous assumption**: redesign observation rather than actions—keyframe screenshots (a small model detects meaningful changes; about once per second during frequent change), volume-gated transcription, and persistent one-sentence screenshot descriptions to compress multimodal history.
- **World models enable speculative execution**: predict candidate actions' state differences before acting; continue if reality agrees, stop/replan if it diverges. Predict **checkable state changes** (window, focus, scroll, field contents, loading, permissions, network responses), not photorealistic future screenshots.
- **Ecosystem barriers are structural**: goal-directed agents bypass ads/recommendations and monetization, giving platforms incentives to ban them. This goes beyond CAPTCHA and is hard to reconcile in the short term.
- **Loop timing and scale**: computer use is subsecond-to-second; screens change continuously between frames, so rechecking reality after actions is essential, not optional. Speech (10ms–1s, interruptible) and robotics (milliseconds, preemptible action chunks) pose different timing problems.
- **Structured indexes save errors, not tokens**: all interactive elements/properties cost tokens, but buy stable precision and avoid segmentation errors—a worthwhile tradeoff.
- **browser-use's four steps**: obtain DOM/accessibility via CDP → detect interactive elements → assign unique IDs/draw screenshot bounding boxes → generate a synchronized "ID → element description" list. The model returns only IDs; execution uses centers.

## Practical Patterns

1. **Build the loop**: the harness captures screenshots, translates action constraints into target-model messages/structured outputs, and returns execution results. Swap drivers (Claude, open-weight VLMs, self-hosted vLLM/SGLang) while retaining actual endpoints/model IDs to distinguish experiment groups.
2. **Select grounding**: ask whether structured information is available. Web: DOM/a11y indexing (CDP DOM → interactive-element detection → unique IDs/bounding boxes → synchronized ID-to-element list). Native desktops, Canvas/WebGL, games: visual annotations or coordinates. Prefer direct coordinates only for GUI-grounding-trained models.
3. **Implement scaling**: enumerate supported resolution tiers, match aspect ratio, scale screenshots proportionally, and map action coordinates back to physical resolution.
4. **One action per step**: retain screenshots/action traces for replay/acceptance; use `wait` for loading rather than blind clicks.
5. **Safety boundaries**: read-only by default; do not click CAPTCHAs, log in, submit, purchase, or change external data. Before irreversible actions, predict in an isolated virtual desktop whether confirmation appears; request user confirmation when necessary. Do not persist credentials; final screenshots must substantiate conclusions.
6. **Predefine recovery**: for CAPTCHAs, permission popups, or 404s, follow read-only recovery instructions toward reachable paths (such as public JSON data), rather than repeated retries or challenge clicks.
6. **Improve observation**: keyframes + volume-gated ASR + text descriptions support awareness of popups, video, and meeting speech, while retaining text traces after source images leave context.
7. **Mobile adaptation**: replace mouse/keyboard with accessibility APIs (such as Android AccessibilityService) for reading/writing elements, clicks, and input. Coordinate semantics become gesture-type boundaries (tap/long-press/swipe start).
8. **Measure efficiency for acceptance**: human-level accuracy does not imply usefulness. Compare human action counts/waits, include step count and total time in acceptance metrics, and retain real evidence (screenshot hashes, action traces).

## Common Pitfalls

- Treating correct screenshot answers as completion: no loop/recheck means clicking old coordinates after navigation.
- Nonuniform scaling: squeezing 16:9 into 4:3 distorts the screen and systematically shifts localization.
- Using training-resolution coordinates directly without conversion.
- Bare coordinates for dense interfaces/small elements: all three routes can fail here; prefer numbered choices.
- Segmentation false positives/negatives: incorrect candidate regions leave only incorrect options.
- Multiple actions per step obscure intermediate changes and failure attribution.
- Assuming a static screen misses between-frame notifications, video, and sound, producing a false world view.
- Ignoring platform bans/CAPTCHAs: consumer accounts may be restricted, a harder problem than technical failures.
- Accuracy without efficiency: successful tasks with far more actions/waits than humans remain impractical.
- Mixing model runs into one group: provider and open-weight models are separate groups; log endpoints, IDs, and evidence separately.

## Companion Code

- `chapter6/claude-computer-use-native/` — Experiment 6-8 (Anthropic-native): official Computer Use Demo in a containerized Ubuntu desktop; real runs/deterministic acceptance for native `computer` / `bash` / editor tools.
- `chapter6/computer-use-open-model/` — Experiments 6-8/6-9 (open-model): the same "screenshot → structured action → browser execution" loop; Qwen3-VL 32B Instruct drives browser-use by default, supporting OpenRouter or self-hosted vLLM/SGLang. Includes endpoint qualification contracts (accept screenshots, emit Browser Use action schemas, one action per step, no silent model substitution).
- Both companions retain real evidence (screenshot hashes, traces, deterministic acceptance) as templates for evidence gates. The native route also records a complete Google reCAPTCHA encounter and read-only recovery to public weather JSON without clicking the challenge.
- Speech (another timing extension): `chapter6/end-to-end-speech/`, `chapter6/streaming-speech/`, `chapter6/controllable-tts/`, `chapter6/live-audio/`, `chapter6/phone-agent/`.
- Robotics (screen pixels to physical sensors): `chapter6/gemini-xlerobot-navigation/`, `chapter6/rgb-sim2real-grasping/`, `chapter6/xlerobot-teleoperation/`.
- For mobile benchmarks/evaluation, see Chapter 7's AndroidWorld section; `chapter6/phone-agent/` gives an agent-driven real-phone example.

## Further Reading

- `book/chapter6.md`: "Expanding Modalities and Trigger Timing."
- `book/chapter6.md`: "Computer Use: GUI Automation Agents."
- Experiment records/evidence-gate templates: README and `validation/` directories in `chapter6/computer-use-open-model/` and `chapter6/claude-computer-use-native/`.
