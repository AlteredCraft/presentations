## Prompt 2: Instrument Your Top 3 Customer Journey Events

> **Copy this prompt and give it to your coding agent with access to your codebase.**

```text
You are an expert in product analytics, user behavior instrumentation, and
customer journey mapping. You are helping me add customer journey event
tracking to my application.

PREREQUISITE: You need access to my codebase to do this work effectively. If
you do not have codebase access, STOP HERE and tell me — explain what access
you need and why, so I can provide it before we continue. Do not proceed
without it.

PHASE 0 — CODEBASE ANALYSIS

Before asking me anything, analyze my codebase and present a summary of what
you find. Cover:

- Existing analytics or event tracking tools (e.g., Mixpanel, Amplitude,
  PostHog, custom solutions)
- Structured logging state — is there a logging foundation with request_id
  threading? (This may have been set up via Prompt 1.)
- Any journey-relevant events already being captured (analytics calls, custom
  events, behavioral logging statements)
- User session management — how sessions are tracked, session IDs, auth flow

Validate your findings: search documentation for the specific versions of
analytics tools and frameworks you found.

Based on what you find, assess our current journey instrumentation and tell
me your proposed scope:
- If event tracking exists: focus on gaps — which key moments are missing,
  which events lack context or request_id threading.
- If tracking is sparse or ad-hoc: propose a consistent event tracking
  approach that builds on what's there.
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

Now ask me these questions ONE AT A TIME — wait for my answer before moving
on.

QUESTION 1: Describe a moment where a user SUCCEEDS in your product — the
  thing they came to do, done. What does that look like?
  (e.g., "user gets an AI-generated answer they act on," "user completes a
  workflow," "user exports a result")
  WHY THIS MATTERS: This is the moment your product delivers value. If you
  can't see when it happens, you can't measure whether your product is
  working.

QUESTION 2: Describe a moment where a user STRUGGLES or repeats themselves.
  Where do they get stuck, rephrase, retry, or hesitate?
  (e.g., "user rephrases the same query," "user clicks back and tries again,"
  "user contacts support")
  WHY THIS MATTERS: In AI products, struggle signals often point to a
  different root cause than in traditional software. A user rephrasing the
  same query might not mean your UI is confusing — it might mean your AI gave
  a confident wrong answer and the user is trying again. We need to capture
  these moments so you can tell which it is.

QUESTION 3: Describe a moment where a user DECIDES TO LEAVE. What's the last
  thing they do before abandoning?
  (e.g., "user closes the tab after seeing results," "user stops mid-workflow,"
  "user doesn't return after first session")
  WHY THIS MATTERS: Abandonment is the hardest signal to capture because the
  user doesn't click a button that says "I'm leaving." Understanding the last
  action before abandonment tells you where to investigate.

PHASE 2 — AUDIT & DESIGN

Using my answers and your codebase analysis from Phase 0:

1. For each of the three moments I described, check whether it's already
   captured in any form. Show me what you find — including gaps between what
   I described and what the code currently captures.
2. For each moment, design a structured JSON event schema. Every event must
   include these base fields:
   - event (descriptive name, e.g., "search_performed", "result_selected")
   - timestamp (ISO 8601)
   - request_id (ties journey events to application log records)
   - user_id
   - session_id
   Plus event-specific fields that capture what matters about that moment
   (e.g., results_returned, time_to_select_ms, last_action_before_abandon).

Show me the existing event audit AND the proposed schemas. Get my approval
before moving on.

PHASE 3 — PLAN

Based on the audit, my answers, and your codebase analysis, propose an
implementation approach. Your plan should address:

1. SIGNAL VS NOISE: For each event, should it be (a) logged for batch
   analysis (query later when investigating), or (b) trigger a realtime alert
   or notification? For realtime alerts, describe what action I would take
   when notified — if there's no clear action, it should be batch analysis.
2. INSTRUMENTATION PATTERN: How will we emit events consistently? Base your
   recommendation on what you found — extend existing analytics infrastructure
   where possible rather than introducing new patterns.

Present the plan with your reasoning. Wait for my approval before proceeding.

PHASE 4 — SPECIFICATION

After I approve the plan, produce a detailed implementation specification
that another coding agent can use to implement the changes. Include: files
to create or modify, the event emission pattern, where each event fires in
the codebase, and testing criteria. Reference specific locations by file path
and line number. Do not include literal implementation code — describe the
change and let the implementing agent write the code.
```
