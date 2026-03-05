# Observability in AI applications

### Observability Prompts for AI Products

Three copy-paste prompts for any AI coding agent — Claude, ChatGPT, Copilot, Cursor, or whatever you use. Each one analyzes your codebase, interviews you about what matters, and produces an implementation spec your team can build from.

These prompts are designed to be run in order. Each builds on the foundation the previous one establishes.

---

## The Prompts

### [Prompt 1: Structured Logging — Starting with Your LLM Calls](prompt-1-structured-logging.md)

Establishes structured logging in your application, focused on instrumenting every LLM/AI call as the highest-priority starting point. The agent analyzes your codebase first — assessing your tech stack, existing logging maturity, and LLM call sites — then adapts its scope to what you actually need: targeted improvements if logging exists, a structured upgrade if it's sparse, or a full buildout from scratch.

### [Prompt 2: Customer Journey Event Instrumentation](prompt-2-journey-events.md)

Adds customer journey event tracking — the moments where users succeed, struggle, or decide to leave. The agent finds your existing analytics and event infrastructure, then interviews you about the three moments that matter most in your product. It designs event schemas tied to your application logs via request_id, turning isolated data points into knowledge you can use to reconstruct what a user experienced and why.

### [Prompt 3: Per-Workflow Cost Observability](prompt-3-cost-observability.md)

Maps your application's cost structure at the workflow level — which user actions, features, and agent chains are actually expensive. The agent traces LLM call chains, audits retry logic for cost amplification risks, and compares your cost perception to what the code reveals. The result is workflow-level cost visibility and runaway protection.

---

## How They Work

Each prompt follows the same pattern:

1. **Codebase Analysis** — The agent analyzes your codebase before asking you anything. It assesses what exists, validates against framework documentation, and adapts its scope.
2. **Interview** — The agent asks only what it can't determine from code: your experience, your priorities, your constraints.
3. **Audit** — Deep inspection of your codebase, building on what the agent already found. Surfaces gaps between your perception and what the code actually does.
4. **Plan** — Proposes an approach scaled to your situation. Waits for your approval before proceeding.
5. **Specification** — Produces an implementation spec another coding agent can execute. No literal code — just what to change, where, and why.

## Requirements

- **A coding agent with codebase access.** These prompts are designed for AI coding assistants that can read and analyze your project files. Prompt 1 will stop and tell you if access isn't available.
- **A working application.** The prompts assume you have a product with users (or an app approaching that stage). They're framed as "add this to your product," not aspirational.

---

**[Altered Craft](https://alteredcraft.com)** — Production-informed AI product development

Want a second look at your instrumentation approach? Reach out: sam@alteredcraft.com
