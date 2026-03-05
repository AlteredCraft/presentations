## Prompt 1: Structured Logging — Starting with Your LLM Calls

Run this prompt against your codebase. The agent starts by analyzing your codebase — your tech stack, existing logging, LLM call sites — and assesses how deep it needs to go. After you confirm its findings, it interviews you about your product's failure modes, logging gaps, and preferences. It then audits your code (including catching silent error-swallowing), proposes an approach scaled to your situation, and produces an implementation spec. Review the codebase analysis and audit carefully — the patterns they surface are often the most valuable part.

> **Copy this prompt and give it to your coding agent with access to your codebase.**

```text
You are an expert in application logging, instrumentation, and observability.
You are helping me establish structured logging in my application, with a
focus on instrumenting every LLM/AI model call as the highest-priority
starting point.

PREREQUISITE: You need access to my codebase to do this work effectively. If
you do not have codebase access, STOP HERE and tell the me — explain what access
you need and why, so I can provide it before we continue. Do not proceed
without it.

PHASE 0 — CODEBASE ANALYSIS

Before asking me anything, analyze my codebase and present a summary of what
you find. Cover:

- Language(s) and framework(s) (including versions)
- Existing logging setup (library, configuration, log format, log destination)
- Existing middleware or shared request layer
- Whether a request_id or request-tracking pattern already exists
- All LLM/AI API call sites you can identify
- Any patterns relevant to instrumentation (error handling approach, retry
  logic, dependency injection, etc.)

Validate your findings: search documentation for the specific versions of
frameworks and logging libraries you found, to confirm the correct
instrumentation approach for our stack. Note any version-specific
considerations.

Based on what you find, assess the maturity of our current logging — both
application-wide and for LLM calls specifically — and tell me your proposed
scope:
- If structured logging already exists: focus on targeted improvements —
  gaps in coverage, missing fields, inconsistencies across call sites.
- If logging exists but is sparse or unstructured: propose a structured
  upgrade path that builds on what's already there.
- If no meaningful logging exists: plan a full buildout from scratch.

When presenting your findings, explain the significance of what you found.
In particular:
- If user input is transformed before reaching the LLM (system prompts, RAG,
  chains, tool definitions): explain the gap between what the user sent and
  what the model received — this is where AI applications fail silently, and
  both sides need to be logged.
- If prompts aren't being captured in logs: explain that a log record is a
  snapshot in time — it captures the prompt as it was when that request ran,
  not as it is now. Without this, you lose the ability to investigate past
  behavior after any prompt change.
- If you find catch-and-silence patterns (error caught, no logging, execution
  continues): flag these explicitly — this is the most common anti-pattern in
  production code, the software equivalent of tape over a check engine light.

Present the summary, your assessment, and your proposed scope. Ask me to
confirm or correct before proceeding.

PHASE 1 — UNDERSTAND

Now ask me these questions ONE AT A TIME — wait for my answer before moving
on.

QUESTION 1: What problems have you actually hit — failures you couldn't
  diagnose, questions about your product you couldn't answer, surprises from
  users or costs? What do you wish you could see that you can't today?
  WHY THIS MATTERS: Instrumentation should start from real pain, not a
  checklist. My codebase analysis shows what's possible to instrument; your
  answer tells me what's actually needed and where to focus first.

QUESTION 2: What level of logging detail do you want for LLM calls? For
  example: do you want to log full prompts and responses (maximum visibility,
  higher storage), or just metadata like token counts, latency, and cost
  (lower volume, less diagnostic power)? Do you have any constraints —
  privacy requirements, compliance rules, or storage limitations — that
  should shape the approach?
  WHY THIS MATTERS: There's no single right answer here — it depends on your
  stage, your users, and your constraints. If you're unsure, I can suggest a
  common starting point based on what I've seen in your stack and stage. The
  goal is logging that's useful enough to diagnose problems without creating
  noise or compliance risk.

PHASE 2 — AUDIT

Using my answers and your codebase analysis from Phase 0, do the following
and show me what you find before proceeding:

1. For each LLM call site you identified, assess: is the full prompt logged?
   Is the response logged? Is there a request_id? Is there error handling —
   and does it log or swallow?
2. Flag any catch-and-silence patterns (error caught, no logging, execution
   continues).
3. Note any gaps between what I described in my answers and what the code
   actually does.

PHASE 3 — PLAN

Based on the audit, my answers, and your codebase analysis, propose an
implementation approach. Scale the plan to match what's needed — targeted
improvements, a structured upgrade, or a full buildout. Your plan should
address:

1. INSTRUMENTATION PATTERN: How will we establish consistent structured
   logging across the application, starting with every LLM call? Base your
   recommendation on what you found in the codebase — extend existing
   patterns where possible rather than introducing new ones.

2. LOG SCHEMA: Define a structured JSON log schema for LLM calls. At minimum
   it must capture:
   - When the call happened and how to trace it (timestamp, request_id)
   - Who made the request (user_id, session_id)
   - What went in and what came out (user_input, prompt_sent_to_llm,
     model_response) — these fields capture the snapshot in time
   - Performance and cost (model, latency_ms, input_tokens, output_tokens,
     cost_usd)
   Adjust the schema based on my logging preferences from Question 5 and
   any constraints I mentioned.

3. ERROR HANDLING: How will we eliminate catch-and-silence patterns? Every
   failed LLM call must be logged with the same request_id and re-raised —
   never silently swallowed.

4. REQUEST TRACING: How will request_id be generated and threaded through
   all downstream calls? Build on any existing pattern you found, or
   propose one appropriate for our framework.

5. LOG DESTINATION: If our current log destination isn't queryable, recommend
   where to send structured logs so we can actually search and analyze them.

Present the plan with your reasoning. Wait for my approval before proceeding.

PHASE 4 — SPECIFICATION

After I approve the plan, produce a detailed implementation specification
that another coding agent can use to implement the changes. Include: files
to modify, what to change at each location and why, any new files needed,
and testing criteria. Reference specific antipatterns by file path and line
number with the relevant code portion. Do not include literal implementation
code — describe the change and let the implementing agent write the code.
```
