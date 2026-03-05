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

**❯ yes, that's accurate — proceed with the questions**

⏺ Question 1: What problems have you actually hit — failures you couldn't diagnose, questions about your product you couldn't answer, surprises from users or costs? What do
   you wish you could see that you can't today?

  Why this matters: Your codebase is early-stage with no logging infrastructure, so we have a clean slate. But instrumentation should start from real pain, not a checklist.
   Your answer tells me what to prioritize — whether that's error visibility, usage analytics, performance tracking, or something else. The GitHub API calls and
  fire-and-forget DB writes are obvious blind spots, but you may have encountered problems I can't see from the code alone.

❯ I often seen errors in the server log, but no erros in the UI, so were are having silent failures 

⏺ That tracks with what I found in the code. The fire-and-forget patterns (team membership inserts, audit writes) and the catch-and-silence patterns (GitHub API calls) mean
   operations fail server-side, the user sees "success," and you have no way to know something went wrong — or how often.

  Question 2: What level of logging detail do you want? For example:

  - Minimal: Request metadata only — route, method, status code, latency, user_id, request_id, error messages. Low volume, enough to spot failures and trace requests.
  - Standard: Metadata plus request/response bodies for API routes. You can see what the user sent and what came back. Moderate volume, strong diagnostic power.
  - Full: Everything above plus Supabase query details (table, operation, row counts). Maximum visibility, higher volume.

  Do you have any constraints — privacy requirements, compliance rules, or Vercel log retention/cost limits — that should shape the approach?

  Why this matters: Your main pain point is silent failures, which even minimal logging would solve. But if you also want to diagnose why things fail (bad input? permission
   edge case? Supabase error?), you need request bodies in the logs. The right level depends on your stage and whether you're comfortable with request data in your log
  stream.

**❯ standard (with log levels as appropriate)**

⏺ Good. Standard with log levels gives us enough to diagnose failures without over-logging. Request/response bodies on errors and warnings, metadata-only on success.

  Now I have what I need. Moving to Phase 2 — the audit.

  ---
  Phase 2 — Audit

  I'll assess every API route and external call site against your requirements: is there a request_id? Is there error logging? Are failures silent?

  API Route Audit
  
```
  Route: POST /api/demos
  File: src/app/api/demos/route.ts
  request_id: No
  Request logged: No
  Response logged: No
  Error handling: Partial — DB insert error logged (L59), but team membership insert (L67-72) has no error check at all
  ────────────────────────────────────────
  Route: PATCH /api/demos/[id]
  File: src/app/api/demos/[id]/route.ts
  request_id: No
  Request logged: No
  Response logged: No
  Error handling: Returns errors to client but logs nothing server-side — all 5 error paths are silent
  ────────────────────────────────────────
  Route: DELETE /api/demos/[id]
  File: same file
  request_id: No
  Request logged: No
  Response logged: No
  Error handling: Same — silent error returns
  ────────────────────────────────────────
  Route: POST /api/events
  File: src/app/api/events/route.ts
  request_id: No
  Request logged: No
  Response logged: No
  Error handling: Partial — DB insert error logged (L74), but event membership insert (L82-86) has no error check
  ────────────────────────────────────────
  Route: PATCH /api/events/[id]
  File: src/app/api/events/[id]/route.ts
  request_id: No
  Request logged: No
  Response logged: No
  Error handling: Silent — 4 error paths, zero logging
  ────────────────────────────────────────
  Route: DELETE /api/events/[id]
  File: same file
  request_id: No
  Request logged: No
  Response logged: No
  Error handling: Silent
  ────────────────────────────────────────
  Route: POST /api/events/[id]/submissions
  File: src/app/api/events/[id]/submissions/route.ts
  request_id: No
  Request logged: No
  Response logged: No
  Error handling: Silent
  ────────────────────────────────────────
  Route: GET /api/events/[id]/submissions
  File: same file
  request_id: No
  Request logged: No
  Response logged: No
  Error handling: Silent
  ────────────────────────────────────────
  Route: PATCH .../submissions/[submissionId]
  File: src/app/api/events/[id]/submissions/[submissionId]/route.ts
  request_id: No
  Request logged: No
  Response logged: No
  Error handling: Partial — audit write error logged (L106) but execution continues (correct behavior, but no alerting)
  ────────────────────────────────────────
  Route: DELETE .../submissions/[submissionId]
  File: same file
  request_id: No
  Request logged: No
  Response logged: No
  Error handling: Silent
  ────────────────────────────────────────
  Route: POST .../resubmit
  File: src/app/api/events/[id]/submissions/[submissionId]/resubmit/route.ts
  request_id: No
  Request logged: No
  Response logged: No
  Error handling: Silent
  ────────────────────────────────────────
  Route: POST /api/github/validate
  File: src/app/api/github/validate/route.ts
  request_id: No
  Request logged: No
  Response logged: No
  Error handling: Silent — delegates to validateGitHubRepo() which also logs nothing
```

  External API Call Audit

