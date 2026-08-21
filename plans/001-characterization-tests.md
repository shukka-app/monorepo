# Plan 001: Add upload/authz/uploader characterization tests

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. Do **not** update `plans/README.md`.
>
> **Drift check (run first)**: `git diff --stat 9afada0..HEAD -- tests/app-api.test.ts tests/release-flow.test.ts tests/auth.test.ts tests/setup-db.ts src/routes/api/v1/upload.init.ts src/routes/api/v1/upload.finalize.ts src/routes/api/v1/apps.$appSlug.ts src/routes/api/v1/apps.$appSlug.keys.ts scripts/shukka-upload.mjs`
> If any in-scope file changed since this plan was written, compare the
> "Current state" excerpts against the live code before proceeding; on a
> mismatch, treat it as a STOP condition.

## Status

- **Priority**: P1
- **Effort**: M
- **Risk**: LOW
- **Depends on**: none
- **Category**: tests
- **Planned at**: commit `9afada0`, 2026-08-21

## Why this matters

The upload HTTP routes, the API-key vs session capability matrix, and
`scripts/shukka-upload.mjs` are the CI publish path. Domain tests in
`tests/release-flow.test.ts` call `initUpload`/`finalizeUpload` directly, so a
broken Zod schema or `authenticateApiKey` wiring would still be green. Later
plans (003, 004, 006) change those surfaces; characterization tests must land
first so regressions are caught.

This plan only **adds tests**. It does not change production behavior.

## Current state

- `tests/app-api.test.ts` — route-level auth matrix. Imports
  `~/routes/api/v1/apps.$appSlug.ts` and `keys.ts`. Helper `routeHandler` reads
  `Route.options.server.handlers[method]`. The first case is titled
  "lets a bound API key read and patch the app" but only asserts GET 200,
  DELETE 403, POST keys 403 — no PATCH, no GET/DELETE keys.
- `tests/release-flow.test.ts` — mocks `~/lib/storage.ts` with an in-memory
  `Map`; calls `initUpload`/`finalizeUpload` directly. Pattern for publish
  helper at lines 51–63.
- `tests/auth.test.ts` — function-level admin/session/API key tests. No HTTP
  login/setup routes. No expired-session case.
- `tests/setup-db.ts` — sets `SHUKKA_DATA_DIR` / `SHUKKA_DB_PATH` /
  `SHUKKA_KEY_PATH` to a temp dir **before** any app import. Every new test
  file must `import './setup-db.ts'` as the **first** import.
- `src/routes/api/v1/upload.init.ts` — Zod body `{ app?, channel, version,
  createChannel?, files[] }`; `authenticateApiKey(request, parsed.data.app)`
  then `initUpload`.
- `src/routes/api/v1/upload.finalize.ts` — Zod `{ uploadId, app?, release? }`.
- `scripts/shukka-upload.mjs` — `versionFromMetadata` only scans `.ya?ml`
  (lines 49–54). `collectFiles` skips dotfiles. No exports; the file is a
  script. For unit tests, extract the pure helpers into the same file as
  named exports **without changing runtime behavior**, or spawn the script
  with mocked env. Prefer exporting: keep `main()` behind
  `if (import.meta.url === pathToFileURL(process.argv[1]).href)` (already
  the Node action entry — check the bottom of the file before changing).

Convention: tests use vitest, `beforeEach` wipes tables, mock storage via
`vi.mock('~/lib/storage.ts', async (importOriginal) => ...)`. Match
`tests/app-api.test.ts` for route tests and `tests/release-flow.test.ts` for
storage mocks.

Error envelope is `{ error, message }` (`src/lib/errors.ts`). Auth errors:
missing/invalid key → `unauthorized` 401; wrong app → `forbidden` 403.

## Commands you will need

| Purpose | Command | Expected on success |
|---------|---------|---------------------|
| Tests (this plan) | `npm test -- tests/app-api.test.ts tests/upload-routes.test.ts tests/auth.test.ts tests/shukka-upload.test.ts` | all pass |
| Full unit suite | `npm test` | all pass (115 existing + new) |
| Lint | `npm run lint` | exit 0 |
| Typecheck | `npm run typecheck` | exit 0 (needs `src/routeTree.gen.ts`; if missing run `npm run build` once) |

## Scope

**In scope**:
- `tests/app-api.test.ts` (extend)
- `tests/upload-routes.test.ts` (create)
- `tests/auth.test.ts` (extend — session expiry only)
- `tests/shukka-upload.test.ts` (create)
- `scripts/shukka-upload.mjs` — **only** if needed to export pure helpers
  (`collectFiles`, `versionFromMetadata`, `readInput`). Do not change
  CLI flags, env names, or upload behavior.

**Out of scope**:
- Any production auth, upload, or dashboard logic.
- E2E (`tests/e2e/`).
- OpenAPI, README, i18n.
- Fixing the Tauri version-detection gap (plan 005).

