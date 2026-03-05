# Structured Logging Implementation Specification

> **Purpose:** This document specifies every change needed to add structured logging to Demo Kit. It is written for a coding agent to implement without ambiguity.
>
> **Date:** 2026-03-05
>
> **Scope:** Full buildout — no logging infrastructure exists today.

---

## Table of Contents

1. [Dependency](#1-dependency)
2. [New File: `src/lib/logger.ts`](#2-new-file-srclibloggerts)
3. [Middleware: Request ID Generation](#3-middleware-request-id-generation)
4. [GitHub API Instrumentation](#4-github-api-instrumentation)
5. [API Route Instrumentation](#5-api-route-instrumentation)
6. [Fire-and-Forget Fixes](#6-fire-and-forget-fixes)
7. [Console.error Replacements](#7-consoleerror-replacements)
8. [Environment Variables](#8-environment-variables)
9. [Log Schema Reference](#9-log-schema-reference)
10. [Testing Criteria](#10-testing-criteria)

---

## 1. Dependency

Install `pino` as a production dependency:

```
npm install pino
```

No other logging packages are needed. Pino outputs JSON to stdout, which Vercel ingests natively. No transports, no log drains, no additional configuration.

---

## 2. New File: `src/lib/logger.ts`

Create this file. It provides two exports:

### `logger` (root pino instance)

- Create a pino instance with these settings:
  - `level`: Read from `process.env.LOG_LEVEL`, default to `"debug"` when `process.env.NODE_ENV === "development"`, otherwise `"info"`.
  - `timestamp`: Use pino's built-in ISO timestamp (`pino.stdTimeFunctions.isoTime`).
  - `formatters.level`: Format the level as a string label (e.g., `"info"`) instead of the default numeric value. Pino's `formatters: { level(label) { return { level: label } } }` achieves this.
- This root logger is used directly only in non-request contexts (e.g., startup). All request-scoped logging uses the helper below.

### `createRequestLogger(request: Request)` (factory function)

- Extract `request_id` from the `x-request-id` header on the incoming request (the middleware sets this — see section 3). If the header is missing (e.g., direct calls in tests), generate a `crypto.randomUUID()` as fallback.
- Extract `method` from `request.method`.
- Extract `path` from `new URL(request.url).pathname`.
- Return `logger.child({ request_id, method, path })` — a pino child logger with these fields bound to every log entry it produces.

### `logLevelForStatus(status: number)` (helper function)

- Returns the appropriate pino log level string based on HTTP status code:
  - `status >= 500` → `"error"`
  - `status >= 400` → `"warn"`
  - otherwise → `"info"`
- Used by route handlers to log the exit record at the correct level.

---

## 3. Middleware: Request ID Generation

**File:** `src/lib/supabase/middleware.ts`

**Current behavior (lines 5-53):** Creates a Supabase server client, refreshes the session, and redirects unauthenticated users away from `/dashboard`.

**Changes:**

1. At the top of `updateSession()`, before any other logic, generate a request ID:
   ```
   const requestId = crypto.randomUUID()
   ```

2. Clone the request headers and add the `x-request-id` header. Next.js middleware can modify request headers using the `NextResponse.next({ request: { headers } })` pattern. The function already uses `NextResponse.next({ request })` at line 6 — update this to include the new header in the forwarded request headers. The `x-request-id` header must be present on the request object that downstream route handlers receive.

3. Before returning `supabaseResponse`, set the `x-request-id` header on the response as well:
   ```
   supabaseResponse.headers.set('x-request-id', requestId)
   ```
   This makes the request ID visible in browser DevTools for client-side correlation.

4. **Important:** The function has multiple return paths. The early return at line 13 (when Supabase credentials aren't configured) must also generate and attach the request ID. Ensure every return path includes `x-request-id` on both request and response.

**Why request headers instead of AsyncLocalStorage:** Next.js App Router route handlers receive the `Request` object directly. Reading a header is simpler and more portable than introducing AsyncLocalStorage, which would require a custom server setup. If the codebase later needs request context in deeply nested utility functions, AsyncLocalStorage can be added as an enhancement — but for now, passing the logger (or request) explicitly is sufficient and easier to test.

---

## 4. GitHub API Instrumentation

**File:** `src/lib/github.ts`

The `validateGitHubRepo()` function makes 1-2 external HTTP calls (repo metadata + languages). These are the only external API calls in the codebase and need instrumentation for latency tracking, error visibility, and rate limit monitoring.

### Changes to `validateGitHubRepo()` signature

Add an optional `logger` parameter (type: pino's `Logger`, imported from `pino`). When provided, the function logs external API calls. When omitted (e.g., in unit tests), it behaves exactly as before — no logging, no side effects. This preserves testability.

```
export async function validateGitHubRepo(
  url: string,
  options?: { token?: string; logger?: Logger }
): Promise<GitHubValidationResult>
```

### Repo metadata fetch (lines 99-139)

**Before the fetch (after line 99):**
- Record `const startTime = Date.now()`.

**After a successful response (after line 132, before the private check):**
- Log at `info` level:
  ```
  {
    msg: "external_api_call",
    service: "github",
    endpoint: `/repos/${owner}/${repo}`,
    status: res.status,
    latency_ms: Date.now() - startTime,
    rate_limit_remaining: res.headers.get("x-ratelimit-remaining")
  }
  ```

**On non-OK responses (lines 105-130):**
- Before each return statement, log at `warn` level with the same fields plus `error` describing the issue (e.g., `"not_found"`, `"rate_limited"`, `"api_error"`).

**Catch block (line 133) — ANTI-PATTERN FIX:**

Currently:
```typescript
} catch {
  return {
    success: false,
    error: "Failed to reach GitHub...",
    code: "API_ERROR",
  };
}
```

This catches network errors and returns a user-friendly message but **logs nothing**. Change to:
- Capture the error: `catch (err)`
- Log at `error` level:
  ```
  {
    msg: "external_api_call",
    service: "github",
    endpoint: `/repos/${owner}/${repo}`,
    error: err instanceof Error ? err.message : String(err),
    latency_ms: Date.now() - startTime
  }
  ```
- Keep the existing return behavior unchanged.

### Language fetch (lines 151-161) — ANTI-PATTERN FIX

Currently:
```typescript
} catch {
  // Non-fatal — languages will be null
}
```

This completely swallows errors. Change to:
- Capture the error: `catch (err)`
- Log at `warn` level (not error — this is genuinely non-fatal):
  ```
  {
    msg: "external_api_call",
    service: "github",
    endpoint: `/repos/${owner}/${repo}/languages`,
    error: err instanceof Error ? err.message : String(err),
    latency_ms: Date.now() - langStartTime
  }
  ```
- Add a `const langStartTime = Date.now()` before the language fetch.
- Keep the existing behavior (languages remains `null`).

Also log successful language fetches at `info` level, same schema as above.

### Callers of `validateGitHubRepo()`

Two files call this function and must pass the logger:

1. **`src/app/api/github/validate/route.ts` (line 38):** Pass the request logger (see section 5).
2. **`src/app/api/demos/route.ts` (line 75):** Pass the request logger (see section 5).

---

## 5. API Route Instrumentation

Every API route handler follows the same instrumentation pattern. This section defines the pattern once, then lists the specific files and any per-route considerations.

### Universal Pattern

At the top of every route handler function:

1. **Create the request logger:**
   ```
   const log = createRequestLogger(request)
   ```

2. **Record start time:**
   ```
   const startTime = Date.now()
   ```

3. **Log request start** at `info` level after authentication resolves (so user_id is available):
   ```
   log.info({ msg: "request_start", user_id: user?.id ?? null })
   ```
   If the user role is also checked in that handler, include `user_role` in this log entry.

4. **Before every `return NextResponse.json(...)` statement**, log the exit:
   ```
   const status = <the status code being returned>
   const latency_ms = Date.now() - startTime
   log[logLevelForStatus(status)]({
     msg: "request_complete",
     user_id: user?.id ?? null,
     status,
     latency_ms,
     // Include on errors only:
     error: <error message if 4xx/5xx, null otherwise>,
     request_body: <parsed request body if 4xx/5xx, omit on success>
   })
   ```

**Note on `request_body` in logs:** Only include the parsed request body object in log entries for 4xx and 5xx responses. Do not log request bodies on success (2xx). This balances diagnostic power with log volume. If the request body was never successfully parsed (JSON parse failed), log `"unparseable"` as the value.

### Files to Instrument

#### `src/app/api/demos/route.ts` — POST handler

- 5 return paths (lines 13, 18, 26, 60, 89). Each needs an exit log.
- Pass logger to `validateGitHubRepo()` at line 75 (see section 4).
- Fire-and-forget fix at line 67 (see section 6).

#### `src/app/api/demos/[id]/route.ts` — PATCH and DELETE handlers

- PATCH has 11 return paths. Each needs an exit log.
- DELETE has 5 return paths. Each needs an exit log.
- Both handlers share the same `createRequestLogger(request)` call — but since they're separate exported functions, each creates its own logger instance.

#### `src/app/api/events/route.ts` — POST handler

- 5 return paths (lines 13, 18, 32, 74, 88). Each needs an exit log.
- Fire-and-forget fix at line 82 (see section 6).

#### `src/app/api/events/[id]/route.ts` — PATCH and DELETE handlers

- PATCH has 8 return paths. Each needs an exit log.
- DELETE has 5 return paths. Each needs an exit log.

#### `src/app/api/events/[id]/submissions/route.ts` — POST and GET handlers

- POST has 8 return paths. Each needs an exit log.
- GET has 3 return paths. Each needs an exit log.

#### `src/app/api/events/[id]/submissions/[submissionId]/route.ts` — PATCH and DELETE handlers

- PATCH has 7 return paths. Each needs an exit log.
- DELETE has 6 return paths. Each needs an exit log.
- Audit write error at line 106 (see section 7).

#### `src/app/api/events/[id]/submissions/[submissionId]/resubmit/route.ts` — POST handler

- 7 return paths. Each needs an exit log.

#### `src/app/api/github/validate/route.ts` — POST handler

- 4 return paths. Each needs an exit log.
- Pass logger to `validateGitHubRepo()` at line 38.

---

## 6. Fire-and-Forget Fixes

These are operations where a Supabase insert is awaited but the result is never checked. A failure here means the user sees success but the system is in an inconsistent state.

### `src/app/api/demos/route.ts` lines 67-72

**Current code:**
```typescript
await supabase.from("team_memberships").insert({
  demo_id: demo.id,
  user_id: user.id,
  role: "owner",
  status: "active",
});
```

**Change:** Capture the result and log on failure:
```
const { error: membershipError } = await supabase.from("team_memberships").insert(...)
```

If `membershipError` is truthy, log at `error` level:
```
{
  msg: "fire_and_forget_failed",
  operation: "team_membership_insert",
  demo_id: demo.id,
  user_id: user.id,
  error: membershipError.message
}
```

**Do not change the response behavior.** The demo was created successfully; the membership failure is a secondary concern. The log entry is what makes this visible and investigatable. The response to the client remains 201.

### `src/app/api/events/route.ts` lines 82-86

**Current code:**
```typescript
await supabase.from("event_memberships").insert({
  event_id: event.id,
  user_id: user.id,
  role: "organizer",
});
```

**Change:** Same pattern as above. Capture error, log at `error` level with `msg: "fire_and_forget_failed"`, `operation: "event_membership_insert"`, `event_id`, `user_id`, `error`. Do not change the response.

---

## 7. Console.error Replacements

All existing `console.error` calls must be replaced with structured log calls using the request-scoped logger. The existing `console.error` calls are:

### `src/app/api/demos/route.ts` line 59

**Current:** `console.error("Demo insert failed:", demoError);`

**Replace with:** This line is immediately before a 500 response return. The exit log from the universal pattern (section 5) will capture this error. Remove the `console.error` — it becomes redundant once the exit log includes `error: demoError?.message` and `request_body`.

### `src/app/api/events/route.ts` line 74

**Current:** `console.error("Event insert failed:", eventError);`

**Replace with:** Same as above — remove, covered by the exit log.

### `src/app/api/events/[id]/submissions/[submissionId]/route.ts` line 106

**Current:** `console.error("Failed to write submission review:", reviewError.message);`

**Replace with:** Log at `error` level using the request-scoped logger:
```
{
  msg: "fire_and_forget_failed",
  operation: "submission_review_insert",
  submission_id: submissionId,
  reviewer_id: user.id,
  action: body.status,
  error: reviewError.message
}
```

This is a fire-and-forget that already correctly continues execution. It just needs structured logging instead of `console.error`.

---

## 8. Environment Variables

Add one new optional environment variable:

| Variable | Default | Purpose |
|---|---|---|
| `LOG_LEVEL` | `"info"` in production, `"debug"` in development | Controls pino's minimum log level. Valid values: `"fatal"`, `"error"`, `"warn"`, `"info"`, `"debug"`, `"trace"`. |

No changes to `.env.local` or Vercel environment configuration are required for the defaults to work. Document this variable in the project README under the existing Environment Variables section.

---

## 9. Log Schema Reference

All log entries are JSON objects written to stdout. Pino adds `level`, `time`, and `pid` automatically. Our fields are layered on top.

### Common Fields (present on all request-scoped logs)

| Field | Type | Source | Description |
|---|---|---|---|
| `level` | string | pino | `"error"`, `"warn"`, `"info"`, `"debug"` |
| `time` | string | pino (ISO format) | ISO 8601 timestamp |
| `request_id` | string (UUID) | middleware | Unique per-request, set in middleware |
| `method` | string | request | HTTP method |
| `path` | string | request | URL pathname |
| `msg` | string | application | Event identifier (see below) |

### Event: `request_start`

| Field | Type | Description |
|---|---|---|
| `user_id` | string or null | Authenticated user ID, null if unauthenticated |
| `user_role` | string or null | User role if checked in that handler |

### Event: `request_complete`

| Field | Type | Description |
|---|---|---|
| `user_id` | string or null | Authenticated user ID |
| `status` | number | HTTP response status code |
| `latency_ms` | number | Time from handler entry to response |
| `error` | string or null | Error message on 4xx/5xx, null on 2xx |
| `request_body` | object or null | Parsed request body on 4xx/5xx only |

### Event: `external_api_call`

| Field | Type | Description |
|---|---|---|
| `service` | string | `"github"` (extensible to future services) |
| `endpoint` | string | API path called |
| `status` | number or null | HTTP status code, null on network error |
| `latency_ms` | number | Round-trip time |
| `rate_limit_remaining` | string or null | From `x-ratelimit-remaining` header |
| `error` | string or null | Error message on failure |

### Event: `fire_and_forget_failed`

| Field | Type | Description |
|---|---|---|
| `operation` | string | What failed (e.g., `"team_membership_insert"`) |
| `error` | string | Supabase error message |
| *(varies)* | | Context fields relevant to the operation (IDs, etc.) |

### Future Extension: LLM Calls

When AI features are added, the `external_api_call` schema extends with:

| Field | Type | Description |
|---|---|---|
| `service` | string | `"openai"`, `"anthropic"`, etc. |
| `model` | string | Model identifier |
| `input_tokens` | number | Tokens sent |
| `output_tokens` | number | Tokens received |
| `cost_usd` | number | Estimated cost |
| `prompt_sent` | string | Full prompt as sent to the model |
| `model_response` | string | Full model response |

The `request_id` and `user_id` tracing is already in place — just add these fields.

---

## 10. Testing Criteria

### Unit Tests

Add a new test file: `src/lib/__tests__/logger.test.ts`

1. **`createRequestLogger` returns a child logger with bound fields:** Call with a mock `Request` that has `x-request-id`, `method`, and `url`. Verify the returned logger's bindings include `request_id`, `method`, and `path`.

2. **`createRequestLogger` generates fallback request_id:** Call with a `Request` that has no `x-request-id` header. Verify a UUID is generated (check format).

3. **`logLevelForStatus` returns correct levels:** Parameterized test:
   - `200` → `"info"`
   - `201` → `"info"`
   - `400` → `"warn"`
   - `401` → `"warn"`
   - `403` → `"warn"`
   - `404` → `"warn"`
   - `409` → `"warn"`
   - `500` → `"error"`

### Integration Tests

Existing integration tests in `src/tests/integration/` should continue to pass without modification. The logging changes are additive — they don't alter request/response behavior. Run `npm run test:all` to verify.

### Manual Verification

After implementation:

1. Start the dev server (`npm run dev`).
2. Create a demo via the UI.
3. Verify stdout shows JSON log entries with:
   - `request_start` and `request_complete` for the `POST /api/demos` call
   - `external_api_call` for the GitHub validation
   - `request_id` consistent across all entries for that request
   - `latency_ms` populated
   - `user_id` populated
4. Trigger an error (e.g., submit an invalid repo URL) and verify:
   - Exit log level is `"warn"` (400 status)
   - `request_body` is included in the exit log
   - `error` field describes the issue

---

## Files Changed (Summary)

| File | Action | Lines Affected |
|---|---|---|
| `package.json` | Modify | Add `pino` to dependencies |
| `src/lib/logger.ts` | **Create** | ~40 lines |
| `src/lib/__tests__/logger.test.ts` | **Create** | ~50 lines |
| `src/lib/supabase/middleware.ts` | Modify | Lines 5-6 (request ID generation), all return paths (header attachment) |
| `src/lib/github.ts` | Modify | Lines 76-79 (signature), 99-139 (repo fetch logging), 151-161 (language fetch logging) |
| `src/app/api/demos/route.ts` | Modify | Add logger + timing at top, exit logs at all return paths, fix fire-and-forget at L67, remove console.error at L59 |
| `src/app/api/demos/[id]/route.ts` | Modify | Add logger + timing at top of PATCH and DELETE, exit logs at all return paths |
| `src/app/api/events/route.ts` | Modify | Add logger + timing at top, exit logs at all return paths, fix fire-and-forget at L82, remove console.error at L74 |
| `src/app/api/events/[id]/route.ts` | Modify | Add logger + timing at top of PATCH and DELETE, exit logs at all return paths |
| `src/app/api/events/[id]/submissions/route.ts` | Modify | Add logger + timing at top of POST and GET, exit logs at all return paths |
| `src/app/api/events/[id]/submissions/[submissionId]/route.ts` | Modify | Add logger + timing at top of PATCH and DELETE, exit logs at all return paths, replace console.error at L106 |
| `src/app/api/events/[id]/submissions/[submissionId]/resubmit/route.ts` | Modify | Add logger + timing at top, exit logs at all return paths |
| `src/app/api/github/validate/route.ts` | Modify | Add logger + timing at top, exit logs at all return paths, pass logger to validateGitHubRepo |
