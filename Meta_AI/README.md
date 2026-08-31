# SKILLs for LLMs - Complete Guide
### What they are, how they work in Meta AI, and how to implement file generation

> Content compiled from the conversation about SKILLs as part of the AI model's harness.

---

## 1. What is a SKILL for LLMs

A **SKILL** in an LLM is a pluggable capability package that transforms the model from a text generator into an agent that acts.

The base model, the weights, knows a lot, but by default it only knows how to *talk*. It doesn't know how to create a slide, how to search your emails, how to generate a transparent image, how to schedule a task. The **SKILL** is the **harness** layer — the hard, engineering part around the model — that teaches it HOW to act.

**Short definition:**

> **Model = brain. SKILL = professional training + toolbox + procedure manual.**

Without skills, every LLM is a generalist. With skills, it becomes a specialist on demand.

### 1.1 Anatomy of a SKILL

Technically, a SKILL is not a prompt. It is a software artifact with 4 parts:

1.  **Manifest (`manifest.yaml`):** Name, description, when it should be activated. This is what the orchestrator reads to decide "does this question need skill X?". Ex: `shopping` -> "Use when user wants to find, buy, compare products..."

2.  **Specialized instructions (`SKILL.md`):** A high-priority system prompt that is injected ONLY when the skill is loaded. It overrides generic behavior. It tells the rules, workflow, errors to avoid, output format.

3.  **Authorized tools:** Which functions it can call. A `gmail-search` skill can call Gmail search. A `slides` skill can call a presentation generator. A `transparent-background-image` skill can call `image_gen` with specific parameters for RGBA PNG.

4.  **Schemas and examples (`templates/` and `examples/`):** How to interpret intent and how to format the final answer.

---

## 2. Basic Concepts Around It

### a) Harness vs. Weights
Weights are "hard" because they are expensive to train and immutable in production. The harness is "hard" because it is production code, tested, with permissions, that extends what the model can do without retraining. Skill is the safest way to scale capability.

Industry-standard formula:

```
Agent = Model + Harness
```

### b) Routing and Activation
The model does not load all skills at once — that would blow up the context window and create conflicts. There is a router that reads your question and decides: `do I need to load local? shopping? deep-research-report? none?`. Only after that do the skill instructions come in.

Flow in Meta AI:
1. Routing: Reads description of all skills
2. Load: Calls `skills.load_skill({"skill_name": "slides"})`
3. Execution: Model is forced to follow the skill protocol

### c) Skill vs. Tool vs. Plugin
*   **Tool:** an atomic function. Ex: `web_search(query)`.
*   **Plugin (legacy):** basically exposing a tool to the model.
*   **Skill:** high-level orchestration. It decides WHICH sequence of tools to call, HOW to think, HOW to validate, and HOW to present. One skill can use 5 different tools in a loop.

Example: the `deep-research-report` skill uses `browser.search` + `browser.open` + synthesis + citations + report formatting.

### d) MCP - Model Context Protocol
It is the current standard that many skills use underneath. It standardizes how the model discovers that a skill exists, what parameters it accepts, and what result it returns. It's the "USB-C" for skills. Created by Anthropic on November 25, 2024 and now an open standard.

### e) State and Memory
Good skills are stateless by default, but can read context: your profile, past conversations, files you uploaded. The `slides` skill that creates a deck about your sales history needs that.

---

## 3. How It Works in Meta AI - Folder Structure

### 3.1 Standard skeleton

```
/skills/
└── skill-name/
    ├── manifest.yaml          # Who I am, when to activate, what tools I can use
    ├── SKILL.md               # Main instruction - 90% of intelligence is here
    ├── handlers/              # Python/TS code that the skill can call
    │   ├── main.py
    │   └── validators.py
    ├── templates/             # Base templates for files
    │   ├── base.html
    │   ├── base.docx
    │   └── style.css
    └── examples/              # Few-shots so model doesn't hallucinate
        └── example_1.md
```

Example `manifest.yaml`:

```yaml
name: xlsx-report
description: Use when user asks to create, build, generate, edit spreadsheet, excel, xlsx, csv
version: 2.1
author: meta
allowed_tools:
  - container.python_execution
  - container.create
  - browser.search
requires_code_execution: true
output_mimetypes:
  - application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
```

### 3.2 The 8 Implemented File Generation Skills

This conversation generated a real structure in `/mnt/data/meta-ai-skills-showcase/` with:

| File type | Skill | Tool the skill calls | What the harness does |
| --- | --- | --- | --- |
| **html** | `html-artifact` | `create_web_artifact_agent` | Spawns a React sub-agent that builds and returns `preview.html` |
| **docx** | `docx-generator` | `python_execution` + `python-docx` | Executes python that writes binary .docx |
| **pdf** | `pdf-generator` | `python_execution` + `reportlab` | Native PDF canvas |
| **png** | `png-transparent` | `image_gen` | Calls generator with `RGBA + transparent background` forced |
| **md** | `md-report` | `container.create` | Direct markdown writing |
| **xlsx** | `xlsx-report` | `python_execution` + `pandas` | `df.to_excel()` |
| **py** | `py-script` | `container.create` | Creates file and tests with `python_execution` |
| **bash** | `bash-task` | `container.create` + `chmod +x` | Creates `.sh` and validates allowed commands |