## Git workflow

- Branch: `advisor/001-characterization-tests`
- Commits: conventional, e.g. `test: cover upload routes and API-key authz matrix`
- Do NOT push or open a PR.

## Steps

### Step 1: Extend the authz matrix in `tests/app-api.test.ts`

Add cases modeled on the existing `routeHandler` + `makeApp` helpers:

1. Bound API key **PATCH** app (name only / valid `appInputSchema` fields)
   returns 200. Body must satisfy `appInputSchema` in
   `src/routes/api/admin/apps.ts` (name, slug, s3 fields). Read that schema
   before writing the body.
2. Bound API key **GET** `/keys` and **DELETE** `/keys/:keyId` return 403.
3. Session cookie **GET** `/keys` returns 200.
4. Unauthenticated GET app returns 401.

Keep the existing two tests. Rename the overclaiming title to match what
it actually asserted plus the new PATCH case, e.g.
`lets a bound API key read and patch the app, but not delete it or manage keys`.

**Verify**: `npm test -- tests/app-api.test.ts` → all pass.

### Step 2: Add `tests/upload-routes.test.ts`

First import `./setup-db.ts`. Mock `~/lib/storage.ts` like
`tests/release-flow.test.ts` (in-memory objects + `presignPut` /
`headObject` / `getObjectText`). Import upload route modules **after**
the mock.

Cases:

1. Valid Bearer key + well-formed init body → 200, JSON has `uploadId` and
   `files`.
2. Missing `Authorization` → 401, `error === 'unauthorized'`.
3. Session cookie only (no Bearer) → 401.
4. Malformed JSON / missing `files` → 400, `error === 'invalid_request'`.
5. Key bound to app A, body `app: "b"` → 403.

Do not assert S3 side effects beyond the mock.

**Verify**: `npm test -- tests/upload-routes.test.ts` → all pass.

### Step 3: Session expiry in `tests/auth.test.ts`

After `createSession`, update the row's `expiresAt` to the past (or insert
a row with `expiresAt` in the past) and assert `sessionIsValid(token)` is
false. Then call `createSession()` and assert the expired row is gone
(`db.select().from(sessions)` count / `expiresAt` filter).

Do not change `SESSION_TTL_SECONDS`.

**Verify**: `npm test -- tests/auth.test.ts` → all pass.

### Step 4: Uploader unit tests

Read `scripts/shukka-upload.mjs` end-to-end. If `collectFiles` /
`versionFromMetadata` / `readInput` are not exported, export them as named
exports and keep the existing `main` entry so `action.yml` still runs
`main: scripts/shukka-upload.mjs`.

Create `tests/shukka-upload.test.ts` (this file does **not** need
`setup-db.ts` unless it imports app modules — prefer not to).

Cases:

1. `collectFiles` on a temp dir includes `latest.yml` + `App.exe`, skips
   `.DS_Store` and `.hidden`.
2. `versionFromMetadata` reads `version: 1.2.3` from a yml.
3. `versionFromMetadata` on a dir with only `latest.json` currently
   **throws** / `fail`s with the existing Electron-only message. Assert
   today's behavior (characterization). Plan 005 will flip this.
4. `readInput` prefers `INPUT_*` over `SHUKKA_*` if that is what the
   script does today — read the function and assert the real precedence.

**Verify**: `npm test -- tests/shukka-upload.test.ts` → all pass.

### Step 5: Full suite + lint

**Verify**: `npm test` → all pass. `npm run lint` → exit 0.

## Test plan

Covered by the steps. Structural patterns: `tests/app-api.test.ts` (routes),
`tests/release-flow.test.ts` (storage mock), `tests/auth.test.ts` (sessions).

## Done criteria

- [ ] `npm test` exits 0
- [ ] `tests/upload-routes.test.ts` exists and covers the five cases in step 2
- [ ] `tests/app-api.test.ts` asserts PATCH-with-key 200 and key GET/DELETE keys 403
- [ ] `tests/auth.test.ts` covers expired sessions
- [ ] `tests/shukka-upload.test.ts` exists
- [ ] No production behavior change except optional named exports from the uploader
- [ ] `git status` shows only in-scope files

## STOP conditions

- `Route.options.server.handlers` shape does not match `tests/app-api.test.ts`
  (TanStack Start route API changed).
- `appInputSchema` cannot be satisfied without a live S3 probe and
  `verifyWritable` is not already mocked in the new file.
- Exporting helpers from `scripts/shukka-upload.mjs` would break
  `action.yml` (`using: node24` / `main`).
- A step's verification fails twice after a reasonable fix.

## Maintenance notes

- Plan 003 will add: GET app with a key must **not** include `keys`.
- Plan 004 will add: second `initUpload` for the same version while pending
  → `conflict`.
- Plan 005 will change the `latest.json` characterization in step 4.3 from
  throw → parse version.