```
  ┌──────────────────────────┬───────────────────────────┬──────────────┬───────────────────────────────────────────────────────────────────────────────────────────────┐
  │        Call site         │         File:Line         │   What's     │                                        What's missing                                         │
  │                          │                           │    logged    │                                                                                               │
  ├──────────────────────────┼───────────────────────────┼──────────────┼───────────────────────────────────────────────────────────────────────────────────────────────┤
  │ GitHub repo metadata     │ src/lib/github.ts:100-139 │ Nothing      │ Latency, status code, rate limit headers, error details. Network errors caught and swallowed  │
  │ fetch                    │                           │              │ at L133.                                                                                      │
  ├──────────────────────────┼───────────────────────────┼──────────────┼───────────────────────────────────────────────────────────────────────────────────────────────┤
  │ GitHub languages fetch   │ src/lib/github.ts:152-161 │ Nothing      │ Same. Failure completely silent at L159 — returns null, no record it happened.                │
  └──────────────────────────┴───────────────────────────┴──────────────┴───────────────────────────────────────────────────────────────────────────────────────────────┘
```

  Fire-and-Forget Operations (no error check)

```
  ┌───────────────────────────────────┬───────────────────────────────────────────────┬────────────────────────────────────────────────────────────────────────────┐
  │             File:Line             │                   Operation                   │                          Impact of silent failure                          │
  ├───────────────────────────────────┼───────────────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────┤
  │ src/app/api/demos/route.ts:67-72  │ team_memberships.insert after demo creation   │ Demo exists but creator can't edit it — they're not on their own team      │
  ├───────────────────────────────────┼───────────────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────┤
  │ src/app/api/events/route.ts:82-86 │ event_memberships.insert after event creation │ Event exists but creator can't manage it — they're not listed as organizer │
  └───────────────────────────────────┴───────────────────────────────────────────────┴────────────────────────────────────────────────────────────────────────────┘
```

  These two are the most dangerous silent failures in the codebase. The user creates a resource, gets a 201 success, then discovers they can't manage what they just
  created. With no logging, you'd have no idea this is happening.
  
  Catch-and-Silence Patterns

