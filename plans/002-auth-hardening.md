# Plan 002: Harden login, cookies, 500s, and feed decoding

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. Do **not** update `plans/README.md`.
>
> **Drift check (run first)**: `git diff --stat 9afada0..HEAD -- src/lib/auth.ts src/lib/errors.ts src/routes/api/admin/login.ts src/routes/api/admin/setup.ts src/routes/api/admin/password.ts src/routes/api/admin/logout.ts src/routes/api/update src/lib/i18n/en.ts src/lib/i18n/zh.ts tests/auth.test.ts tests/i18n.test.ts docs/spec.md docs/adr/auth-model.md`
> On mismatch with "Current state" excerpts, STOP.

## Status

- **Priority**: P1
- **Effort**: M
- **Risk**: LOW
- **Depends on**: none
- **Category**: security
- **Planned at**: commit `9afada0`, 2026-08-21

## Why this matters

`docs/adr/auth-model.md` says login should use a fixed-window rate limit.
`POST /api/admin/login` currently calls `login()` with no throttle — the
single admin password can be guessed online. Session cookies are `HttpOnly;
SameSite=Lax` but never `Secure`, so a TLS-terminating deploy can leak the
cookie on HTTP. Unhandled exceptions return `error.message` to the client.
Malformed percent-encoding on the public feed splat or the session cookie
throws `URIError` → 500.

## Current state

`src/lib/auth.ts` (cookie + login):

```
export function sessionCookieHeader(token: string, maxAge = SESSION_TTL_SECONDS): string {
  return `${SESSION_COOKIE}=${token}; Path=/; HttpOnly; SameSite=Lax; Max-Age=${maxAge}`
}
```

`readSessionCookie` does `decodeURIComponent(rest.join('='))` with no try/catch
(around line 74). `login()` has no rate-limit hook.

Callers of `sessionCookieHeader(token)` with no request context:
- `src/routes/api/admin/login.ts:15`
- `src/routes/api/admin/setup.ts:15`
- `src/routes/api/admin/password.ts:16`

`src/lib/errors.ts` `jsonError` for non-`ShukkaError`:

```
const message = error instanceof Error ? error.message : 'Unexpected error'
return Response.json({ error: 'internal_error', message }, { status: 500 })
```

`ShukkaErrorCode` is a closed set: `unauthorized`, `forbidden`, `not_found`,
`conflict`, `invalid_request`, `storage_error`, `metadata_error`. Spec
`docs/spec.md:62` lists the same set. Adding a code requires spec +
`src/lib/i18n/en.ts` / `zh.ts` `errors` keys (en is source; zh `satisfies Dictionary`).

Feed splat: `src/routes/api/update/$appSlug/$channel/$.ts:13`
`decodeURIComponent(params._splat ?? '')` uncaught.

ADR quote (`docs/adr/auth-model.md`): "无速率限制之外的防爆破设计，登录接口做固定窗口限速即可。"

View roles / unauthenticated feed are **not** in scope. Do not add CSRF tokens.
Do not revoke all sessions on login (password change already does).

## Commands you will need

| Purpose | Command | Expected on success |
|---------|---------|---------------------|
| Tests | `npm test -- tests/auth.test.ts tests/i18n.test.ts tests/errors.test.ts` | all pass |
| Full suite | `npm test` | all pass |
| Lint | `npm run lint` | exit 0 |
| Typecheck | `npm run typecheck` | exit 0 |

## Scope

**In scope**:
- `src/lib/auth.ts`
- `src/lib/errors.ts`
- `src/lib/rate-limit.ts` (create)
- `src/routes/api/admin/login.ts`
- `src/routes/api/admin/setup.ts`
- `src/routes/api/admin/password.ts`
- `src/routes/api/update/$appSlug/$channel/$.ts`
- `src/lib/i18n/en.ts`, `src/lib/i18n/zh.ts` (only if a new error code is added)
- `docs/spec.md` (error-code list + login rate-limit sentence)
- `tests/auth.test.ts`
- `tests/errors.test.ts` (create)
- `tests/i18n.test.ts` only if dictionaries change

**Out of scope**:
- Docker `USER` / `HEALTHCHECK` (plan 007)
- CSRF, multi-admin, captcha
- Feed authorization (public by contract)
- Changing password strength beyond the existing 8-char check

## Git workflow

- Branch: `advisor/002-auth-hardening`
- Commit: `fix: rate-limit login and stop leaking 500s`
- Do NOT push.

## Steps

### Step 1: Generic `jsonError` must not leak `error.message`

In `src/lib/errors.ts`, for non-`ShukkaError` return
`{ error: 'internal_error', message: 'Unexpected error' }`.
Log the real error with `console.error` (no request body, no secrets).

Add `tests/errors.test.ts`: first import `./setup-db.ts` only if the module
graph needs the DB — `errors.ts` does not import the DB, so **do not**
import setup-db. Assert `jsonError(new Error('ENOENT: /etc/passwd'))`
body.message is exactly `Unexpected error` and status 500. Assert a
`ShukkaError('not_found', 'App missing')` still returns that message.

