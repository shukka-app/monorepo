# Plan 008: Regenerate lockfile so `npm ci` works

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. Do **not** update `plans/README.md`.
>
> **Drift check (run first)**: `git diff --stat 9afada0..HEAD -- package.json package-lock.json AGENTS.md README.md Dockerfile`
> On mismatch, STOP.

## Status

- **Priority**: P2
- **Effort**: S
- **Risk**: LOW
- **Depends on**: none
- **Category**: dx
- **Planned at**: commit `9afada0`, 2026-08-21

## Why this matters

`AGENTS.md` documents that `package-lock.json` is out of sync with
`package.json`, so `npm ci` fails. CI and `Dockerfile` both run `npm ci`.
README tells newcomers to use `npm ci`. If the lock is actually valid,
this plan is a no-op plus a doc fix. If it is stale, regenerate it.

## Current state

- `package.json` scripts and deps as of 1.0.2
- `AGENTS.md` Cursor Cloud notes: "`npm ci` fails: the committed
  `package-lock.json` is slightly out of sync… use `npm install`"
- `README.md` Develop / from-source: `npm ci`
- `Dockerfile` lines 6–7 and 18–20: `npm ci` / `npm ci --omit=dev`

Do **not** bump dependency ranges. Do **not** add packages.

## Commands you will need

| Purpose | Command | Expected on success |
|---------|---------|---------------------|
| Check lock | `npm ci` | exit 0 **or** a clear `EINTEGRITY` / missing-dep error |
| Regen | `npm install` | exit 0, lockfile updated if needed |
| Re-check | `npm ci` | exit 0 |
| Tests | `npm test` | pass |
| Lint | `npm run lint` | exit 0 |

`npm ci` deletes `node_modules` — that is allowed in an isolated
worktree. Do not run this against a worktree the user is actively
developing in if you were dispatched without isolation; if `npm ci`
would destroy a dirty `node_modules` the user needs, STOP.

## Scope

**In scope**:
- `package-lock.json`
- `AGENTS.md` — remove or rewrite the "npm ci fails" bullet after
  `npm ci` succeeds
- `README.md` — keep `npm ci` if it works; if you cannot make `npm ci`
  work, document `npm install` (last resort)

**Out of scope**:
- `package.json` version ranges
- `tests/e2e/package-lock.json`
- Nitro / TS / Zod upgrades
- Adding Prettier

## Git workflow

- Branch: `advisor/008-lockfile-npm-ci`
- Commit: `chore: sync package-lock.json so npm ci succeeds`
- Do NOT push.

## Steps

### Step 1: See whether `npm ci` already works

Run `npm ci`. 

- If exit 0: skip regen. Edit `AGENTS.md` to delete the stale warning
  (it is now wrong). README stays `npm ci`.
- If exit ≠ 0: go to step 2.

**Verify**: record the exit code in NOTES.

### Step 2: Regenerate lockfile (only if step 1 failed)

`npm install` with the existing `package.json` ranges. Do not pass
`--force` or `--legacy-peer-deps` unless install fails and the error is
a peer conflict — then STOP and report the peer error.

**Verify**: `npm ci` now exits 0.

### Step 3: Tests

**Verify**: `npm test` → pass. `npm run lint` → 0.

If `routeTree.gen.ts` is missing after a clean `npm ci`, `npm test` may
still pass (vitest does not need it). Do not commit generated route
tree.

## Test plan

No new tests. `npm ci` + `npm test` is the gate.

## Done criteria

- [ ] `npm ci` exits 0
- [ ] `AGENTS.md` does not claim `npm ci` fails
- [ ] `package.json` dependency ranges unchanged (`git diff package.json`
      empty except if you must not touch it at all)
- [ ] `npm test` exits 0
- [ ] No files outside scope

## STOP conditions

- `npm install` wants to change `package.json` (save it and revert
  `package.json`; only the lock should change).
- Peer dependency conflict that needs a major bump.
- You are not in an isolated worktree and `npm ci` would wipe the
  user's `node_modules`.

## Maintenance notes

Reviewer: lockfile-only diffs can hide unexpected version floats inside
semver ranges. Skim `git diff package-lock.json` for major jumps on
`@tanstack/*`, `drizzle-orm`, `better-sqlite3`, `nitro`.