```
  ┌───────────────────────┬─────────────────────────────────┬───────────────────────────────────────────────────────────────────────────────────────┐
  │       File:Line       │          What's caught          │                                     What happens                                      │
  ├───────────────────────┼─────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┤
  │ src/lib/github.ts:133 │ Network error on repo fetch     │ Returns user-friendly error object, logs nothing server-side                          │
  ├───────────────────────┼─────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┤
  │ src/lib/github.ts:159 │ Network error on language fetch │ Sets languages = null, logs nothing, continues                                        │
  ├───────────────────────┼─────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┤
  │ src/app/layout.tsx:55 │ localStorage access error       │ catch (e) {} — empty catch. This one is fine (client-side, SSR guard), not a concern. │
  └───────────────────────┴─────────────────────────────────┴───────────────────────────────────────────────────────────────────────────────────────┘
```

  Gap Summary

  Your stated pain — "errors in the server log but no errors in the UI, so we're having silent failures" — is explained by two patterns in the code:

  1. Operations that don't check for errors at all (fire-and-forget inserts). These fail silently both server-side and client-side.
  2. Operations that return errors to the client as JSON but log nothing server-side. The HTTP response carries the error, but if the client doesn't surface it (or the user
   dismisses it), the failure vanishes.

  Every single route handler lacks: a request_id, a log on entry, a log on exit with latency, and structured error logging. The 3 existing console.error calls are better
  than nothing but are unstructured and missing context (who? which request? how long did it take?).

  ---
  Ready to move to Phase 3 — the implementation plan. Shall I proceed?

**❯ yes, proceed**

