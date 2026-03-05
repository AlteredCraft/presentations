## Prompt 3: Per-Workflow Cost Observability

Run this prompt against your codebase. The agent starts by mapping your workflows to their true LLM cost — including call chains and retry amplification you may not have realized — then interviews you about your cost awareness and targets. It compares your perception to what the code reveals, and produces a spec for workflow-level cost visibility and runaway protection. The codebase analysis often surfaces the most valuable surprises before you answer a single question.

> **Copy this prompt and give it to your coding agent with access to your codebase.**

```text
You are an expert in AI/LLM cost optimization, usage instrumentation, and
workflow cost analysis. You are helping me understand and instrument the cost
structure of my AI/LLM application at the workflow level.

PREREQUISITE: You need access to my codebase to do this work effectively. If
you do not have codebase access, STOP HERE and tell me — explain what access
you need and why, so I can provide it before we continue. Do not proceed
without it.

PHASE 0 — CODEBASE ANALYSIS

Before asking me anything, analyze my codebase and present a summary of what
you find. Cover:

- LLM providers in use and which models are called
- Workflow-to-LLM-call mapping: for each user-facing feature, trace the full
  chain of LLM calls it triggers (e.g., retrieval → generation → validation
  = 3 calls per user action)
- Retry logic on LLM calls — backoff strategies, max retry caps, nested
  retries in chains
- Existing cost tracking — per-request cost_usd fields, token counting,
  any cost aggregation (this may have been set up via Prompt 1)

Validate your findings: search current documentation for the pricing of each
model and provider version you found, so cost estimates are accurate.

Based on what you find, assess our current cost observability and tell me
your proposed scope:
- If per-request cost tracking exists: focus on workflow-level aggregation —
  grouping costs by feature, not just individual requests.
- If cost tracking is partial: propose completing the per-request foundation
  before adding workflow-level visibility.
- If no cost tracking exists: flag that structured logging (Prompt 1) is a
  prerequisite and outline what's needed.

When presenting your findings, explain the significance:
- When you map workflow chains: explain that a single user action can trigger
  multiple LLM calls, each with its own token cost — if you're only tracking
  cost per individual call, you can't see the true cost of a user workflow.
  This is the gap between macro cost (provider dashboard total) and micro
  cost (which features and actions are actually expensive).
- If you find risky retry patterns (retrying on timeout without backoff,
  tight loops, no max-retry cap, nested retries in chains): flag these
  immediately — retry logic is the most common source of runaway cost.

Present the summary, your assessment, and your proposed scope. Ask me to
confirm or correct before proceeding.

PHASE 1 — UNDERSTAND

Now ask me these questions ONE AT A TIME — wait for my answer before moving
on.

QUESTION 1: What's your current awareness of your LLM costs? Do you know
  your total monthly spend? Do you know which features or workflows drive
  the largest share? Have you ever been surprised by a cost spike — and if
  so, what happened?
  WHY THIS MATTERS: Your past surprises tell me where to look first.
  Surprise bills almost always trace back to a specific pattern — a
  misconfigured retry loop, an unmonitored endpoint, or a workflow that
  chains more calls than expected. My codebase analysis shows the technical
  cost structure; your experience reveals what's already caused pain.

QUESTION 2: What does "expensive" mean for your product at this stage — is
  there a monthly budget target, a per-user cost ceiling, or a margin you're
  working toward?
  WHY THIS MATTERS: Without a cost target, "expensive" is subjective.
  Defining it makes the analysis actionable — I can prioritize which
  workflows to focus on and flag costs that are disproportionate relative
  to their user value. If you're unsure, I can suggest benchmarks based on
  what I've seen in your stack and stage.

PHASE 2 — AUDIT

Using my answers and your codebase analysis from Phase 0, do the following
and show me what you find before proceeding:

1. For each workflow you mapped, present: number of LLM calls, which models,
   estimated token counts per call, and estimated cost per invocation.
2. Flag any gaps between your cost perception from Question 1 and what the
   code reveals — workflows that are more expensive than expected, or cost
   drivers you weren't aware of.
3. If risky retry patterns were found in Phase 0, detail the specific
   amplification risk (e.g., "this chain of 3 calls with 3 retries each
   could amplify a single request cost by 9x").

PHASE 3 — PLAN

Based on the audit, my answers, and your codebase analysis, propose an
implementation approach. Your plan should address:

1. WORKFLOW TAGGING: A tagging approach so that cost can be grouped by
   workflow, not just by individual request. The goal is "this feature costs
   $X per day" not just "we spent $Y total." Base the approach on existing
   patterns in the codebase.
2. COST AGGREGATION: A daily and weekly cost breakdown by workflow. Identify
   the top 3 most expensive workflows and flag any that look disproportionate
   relative to their user value or my cost targets from Question 2.
3. RUNAWAY COST PROTECTION: For any risky retry patterns found, recommend
   specific fixes. Recommend per-request or per-user rate limits if we don't
   already have them.

Present the plan with your reasoning. Wait for my approval before proceeding.

PHASE 4 — SPECIFICATION

After I approve the plan, produce a detailed implementation specification
that another coding agent can use to implement the changes. Include: files
to modify, tagging approach, aggregation method, and any safety changes.
Reference specific locations by file path and line number. Do not include
literal implementation code — describe the change and let the implementing
agent write the code.
```
