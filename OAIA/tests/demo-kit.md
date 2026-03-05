## Phase 0 — Codebase Analysis

### Stack Summary
```
┌──────────────┬───────────────────────────────────────┐
│    Aspect    │                Details                │
├──────────────┼───────────────────────────────────────┤
│ Language     │ TypeScript 5.9.3                      │
├──────────────┼───────────────────────────────────────┤
│ Framework    │ Next.js 16.1.6 (App Router), React 19 │
├──────────────┼───────────────────────────────────────┤
│ Backend/BaaS │ Supabase (Postgres + Auth + RLS)      │
├──────────────┼───────────────────────────────────────┤
│ Styling      │ Tailwind CSS v4, Radix UI             │
├──────────────┼───────────────────────────────────────┤
│ Testing      │ Vitest 4.0 (unit + integration)       │
├──────────────┼───────────────────────────────────────┤
│ Deployment   │ Vercel                                │
└──────────────┴───────────────────────────────────────┘
```
Existing Logging: Effectively None

There is no logging library installed — no pino, winston, bunyan, or any structured logging package. What exists is exactly 5 console.* calls across theentirecodebase:
```
┌─────────────────────────────────────────────────────────────┬──────┬──────────────────────────────────────────────────────────────────────────┬────────────────────────┐
│                            File                             │ Line │                                   Call                                   │       Whatitdoes       │
├─────────────────────────────────────────────────────────────┼──────┼──────────────────────────────────────────────────────────────────────────┼────────────────────────┤
│ src/app/api/demos/route.ts                                  │ 59   │ console.error("Demo insert failed:", demoError)                          │ LogsDBinsert failure   │
├─────────────────────────────────────────────────────────────┼──────┼──────────────────────────────────────────────────────────────────────────┼────────────────────────┤
│ src/app/api/events/route.ts                                 │ 74   │ console.error("Event insert failed:", eventError)                        │ LogsDBinsert failure   │
├─────────────────────────────────────────────────────────────┼──────┼──────────────────────────────────────────────────────────────────────────┼────────────────────────┤
│ src/app/api/events/[id]/submissions/[submissionId]/route.ts │ 106  │ console.error("Failed to write submission review:", reviewError.message) │ Logsauditwrite failure │
├─────────────────────────────────────────────────────────────┼──────┼──────────────────────────────────────────────────────────────────────────┼────────────────────────┤
│ vitest.integration.config.ts                                │ 9    │ console.warn(...)                                                        │ Testconfigwarning      │
├─────────────────────────────────────────────────────────────┼──────┼──────────────────────────────────────────────────────────────────────────┼────────────────────────┤
│ src/tests/integration/helpers/users.ts                      │ 74   │ console.warn(...)                                                        │ Testcleanupwarning     │
└─────────────────────────────────────────────────────────────┴──────┴──────────────────────────────────────────────────────────────────────────┴────────────────────────┘
```

These are unstructured strings going to stdout. No JSON, no log levels, no request context, no user identity. On Vercel, these end up as flat text infunctionlogs — searchable only by substring, not
queryable by field.

Request Tracking: None

There is no request_id generation or propagation anywhere. The middleware (src/proxy.ts → src/lib/supabase/middleware.ts) handles session refresh androuteprotection only. No headers are injected, no
correlation IDs exist. If two users hit the same API route simultaneously, their error logs are indistinguishable.

LLM/AI Call Sites: Zero

There are no LLM or AI API calls in this codebase. No OpenAI, Anthropic, Google AI, LangChain, Vercel AI SDK, or any similar dependency. The package.jsonhaszero AI-related packages. The only external
API calls are to the GitHub REST API in src/lib/github.ts — validateGitHubRepo() makes 1-2 fetch calls per invocation (repo metadata + languages).

Patterns Relevant to Instrumentation

API route structure (consistent): All 8 route handlers follow the same pattern:

1. Create Supabase client → authenticate user → check role/permission
2. Parse request body (with try/catch returning 400)
3. Execute DB operations → return result or error

This consistency is great news — it means a single instrumentation pattern can cover all routes.
Error handling — mostly good, with notable gaps:

- DB errors in main flows are properly checked and returned as HTTP errors, but only 3 of ~15 error paths include a console.error. The rest returnerrorresponses silently — you'd see the HTTP 500 in

Vercel's request log but have no idea what Supabase error caused it.

- One catch-and-silence anti-pattern worth flagging:
- src/lib/github.ts:133 and src/lib/github.ts:159 — Both catch blocks in validateGitHubRepo() catch network errors and return user-friendly error objects,butlog nothing. If GitHub API calls start
failing due to a configuration change, DNS issue, or rate limiting pattern, you'd only know if users reported it. There's no server-side record thatthesefailures are happening or how frequently.
- src/lib/github.ts:159 is particularly notable — the language fetch failure is completely swallowed (returns null). This is labeled "Non-fatal" which istrue,but if it starts failing 100% of the time,
 you'd never know.
- Fire-and-forget DB writes:
  - src/app/api/demos/route.ts:67-72 — Team membership insert after demo creation: the await runs but the error is never checked. If this fails, the demo exists but the creator isn't on their own team.
  - src/app/api/events/route.ts:82-86 — Same pattern for event membership.
  - src/app/api/events/[id]/submissions/[submissionId]/route.ts:93-107 — The audit record write to submission_reviews logs the error but continues execution — the review succeeds but the audit trail has
 a gap. This is the right call (don't fail the user's action over an audit write), but without alerting on this error, you'd accumulate audit gaps silently.

### Assessment

Logging maturity: Essentially zero. This is a greenfield logging buildout. There's no logging library, no structured output, no request tracing, andnosystematic error capture. The 3 console.error
calls are ad-hoc breadcrumbs, not an instrumentation strategy.
However — and this is the critical finding — there are no LLM calls to instrument. The highest-priority target described in your prompt (instrumentingeveryLLM/AI model call) doesn't apply to this
codebase as it stands today. Your external API surface is the GitHub REST API only.
Proposed Scope
Given this is a full buildout from scratch with no LLM calls to instrument, I'd propose reframing the scope:

1. Structured logging foundation — Install a proper logging library, establish JSON log format, add request_id generation in middleware
2. API route instrumentation — Instrument all route handlers with consistent structured logs (request in, response out, errors, latency, user identity)
3. External API call instrumentation — GitHub API calls get the same treatment that LLM calls would: request/response logging, latency, error rates
4. Eliminate silent failures — Fix fire-and-forget patterns and catch-and-silence patterns
5. LLM-ready schema — Design the log schema so that when you do add AI features (step 5-11 of your build order are still ahead), the instrumentation patternisalready there — just extend it

Before I proceed to Phase 1 questions — does this analysis match your understanding of the codebase? Do you want to correct anything, or should I proceedwiththe questions to shape the implementation?
