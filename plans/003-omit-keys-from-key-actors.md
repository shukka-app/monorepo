# Plan 003: Omit API keys from key-authenticated app detail

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. Do **not** update `plans/README.md`.
>
> **Drift check (run first)**: `git diff --stat 9afada0..HEAD -- src/routes/api/v1/apps.$appSlug.ts src/server/dashboard.ts src/lib/auth.ts tests/app-api.test.ts docs/spec.md`
> On mismatch with "Current state" excerpts, STOP.

## Status

- **Priority**: P1
- **Effort**: S
- **Risk**: LOW
- **Depends on**: plans/001-characterization-tests.md
- **Category**: security
- **Planned at**: commit `9afada0`, 2026-08-21

## Why this matters

Spec (`docs/spec.md`): API keys may read app detail and mutate app-scoped
resources, but **must not manage API keys**. Dedicated key routes already
use `requireSessionApp`. `GET /api/v1/apps/{slug}` uses `requireAppActor`
(Bearer allowed) and always returns `keys` (id, name, hint, lastUsedAt,
revokedAt). A leaked CI key can enumerate every sibling key. Plaintext
key values are not returned; metadata still is.

## Current state

`src/routes/api/v1/apps.$appSlug.ts`:

```
GET: handle(async ({ request, params }) => {
  const slug = textParam(params, 'appSlug')
  requireAppActor(request, slug)
  return Response.json(appDetailBySlug(slug, new URL(request.url).origin))
}),
```

`requireAppActor` (`src/lib/auth.ts:136-141`) returns
`{ app, via: 'session' | 'key' }` — the route currently **discards** `via`.

`src/server/dashboard.ts` `appDetail` (lines 87–96) always builds:

```
const keys = listApiKeys(app.id).map((key) => ({
  id, name, hint, createdAt, lastUsedAt, revokedAt
}))
return { app: publicApp(app), channels, keys }
```

Panel session loaders (`src/routes/_panel/apps.$appSlug.tsx` and
`src/features/apps/requests/apps.ts`) need `keys` for the API keys tab.
Session-authenticated GET must keep the current shape.

`tests/app-api.test.ts` (after plan 001) already hits GET with a Bearer key
and checks `app.slug`. Extend that assertion: response must not have a
`keys` array (or `keys` must be `[]` / omitted). Prefer **omitting** the
property so key actors cannot tell key count.

## Commands you will need

| Purpose | Command | Expected on success |
|---------|---------|---------------------|
| Tests | `npm test -- tests/app-api.test.ts` | all pass |
| Full suite | `npm test` | all pass |
| Lint | `npm run lint` | exit 0 |

## Scope

**In scope**:
- `src/routes/api/v1/apps.$appSlug.ts`
- `src/server/dashboard.ts` — only if you add an options flag
  `appDetail(id, origin, { includeKeys?: boolean })` defaulting to `true`
- `tests/app-api.test.ts`
- `docs/spec.md` — one sentence under App API: GET detail includes `keys`
  only for session actors

**Out of scope**:
- Changing `requireSessionApp` key routes
- Hiding `s3AccessKeyId` from key actors (keys may PATCH app settings)
- Numeric id removal from DTOs
- Panel UI

## Git workflow

- Branch: `advisor/003-omit-keys-from-key-actors`
- Commit: `fix: hide API key metadata from key-authenticated app detail`
- Do NOT push.

## Steps

### Step 1: Thread `via` through GET

In `apps.$appSlug.ts` GET:

```
const { via } = requireAppActor(request, slug)
return Response.json(appDetailBySlug(slug, origin, { includeKeys: via === 'session' }))
```

Add the optional third argument to `appDetailBySlug` / `appDetail`. When
`includeKeys` is false, return `{ app, channels }` with **no** `keys`
property. When true (default), keep today's `{ app, channels, keys }`.

Do not change `publicApp()`.

**Verify**: `npm test -- tests/app-api.test.ts` — existing tests still pass
(session POST key still 200; key GET still 200).

### Step 2: Assert the new contract

In the Bearer GET case: `expect(body).not.toHaveProperty('keys')`.

Add a session GET case: `expect(Array.isArray(body.keys)).toBe(true)` and
the created key's `hint` is present.

**Verify**: `npm test -- tests/app-api.test.ts` → pass.

### Step 3: Spec sentence

In `docs/spec.md` App API bullet about key capabilities, add that GET
`/api/v1/apps/{slug}` omits `keys` when authenticated with an API key.

**Verify**: `npm test` → pass. `npm run lint` → exit 0.

## Test plan

- Key GET: 200, has `app`, no `keys`.
- Session GET: 200, `keys` array includes the issued hint.
- Pattern: existing `tests/app-api.test.ts`.

## Done criteria

- [ ] Bearer GET app detail has no `keys` property
- [ ] Session GET app detail still includes `keys`
- [ ] `npm test` exits 0
- [ ] Spec updated
- [ ] No files outside scope

## STOP conditions

- Panel fetches keys only via GET app detail and a TypeScript error appears
  in `_panel/apps.$appSlug.tsx` because `keys` became optional — if so, keep
  session responses unchanged (default `includeKeys: true`) and only omit
  on the API route for `via === 'key'`. Do not make the panel type optional
  unless a compile error forces a narrow type guard.
- `requireAppActor` no longer returns `via`.

## Maintenance notes

Reviewer: confirm OpenAPI (`src/server/openapi.ts`) does not document `keys`
as required for this GET. If it does, leave OpenAPI to plan 007 unless a
one-line description is needed here.
