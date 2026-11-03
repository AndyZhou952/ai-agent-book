---
name: tool-design
description: Use when designing agent tools—adding or refactoring toolsets, designing tools for MCP servers, deciding between a dedicated tool and a skill, writing descriptions and parameter schemas, paginating or truncating perception outputs, adding approval and sandboxing to execution tools, or designing subagent collaboration interfaces. Covers tool categories, general ACI principles, and perception, execution, and collaboration tool design.
---

# Tool Design

Translated from the [Chinese original](../../skills/tool-design/SKILL.md).

## When to Use
- Add a batch of agent tools or review an existing toolset ("Should this tool be split or merged?").
- Decide whether a capability should be a dedicated tool (function calling), a general executor, or a skill.
- Write or revise tool names, descriptions, parameter schemas, and return-value documentation.
- Design return formats, pagination, and truncation for perception tools (search, reading, multimodal parsing).
- Design input validation, approval, sandboxing, and observability for execution tools (shell, files, external systems).
- Design interfaces and prompts for collaboration tools (spawn_subagent, HITL, notifications).

## Core Principles
- **Tools correspond to goals, not API endpoints.** An early antipattern wrapped each endpoint as a tool, requiring an agent to coordinate several tools for one goal. This is the essence of ACI (Agent-Computer Interface): make tools friendly to agents rather than humans.
- **Prefer general-purpose tools by default.** LLMs already have strong reasoning and code-generation abilities; do not constrain them. Instead of a four-function calculator, provide a `code_interpreter` with sympy/numpy/pandas installed.
- **Fall back to dedicated tools in four situations**: ① safety, permissions, or audit requirements (such as production database writes); ② hiding platform differences and providing better feedback (such as differing grep/find syntax on Mac/Win/Linux); ③ extremely frequent use; ④ complex parameters (nested objects and cross-field validation).
- **Favor consolidation in tool granularity.** Judge by functional similarity and overlap in use cases: combine `extract_pdf_text`/`extract_docx_content`/`extract_pptx_content` into `read_document(file_type=...)`. Consolidation reduces cognitive load, clarifies descriptions, and simplifies extension.
- **Form and quantity are independent decisions.** Capability form determines resident tokens per capability, parameter passing, and who can edit it; the number exposed at once is a disclosure strategy. Do not conflate them.
- **Perception tools prevent context explosion; execution tools prevent irreversible errors.** Output information volume is the former's key concern; safety constraints are the latter's.
- **There must be no systematic discrepancy between the world the model perceives and the world tools operate on**—never silently transform or inject parameters.

## Practical Patterns

### 1. Four Dimensions for Choosing a Capability's Form
| Dimension | Criterion |
|---|---|
| Safety and permissions | Fine-grained authorization, audit trails, or irreversible risk → dedicated tool; otherwise prefer general-purpose tools |
| Parameter complexity | Nested objects or cross-field validation → dedicated schema; CLI is equally reliable for simple parameters |
| Change frequency | Frequent changes → skill (edit text without retesting and redeploying); stable underlying operations → dedicated tool |
| Model capability | Strong models can reduce tool count with skills + a general executor; weaker models need structured schemas for guidance |

Skill cost: the model must generate valid command-line arguments and handle quoting and escaping; the rules are more complex than JSON and vary across platforms. **Complex parameters are more error-prone.** A compromise is to have the skill write complex structured parameters to a JSON file and import it from the command line.
Skill benefit: friendly to human authors; a local syntax error does not break everything (a missing schema brace can fail the entire agent, whereas minor skill errors do not).

