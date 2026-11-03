# Companion Skills for *AI Agent*

The book's 10 chapters are distilled into 22 development skills. Each skill is a package of practical knowledge loaded on demand: when a development task matches its trigger conditions, it provides decision frameworks, practical patterns, and entry points to the book's experimental code.

This collection translates the [original Chinese skills](../skills/README.md). Each English skill links to its Chinese source for comparison. The Chinese collection remains available in `skills/`.

## Installation

Run these commands from the repository root. Choose either the Chinese or English collection: both use the same skill names and installation destinations.

```bash
# Option 1: copy to the user-level skill directory (available across projects)
mkdir -p ~/.claude/skills
cp -R skills-en/*/ ~/.claude/skills/

# Option 2: symlink to follow repository updates (recommended)
mkdir -p ~/.claude/skills
for d in skills-en/*/; do ln -sfn "$(pwd)/${d%/}" ~/.claude/skills/"$(basename "$d")"; done
```

## Part One: How to Build an Agent

Code paths in the table are relative to the repository root.

| Skill | When to use | Chapters | Companion code |
|---|---|---|---|
| [context-engineering](context-engineering/SKILL.md) | Design system prompts, context structures, and skills loaded on demand | ch2 | `chapter1/context/`, `chapter2/prompt-engineering/` |
| [kv-cache-design](kv-cache-design/SKILL.md) | Optimize inference latency and cost; diagnose declining cache hit rates | ch2 | `chapter2/kv-cache/` |
| [context-compression](context-compression/SKILL.md) | Context grows too large or decision quality declines over successive turns | ch2 | `chapter2/context-compression/` |
| [agent-state-bar](agent-state-bar/SKILL.md) | An agent loops indefinitely, forgets TODOs, or uses increasing numbers of thinking tokens | ch2 | `chapter2/system-hint/` |
| [memory-system](memory-system/SKILL.md) | Design, implement, or evaluate a user memory system | ch3 | `chapter3/user-memory/`, `chapter3/mem0/`, `chapter3/memobase/` |
| [rag-pipeline](rag-pipeline/SKILL.md) | Build or tune a RAG retrieval pipeline | ch3 | `chapter3/retrieval-pipeline/`, `chapter3/dense-embedding/` |
| [knowledge-org](knowledge-org/SKILL.md) | Organize knowledge beyond flat text chunks, including RAPTOR and GraphRAG | ch3 | `chapter3/structured-index/`, `chapter3/agentic-rag/` |
| [tool-design](tool-design/SKILL.md) | Design agent tools and decide how to expose a capability | ch4 | `chapter4/perception-tools/`, `chapter4/execution-tools/` |
| [tool-discovery](tool-discovery/SKILL.md) | Tool counts grow and definitions fill the context | ch4 | `chapter4/active-tool-discovery/` |
| [mcp-skill-hub](mcp-skill-hub/SKILL.md) | Integrate third-party capabilities; choose between MCP and a Skill Hub | ch4 | `chapter4/DOCKER_DEPLOYMENT.md` |
| [coding-agent-harness](coding-agent-harness/SKILL.md) | Build a coding agent or design its guardrails | ch5 + ch1 | `chapter5/coding-agent/` |
| [error-recovery](error-recovery/SKILL.md) | Recover from failures, interrupted streams, or handoffs between providers | ch5 | `chapter5/provider-failover/` |
| [async-event-agent](async-event-agent/SKILL.md) | Build an asynchronous or event-driven agent that responds to external events | ch6 | `chapter6/async-agent/` |
| [computer-use](computer-use/SKILL.md) | Build a GUI automation agent or implement visual grounding | ch6 | `chapter6/claude-computer-use-native/` |

## Part Two: How to Improve Agent Capabilities

| Skill | When to use | Chapters | Companion code |
|---|---|---|---|
| [agent-evaluation](agent-evaluation/SKILL.md) | Establish an evaluation system; design metrics and environments | ch7 | `chapter7/tau2-bench-eval/`, `chapter7/model-benchmark/` |
| [eval-dataset-design](eval-dataset-design/SKILL.md) | Design evaluation datasets and examine the structure of evaluation tasks | ch7 | `chapter7/tau2-bench-eval/`, `chapter7/android-world/` |
| [post-training-strategy](post-training-strategy/SKILL.md) | Choose among mid-training, SFT, and RL | ch8 | `chapter8/continued-pretraining/`, `chapter8/retool/` |
| [reward-design](reward-design/SKILL.md) | Design reward functions or investigate reward hacking | ch8 | `chapter8/RLVP/`, `chapter8/retool/` |
| [bad-case-to-dpo](bad-case-to-dpo/SKILL.md) | Turn production failure cases into training data | ch8 + ch9 | `chapter8/premature-completion-dpo/` |
| [agent-evolution](agent-evolution/SKILL.md) | Learn continuously from execution experience and build a self-evolution loop | ch9 | `chapter9/self-evolution-eval/`, `chapter9/self-evolving-tools/` |
| [multi-agent-design](multi-agent-design/SKILL.md) | Choose between one agent and multiple agents; design collaboration topology | ch10 | `chapter10/generative-agents/`, `chapter10/parallel-web-research/` |
| [loop-engineering](loop-engineering/SKILL.md) | Address premature completion claims and design verifiers | ch10 | `chapter5/paper-to-ppt/`, `chapter5/video-edit/` |

## Conventions

- Each skill is a self-contained `SKILL.md` file and can be used independently.
- Code and book paths inside skills are relative to the repository root, such as `chapter4/active-tool-discovery/` and `book/chapter4.md`, so readers can return to the book and experimental code. The Chinese-source Markdown links are relative to each skill file.
- The content distills the book's decision rules, procedures, and key figures; it is not a collection of excerpts.
- After editing a skill, update the installed copy in `~/.claude/skills/`. Symlinks reflect changes automatically.
- Book references retain the Chinese source paths. The [English book](../book-en/) is also available.