**Verify**: `npm test -- tests/errors.test.ts` → pass.

### Step 2: Safe `decodeURIComponent`

In `src/lib/auth.ts`, wrap cookie decode in try/catch; on `URIError` return
`null` (invalid session).

In the feed splat route, wrap splat decode; on failure throw
`ShukkaError('not_found', 'Not found')` so `handle()` returns 404.

**Verify**: add a case in `tests/auth.test.ts`: `Cookie: shukka_session=%E0%A4%A`
→ `readSessionCookie` returns `null`, `sessionIsValid` false. Add a feed
route test **or** a tiny unit if you extract `safeDecodeURIComponent` to
`src/lib/auth.ts` or `src/lib/http.ts` and reuse it. Prefer one helper.

### Step 3: Secure cookie flag

Change `sessionCookieHeader` to accept the `Request` (or an options object
`{ secure?: boolean }`):

- `secure === true` when `SHUKKA_SECURE_COOKIES` is `1`/`true`, **or** the
  request URL protocol is `https:`, **or** `X-Forwarded-Proto` is `https`.
- Append `; Secure` only then.
- `clearSessionCookieHeader` should take the same signal so logout clears
  the same cookie flags.

Update login, setup, password, and logout (`src/routes/api/admin/logout.ts`)
to pass `request`.

Default local `http://localhost:3000` must **not** set Secure (otherwise
the panel cannot log in in dev).

**Verify**: unit-test a helper `cookieShouldBeSecure(request)`:
- `http://localhost/api/admin/login` → false
- `https://updates.example.com/...` → true
- `http://...` + `X-Forwarded-Proto: https` → true
- env `SHUKKA_SECURE_COOKIES=1` → true (restore env after)

### Step 4: Login rate limit

Create `src/lib/rate-limit.ts`: in-process fixed window, key = client IP.

- Window: 15 minutes. Limit: 10 **failed** logins per IP.
- Successful login does not increment (or resets the counter — pick reset
  on success, document in a one-line comment).
- IP from `X-Forwarded-For` first hop if present, else `X-Real-IP`, else
  `'local'`. Do not invent a new trust model; this is single-admin
  self-host behind a reverse proxy.

Add `rate_limited` to `ShukkaErrorCode` with HTTP 429. Update:
- `src/lib/errors.ts` `statusByCode`
- `docs/spec.md` error-code list (line ~62)
- `src/lib/i18n/en.ts` and `zh.ts` `errors.rate_limited`

In `src/routes/api/admin/login.ts`, **before** `login()`:
if `isLimited(ip)` throw `ShukkaError('rate_limited', 'Too many login attempts. Try again later.')`.
On `login()` throwing `unauthorized`, `recordFailure(ip)` then rethrow.
Do not rate-limit setup (first-boot) or password change (already session).

Export `resetRateLimitForTests()` from `src/lib/rate-limit.ts` for tests
(clears the Map). Do not export anything else the panel needs.

**Verify**: `tests/auth.test.ts` (or `tests/login-route.test.ts` modeled on
`tests/app-api.test.ts`):
1. 10 failed POSTs with `password: "wrong"` → 11th is 429, `error === 'rate_limited'`.
2. After `resetRateLimitForTests()`, a correct password still 200.
3. Correct password after fewer than 10 failures still 200.

Use `routeHandler` on `~/routes/api/admin/login.ts`.

### Step 5: Full suite

**Verify**: `npm test` → pass. `npm run lint` → exit 0. `npm run typecheck` → exit 0.

## Test plan

- `tests/errors.test.ts` — 500 redaction; ShukkaError passthrough.
- `tests/auth.test.ts` — bad cookie decode; Secure helper; rate-limit window.
- `tests/i18n.test.ts` already asserts zh/en key parity — must stay green
  if you add `rate_limited`.

## Done criteria

- [ ] `npm test` exits 0
- [ ] `jsonError(new Error('secret'))` message is `Unexpected error`
- [ ] Login 11th failure is 429 `rate_limited`
- [ ] `sessionCookieHeader` includes `Secure` only for HTTPS / env / forwarded proto
- [ ] Bad percent-encoding on cookie or splat does not 500
- [ ] `docs/spec.md` lists `rate_limited`
- [ ] No files outside scope (`git status`)

## STOP conditions

- Adding `rate_limited` would require changing the public App API error
  envelope beyond the closed code set (it should not — same `{ error, message }`).
- Login route is no longer a TanStack Start `createFileRoute` handler.
- You believe you must add Redis or a dependency — STOP; in-process Map is required.

## Maintenance notes

- In-process limit does not share across multiple Node processes. Document
  that in plan 007's operator runbook, not here.
- Reviewer: confirm failed-login counter cannot be used as a user-enumeration
  oracle beyond the existing `unauthorized` message (same message as today).