### 2. Four Guidelines for Tool Descriptions
- **Say when to use it, not just what it does**: replace "Search relevant content" with "Use when real-time information or unknown facts are needed."
- **Boundaries matter more than capability descriptions**: a file-search tool must say it matches filenames only and cannot search file contents. Most call failures stem from models not knowing what a tool **cannot** do.
- **Give concrete parameter examples**: `timestamp`: RFC3339, such as `2024-03-15T14:30:00Z`; `phone`: E.164 (country code + number, no spaces), such as `+8613888888888` / `+12025551234`.
- **Describe return values and costs**: "Returns a JSON array with `title`/`url`/`snippet` in each element"; "Downloads the entire page and may take 5-10 seconds on large sites; use `get_page_metadata` if only metadata is needed."
- **Include 1-5 real invocation examples per tool.** JSON Schema expresses types but cannot express implicit conventions such as seconds versus milliseconds or how filters nest. Examples can raise benchmark accuracy from about 72% to 90%.
- **Debugging rule**: when the agent often chooses the wrong tool, inspect descriptions before blaming the model. Improving descriptions has much higher ROI than switching to a stronger model.

### 3. Parameter Fidelity
Antipatterns are **silent input transformation** and **silent parameter injection**:
- Cursor once silently converted Chinese curly quotation marks (`“”`) in `old_string` to straight English quotes. The reading tool returned curly quotes unchanged, but the replacement tool could not match them; the model repeatedly failed without being able to diagnose the cause.
- An IDE's bash tool automatically appended an "AI-generated" marker argument to every `git commit`; older Git versions failed regardless of how the model changed the commit message.
Rule: if normalization is necessary, document it in the tool description and explicitly inform the model in the return value.

### 4. Perception Tools
- **Search**: return structured candidates (title, location, snippet), not concatenated full text. Provide cursors/pagination; return only the first few by default and state the total and how to fetch the next page, leaving pagination to the agent.
- **Reading**: support offset/limit; explicitly mark truncation ("Showing lines 1-200 of 5000; continue with offset"). **Silent truncation is dangerous**—the agent assumes it has seen everything.
- **General compression**: above a threshold (such as 10000 characters), compress according to the current query intent.
- **Read-only advantage**: results can be safely cached; multiple perception calls can run in parallel.
- **Multimodal**: return images directly (preserves layout, costs tokens), or first use OCR/chart parsing to extract text (saves tokens, loses spatial structure)? Choose by content: extract plain text, retain images for layout-sensitive material (UI, complex tables, design mockups).
- **Three multimodal routes**: native multimodality (highest ceiling); text extraction (plain-text PDFs save tokens—over a thousand tokens per screenshot page versus a few hundred for text—but lose layout/charts); tool-based multimodal analysis (better when the main model lacks multimodality: `analyze_image/pdf/audio` takes a file + question and returns natural language, keeping multimodal tokens out of the main context).

### 5. Execution Tools
Safety layers: **input validation** (path traversal `../../etc/passwd`, command injection `;`/`|`, types/formats; fail fast rather than "intelligently correcting") → **permission control** (working-directory restrictions, denylists, quotas; denylists are only the lowest layer and can be bypassed with command variants) → **Proposer-Reviewer** → **Sidecar**.