⏺ Phase 3 — Implementation Plan                                                                                                                                           
                                                                                                                                                                            
  1. Logging Library: Pino                                                                                                                                                  
                                                                                                                                                                            
  Why pino: It's the standard for structured JSON logging in Node.js/Next.js. It's fast (low overhead in serverless), outputs JSON natively (Vercel parses it               
  automatically), and has first-class support for child loggers (which is how we'll thread request_id). Next.js 16 on Vercel sends stdout to Vercel Logs, so pino's default
  stdout output works with zero transport configuration.

  The alternative would be winston, but it's heavier and slower — pino is the better fit for serverless.

  2. Request Tracing: Middleware-Generated request_id

  How it works: The existing middleware (src/proxy.ts → src/lib/supabase/middleware.ts) already runs on every matched request. We'll generate a crypto.randomUUID() there
  and attach it as a response header (x-request-id).

  Threading it to route handlers: Next.js App Router doesn't have a built-in request context mechanism like Express's req.locals. The cleanest approach for this stack is to
   read the request_id from the request headers in each route handler. The middleware sets it as a request header (via the request.headers rewrite pattern already used for
  cookies), and route handlers extract it. This avoids introducing AsyncLocalStorage or a context library — minimal new machinery.

  3. Logger Architecture

  src/lib/logger.ts          — Creates root pino instance, exports helper to create request-scoped child loggers
  src/lib/supabase/middleware.ts — Generates request_id, attaches to request + response headers

  Root logger: Configured with JSON output, log level from LOG_LEVEL env var (default: "info" in production, "debug" in development).

  Request-scoped child logger: A function like createRequestLogger(request: Request) that extracts request_id from headers and returns a pino child logger with {
  request_id, method, path } bound. Every route handler calls this once at the top.

  4. Log Schema

  Every API route handler produces two log records minimum — one on entry, one on exit:

  Entry log (info level):
```
  {
    "level": "info",
    "msg": "request_start",
    "request_id": "uuid",
    "method": "POST",
    "path": "/api/demos",
    "user_id": "uuid | null",
    "user_role": "builder | organizer | admin | guest | null"
  }
```

  Exit log (info on success, warn on 4xx, error on 5xx):
```json
  {
    "level": "info | warn | error",
    "msg": "request_complete",
    "request_id": "uuid",
    "method": "POST",
    "path": "/api/demos",
    "user_id": "uuid",
    "status": 201,
    "latency_ms": 142,
    "error": null
  }
```

  On error (4xx/5xx), the exit log also includes:
```json
  {
    "error": "Demo insert failed",
    "error_detail": "duplicate key value violates unique constraint...",
    "request_body": { "title": "My Demo", "repo_url": "..." }
  }
```

  Request bodies are logged only on errors — this matches your "standard" preference while keeping success-path logs lean.

  External API call log (GitHub):
```json
  {
    "level": "info | error",
    "msg": "external_api_call",
    "request_id": "uuid",
    "service": "github",
    "endpoint": "/repos/owner/repo",
    "status": 200,
    "latency_ms": 340,
    "rate_limit_remaining": "4832",
    "error": null
  }
```

  This schema is LLM-ready — when you add AI features later, you extend it with model, input_tokens, output_tokens, cost_usd, prompt_sent, model_response fields. The
  request_id and user_id tracing is already in place.

  5. Error Handling Fixes

  Fire-and-forget operations → check errors and log:

  Both demos/route.ts:67-72 (team membership) and events/route.ts:82-86 (event membership) will check the insert result and log errors at error level. These shouldn't block
   the response (the primary resource was created), but the failure must be logged with the request_id so you can detect and investigate the pattern.

  Catch-and-silence → catch-and-log:

  github.ts:133 and github.ts:159 — These catch blocks will log at warn level with the error details, request_id, and the URL being fetched. The return behavior stays the
  same (user-friendly error or null), but now you have a server-side record.

  Submission review audit write ([submissionId]/route.ts:106) — Already logs via console.error, will upgrade to structured error-level log with request_id. This is
  correctly non-blocking, just needs proper instrumentation.

  6. Log Levels Strategy

```
  ┌───────┬────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
  │ Level │                                                            When                                                            │
  ├───────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ error │ 5xx responses, failed DB writes (including fire-and-forget), external API failures that affect functionality               │
  ├───────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ warn  │ 4xx client errors (bad input, forbidden, not found), non-fatal external API failures (language fetch), degraded operations │
  ├───────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ info  │ Request start/complete on success, external API calls that succeed                                                         │
  ├───────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ debug │ Request bodies on success (off by default), detailed auth/permission checks                                                │
  └───────┴────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

  7. Log Destination

  Vercel automatically ingests structured JSON from stdout and makes it searchable in the Vercel dashboard with field-level filtering. Since pino outputs JSON to stdout by
  default, no additional log destination is needed right now.

  If you later need longer retention, alerting, or advanced querying, you can add a Vercel Log Drain to forward to Datadog, Axiom, or similar — but that's a future concern,
   not a blocker.

  8. Changes Summary

```
  ┌──────────────────────────────────────────────────────────────────────┬──────────────────────────────────────────────────────────────────┐
  │                                 File                                 │                              Change                              │
  ├──────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┤
  │ package.json                                                         │ Add pino dependency                                              │
  ├──────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┤
  │ src/lib/logger.ts                                                    │ New file — root pino instance + createRequestLogger() helper     │
  ├──────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┤
  │ src/lib/supabase/middleware.ts                                       │ Generate request_id, attach to request + response headers        │
  ├──────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┤
  │ src/lib/github.ts                                                    │ Accept logger parameter, log external API calls and catch blocks │
  ├──────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┤
  │ src/app/api/demos/route.ts                                           │ Add request logging, fix fire-and-forget, replace console.error  │
  ├──────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┤
  │ src/app/api/demos/[id]/route.ts                                      │ Add request logging for PATCH + DELETE                           │
  ├──────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┤
  │ src/app/api/events/route.ts                                          │ Add request logging, fix fire-and-forget, replace console.error  │
  ├──────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┤
  │ src/app/api/events/[id]/route.ts                                     │ Add request logging for PATCH + DELETE                           │
  ├──────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┤
  │ src/app/api/events/[id]/submissions/route.ts                         │ Add request logging for POST + GET                               │
  ├──────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┤
  │ src/app/api/events/[id]/submissions/[submissionId]/route.ts          │ Add request logging, replace console.error                       │
  ├──────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┤
  │ src/app/api/events/[id]/submissions/[submissionId]/resubmit/route.ts │ Add request logging                                              │
  ├──────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┤
  │ src/app/api/github/validate/route.ts                                 │ Add request logging                                              │
  └──────────────────────────────────────────────────────────────────────┴──────────────────────────────────────────────────────────────────┘
```

  Not changing: Test files, client components, layout files. The catch (e) {} in layout.tsx:55 is a client-side SSR guard for localStorage — not a server-side concern.

  ---
  Does this plan look right? Any adjustments before I produce the implementation specification?