#### Handler example - docx
```python
from docx import Document
doc = Document()
doc.add_heading('Report Generated by Skill', 1)
doc.add_paragraph('This DOCX was generated by the docx-generator skill.')
doc.save('/mnt/data/output.docx')
```

#### Handler example - xlsx
```python
import pandas as pd
df = pd.DataFrame([{'Product':'A','Sales':100},{'Product':'B','Sales':200}])
df.to_excel('/mnt/data/output.xlsx', index=False)
```

#### Handler example - pdf
```python
from reportlab.lib.pagesizes import A4
from reportlab.pdfgen import canvas
c = canvas.Canvas('/mnt/data/output.pdf', pagesize=A4)
c.drawString(100, 750, 'PDF generated by pdf-generator skill - Meta AI Harness')
c.save()
```

---

## 4. Why Skills Are So Important?

1.  **Reliability:** Without skill, LLM improvises. With skill, it follows validated procedure.
2.  **Security and Permissions:** Skill delimits what can be done via `allowed_tools` and `allowed_commands`.
3.  **Composability:** You can chain skills. "Research competitors (rival-watch-2) and create pitch deck (slides)".
4.  **Evolution without retraining:** Want AI to learn to make reservations? Create `opentable` skill and plug it in, without retraining the 400B model.

> If the LLM is a very smart actor, the SKILL is the script, costume, and set that allow it to actually play a specific role.

---

## 5. Bibliography

### 1. What is a SKILL in LLM
- **Anthropic - Official Skills Documentation:** https://docs.anthropic.com/en/docs/build-with-claude/skills
- **Anthropic launches Skills - The Decoder:** https://the-decoder.com/anthropic-launches-skills-so-claude-can-automatically-pick-prompts-for-specialized-tasks/
- **Anthropic Skills - The Landscape:** https://dev.to/dbolotov/anthropic-skills-the-landscape-for-new-models-and-architectures-2ld3
- **Anthropic makes Skills an open standard - SiliconANGLE:** https://siliconangle.com/2025/12/18/anthropic-makes-agent-skills-open-standard/

### 2. Harness Concept - Agent = Model + Harness
- **Agent Harness - Wikipedia:** https://en.wikipedia.org/wiki/Agent_harness
- **AI Agent Harnesses: The Infrastructure Behind Autonomy - TechTarget:** https://www.techtarget.com/ai/tip/AI-agent-harnesses-The-infrastructure-behind-autonomy
- **Agent = Model + Harness - Towards AI:** https://pub.towardsai.net/agent-model-harness-what-a-coding-agent-harness-actually-is-3149945c26b5
- **What is an Agent Harness? - ODSC:** https://odsc.medium.com/what-is-an-agent-harness-the-architecture-behind-reliable-agentic-ai-76f4c1f243fb

### 3. Model Context Protocol (MCP)
- **Model Context Protocol - Official Site:** https://modelcontextprotocol.io/
- **Model Context Protocol - Wikipedia:** https://en.wikipedia.org/wiki/Model_Context_Protocol
- **MCP Documentation and Spec - GitHub:** https://github.com/barefootford/anthropic-mcp-docs

### 4. Function Calling / Tool Use
- **Tool Calling - OpenAI Platform Docs:** https://platform.openai.com/docs/guides/function-calling
- **Tool Calling - vLLM Docs:** https://docs.vllm.ai/en/v0.7.1/features/tool_calling.html
- **Claude Skill Registry - llm-function-calling:** https://github.com/majiayu000/claude-skill-registry/blob/HEAD/skills/other/other/llm-function-calling/SKILL.md

### 5. File generation
- **python-docx - Docs:** https://python-docx.readthedocs.io/
- **ReportLab - Docs:** https://www.reportlab.com/docs/reportlab-userguide.pdf
- **pandas to_excel:** https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.to_excel.html

---

## 6. How to use this repository

The directory `/mnt/data/meta-ai-skills-showcase/` contains the 8 ready-to-use skills:

```
meta-ai-skills-showcase/
├── html-artifact/
│   ├── manifest.yaml
│   ├── SKILL.md
│   └── handlers/main.py
├── docx-generator/
├── pdf-generator/
├── png-transparent/
├── md-report/
├── xlsx-report/
├── py-script/
└── bash-task/
```

To test, ask: "generate an xlsx spreadsheet", "generate a pdf", "generate a transparent png".

---
*Generated on August 31, 2026 - Meta AI - Conversation about SKILLs as model hardness*