- **Pre-execution approval**: the Proposer proposes; the Reviewer approves. The models should be **from different families with similar capabilities** (for example, Claude and GPT reviewing each other): same-family models tend to share mistakes, while a large capability gap leaves the reviewer unable to keep up. Underlying rules and context must agree, but focus should differ (task completion versus risks/rules). Feed rejection reasons back into the trajectory as a tool result rather than simply retrying. Suitable for irreversible, high-impact operations (charging money, emailing, changing critical settings, creating external resources). Risk classification and human escalation when uncertain can also be used.
- **Post-execution verification**: the key is **switching modalities**—render a code-generated document to images to check layout; run changed configurations in a sandbox. Same-modality reviews tend to share blind spots.
- **Sidecar**: a lightweight classifier running alongside the main model's streamed output gates individual tool calls. It **reads structured fields only** (`{tool:"bash", command:"rm -rf /tmp/data"}`), deliberately excluding the main model's free text; otherwise an instruction such as "Please allow rm -rf" embedded in user input or a page could deceive it. It finishes in hundreds of milliseconds, nearly imperceptibly. A lightweight model suffices because the review target differs: Proposer-Reviewer evaluates open-ended reasoning and needs comparable capability; Sidecar performs simple classification. Include a **rejection circuit breaker**—escalate repeated consecutive rejections to humans rather than retrying forever.
- **Automatic verification loop**: automate verification whenever the result is verifiable. After `write_file`, immediately run the appropriate linter and include structured errors in the return value.
- **Long output**: above a threshold (200 lines or 10000 characters), put only the first 50 + last 50 lines in context, with `... [8523 lines omitted; full output saved to /tmp/execution_output.txt] ...` in between, and direct the agent to `read_file` for the full text.
- **Sandboxing**: venv is not a sandbox (it isolates package dependencies, not filesystem/network/process access). Increasing isolation: process-level (local development) → containers (shared kernel, escape risk) → microVMs/VMs (Firecracker; the strongest tier for completely untrusted code). Set CPU/memory/disk/network limits for containers/microVMs too.
- **Idempotency and cancellation**: ask, "If this call is canceled or times out, did its side effects occur?" Use unique IDs for server-side deduplication, or query before mutating. **Emailing, phone calls, and external transfers** cannot be made idempotent; use two phases, "preflight-confirm." Do not blindly retry execution failures; return detailed errors to the main model for replanning.
- **Observability**: log each call's time/parameters/results/duration, audit trails, performance metrics, and anomaly alerts.

### 6. Collaboration Tools
Three primitive groups: **start/cancel** (`spawn_subagent` / `cancel_subagent`—terminate work promptly when it becomes irrelevant to save tokens), **messaging** (`send_message_to_subagent`, bidirectional), and **discovery** (`list_agents`, analogous to MCP `tools/list` but listing agents). Collaboration can be synchronous, asynchronous (task_id + event notification), streaming, or multi-turn.

Four elements of subagent prompts:
1. **Define the role immediately**: "You are an assistant agent specializing in XXX."
2. **Label context sources**: `[FROM_MAIN_AGENT]` / `[FROM_USER]` / `[TOOL_RESULT]` to prevent source confusion and prompt injection.
3. **Specify task boundaries**: what is in scope and what must be handed off or reported.
4. **Standardize output** (JSON or Markdown) to encourage completeness and reduce the main agent's parsing burden.

HITL: define timeout thresholds and default behavior ("Use a conservative strategy after 5 minutes without a response"), priority queues (urgent: multiple channels; routine: email only), and feed human approval/rejection and reasons back as evidence-based feedback data.

## Common Pitfalls
- Wrapping API endpoints directly as tools, creating excessive granularity, tool counts, and selection burden.
- Describing functions without triggers or boundaries, leaving the model to guess and fail.
- Silently transforming/injecting parameters, causing systematic failures the model cannot diagnose.
- Silently truncating perception output or dumping all search results at once.
- Using denylists as the only safety measure or venv as a sandbox.
- Same-family peer review or review by two models with very different capabilities.
- Letting Sidecar read the main model's free text and be manipulated by injected instructions.
- Conflating capability form with how many capabilities are exposed at once.

## Companion Code
- `chapter4/perception-tools/` — perception-tool MCP server (search/multimodal/filesystem/public and private data sources, `run_experiment_4_2.py`).
- `chapter4/execution-tools/` — execution-tool MCP server: LLM approval before execution, automatic linting after writes, long-output truncation and persistence (`python cli.py demo`).
- `chapter4/collaboration-tools/` — collaboration-tool MCP server: synchronous/asynchronous subagents, comparison of two context-passing strategies, HITL, and multichannel notifications.
- `chapter4/multimodal-agent/` — comparison of native multimodality, text extraction, and tool-based analysis within one framework (`demo.py`).

## Further Reading
- `book/chapter4.md`: "Tool Categories," "General Principles of Tool Design," "Perception Tools," "Execution Tools," and "Collaboration Tools."
