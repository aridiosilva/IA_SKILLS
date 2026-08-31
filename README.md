# What a SKILL is in an LLM

[English](README.md) | [Português](README.pt.md)

A SKILL in an LLM is a pluggable capability package that turns the model from a text generator into an agent that acts.

The base model — the weights — knows a lot, but by default it only knows how to talk. It doesn't know how to build a slide, search your emails, generate a transparent image, or schedule a task. The SKILL is the harness layer — the hard, engineering-heavy shell around the model — that teaches it HOW to act.

Think of it this way:

Model = brain. SKILL = professional training + toolbox + procedure manual.

Without skills, every LLM is a generalist. With skills, it becomes a specialist on demand.

## 1. What a SKILL actually is

Technically, a SKILL is not a prompt. It's a software artifact made of 4 parts:

1. **Manifest**: Name, description, and when it should be activated. This is what the orchestrator reads to decide "does this question need skill X?". Example: shopping → the description says "Use when the user wants to find, buy, or compare products..."

2. **Specialized instructions**: A high-priority system prompt that is injected ONLY when the skill is loaded. It overrides generic behavior. It spells out the rules, the workflow, the mistakes to avoid, and the output format. It's far more detailed than a normal prompt.

3. **Authorized tools**: Which functions it's allowed to call. A gmail-search skill can call Gmail search. A slides skill can call a presentation generator. A transparent-background-image skill can call image_gen with specific parameters for RGBA PNG.

4. **Schemas and examples**: How to interpret intent and how to format the final response. Example: a places skill must ALWAYS return a place_id as a chip, not just plain text.

## 2. Core concepts around it

**a) Harness vs. Weights**

The weights are "hard" because they're expensive to train and immutable in production. The harness is "hard" because it's tested production code, with permissions, that extends what the model can do without needing retraining. A skill is the safest way to scale capability.

**b) Routing and Activation**

The model doesn't load every skill at once — that would blow out the context window and create conflicts. There's a router that reads your question and decides: do I need to load places? shopping? deep-research-report? none of them? Only after that do the skill's instructions come into play.

**c) Skill vs. Tool vs. Plugin**

- **Tool**: an atomic function. Example: web_search(query).
- **Plugin (older term)**: essentially just exposing a tool to the model.
- **Skill**: high-level orchestration. It decides WHICH sequence of tools to call, HOW to reason, HOW to validate, and HOW to present the result. A skill can use 5 different tools in a loop.

Example: the deep-research-report skill uses browser.search + browser.open + synthesis + citations + report formatting. The tool alone wouldn't do any of that.

**d) MCP — Model Context Protocol**

This is the current standard many skills rely on under the hood. It standardizes how the model discovers that a skill exists, what parameters it accepts, and what result it returns. It's the "USB-C" of skills.

**e) State and Memory**

Good skills are stateless by default, but they can read context: your profile, your past conversations, files you've uploaded. A slides skill that builds a deck from your sales history needs this.

## 3. Why this matters so much

- **Reliability**: Without a skill, the LLM improvises how to search Gmail. With a skill, it follows a validated procedure.
- **Security and Permissions**: The skill defines the boundaries of what can be done. An ads-audiences-pages skill will never delete your account, because it simply doesn't have that tool.
- **Composability**: You can chain skills together. "Research my competitors (rival-watch-2) and build a pitch deck (slides)." The system loads one, then the other.
- **Evolution without retraining**: Want the AI to learn how to make OpenTable reservations? You don't retrain a 400B-parameter Llama. You build the opentable skill and plug it in.

In short: if the LLM is a very intelligent actor, the SKILL is the script, the costume, and the set that let it actually perform a specific role and deliver a useful result — not just a good improvisation.
