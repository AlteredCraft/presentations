## Prompt 2: Instrument Your Top 3 Customer Journey Events

The agent analyzes your codebase for existing event tracking and session infrastructure, then interviews you about the three moments that matter most (success, struggle, abandonment). It designs event schemas, checks what's already captured vs what's missing, and produces an implementation spec. The codebase analysis often reveals tracking you didn't know you had — or gaps you assumed were covered.

> **Copy this prompt and give it to your coding agent with access to your codebase.**

```text
You are an expert in product analytics, user behavior instrumentation, and
customer journey mapping. You are helping me instrument the key moments in
my product's value stream — the path a user takes from "I have a problem"
to "this product solved it."

PREREQUISITE: You need access to my codebase to do this work effectively. If
you do not have codebase access, STOP HERE and tell me — explain what access
you need and why, so I can provide it before we continue. Do not proceed
without it.

PHASE 0 — CODEBASE ANALYSIS

Before asking me anything, analyze my codebase and present a summary of what
you find. Cover:

- Existing event tracking or analytics infrastructure — any event emission
  patterns, tracking calls, behavioral logging, or custom analytics
- Structured logging state — is there a logging foundation with request_id
  threading? (This may have been set up via Prompt 1.)
- User session management — how sessions are tracked, session IDs, auth flow
- The user-facing workflows you can identify — what paths does the
  application support?

Validate your findings: search documentation for the specific versions of
frameworks and any analytics libraries you found.

Based on what you find, assess our current event instrumentation and tell
me your proposed scope:
- If event tracking exists: focus on gaps — which key moments are missing,
  which events lack context or request_id threading.
- If tracking is sparse or ad-hoc: propose a cohesive, extendable event
  tracking approach that builds on what's there.
- If no meaningful event tracking exists: plan a full buildout, noting
  whether structured logging (Prompt 1) is a prerequisite.

When presenting your findings, explain the significance:
- If events exist but aren't threaded with request_id or session context:
  explain that individual events are data points, but threaded together with
  request_id and session context they become knowledge — you can reconstruct
  what a user experienced and why.
- If you find behavioral signals already being captured (retries, back-
  navigation, rephrasing): note that in AI products, these may signal silent
  output failure, not just UX friction.

Present the summary, your assessment, and your proposed scope. Ask me to
confirm or correct before proceeding.

PHASE 1 — UNDERSTAND

I need to tell you about my product's value stream so you know which moments
matter. Ask me these questions ONE AT A TIME — wait for my answer before
moving on.

QUESTION 1: Walk me through your product's value stream — what does a user
  come to your product to accomplish, and what does the journey look like
  from their first action to the moment they've gotten what they came for?
  WHY THIS MATTERS: Before we can instrument anything, I need to understand
  the path your product is designed to support. The value stream is the
  sequence of moments that matter — each one is a potential instrumentation
  point. Your codebase shows me the technical flow; this tells me the
  human flow.

QUESTION 2: What does success look like for your business at this stage?
  What user behaviors would tell you that your product is working — and
  what would tell you it's not?
  WHY THIS MATTERS: Not every event is worth capturing. The signals that
  matter are the ones tied to your product's success criteria. If you're
  focused on retention, we instrument differently than if you're focused on
  activation or conversion. This question defines what "working" means so
  we can measure it.

PHASE 2 — AUDIT & DESIGN

Using my understanding of your value stream, your success criteria, and your
codebase analysis from Phase 0:

1. Identify the key moments in your value stream that should be instrumented.
   For each one, check whether it's already captured in any form. Show me
   what you find — including gaps between what matters and what the code
   currently captures.
2. For each moment, design a structured JSON event schema. Every event must
   include these base fields:
   - event (descriptive name, e.g., "search_performed", "result_selected")
   - timestamp (ISO 8601)
   - request_id (ties journey events to application log records)
   - user_id
   - session_id
   Plus event-specific fields that capture what matters about that moment.
   The schemas should be cohesive — consistent naming, shared base fields,
   and designed to extend as the product evolves.

Show me the identified moments, the existing event audit, AND the proposed
schemas. Get my approval before moving on.

PHASE 3 — PLAN

Based on the audit, my answers, and your codebase analysis, propose an
implementation approach. Your plan should address:

1. SIGNAL VS NOISE: For each event, should it be (a) logged for batch
   analysis (query later when investigating), or (b) trigger a realtime alert
   or notification? For realtime alerts, describe what action I would take
   when notified — if there's no clear action, it should be batch analysis.
2. INSTRUMENTATION PATTERN: Design a cohesive, extendable event emission
   approach — consistent across the codebase and built to grow as the product
   evolves. Build on existing infrastructure where possible rather than
   introducing new dependencies.

Present the plan with your reasoning. Wait for my approval before proceeding.

PHASE 4 — SPECIFICATION

After I approve the plan, produce a detailed implementation specification
that another coding agent can use to implement the changes. Include: files
to create or modify, the event emission pattern, where each event fires in
the codebase, and testing criteria. Reference specific locations by file path
and line number. Do not include literal implementation code — describe the
change and let the implementing agent write the code.
```
