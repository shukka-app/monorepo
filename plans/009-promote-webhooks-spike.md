# Plan 009: Spike — post-promote webhooks (design only)

> **Executor instructions**: This is a **design/spike plan**. You will
> write a PRD (and an ADR only if a material technical choice is forced).
> You will **not** implement webhook delivery, HTTP clients, or schema
> columns. If you start editing `src/`, you have gone out of scope.
> Do **not** update `plans/README.md`.
>
> **Drift check (run first)**: `git diff --stat 9afada0..HEAD -- docs/prd/draft-releases.md docs/spec.md src/server/channels.ts`
> On mismatch, STOP.

## Status

- **Priority**: P3
- **Effort**: S
- **Risk**: LOW (docs only)
- **Depends on**: none
- **Category**: direction
- **Planned at**: commit `9afada0`, 2026-08-21

## Why this matters

Draft → promote is the release model (`docs/prd/draft-releases.md`).
`setCurrentVersion` in `src/server/channels.ts` atomically writes
`releasedAt` and switches the pointer. Nothing notifies CI, Slack, or
an ops channel when the feed moves. Implementing delivery without a
PRD would skip the repo's `$feature-dev` 质问 (see `AGENTS.md`).

This plan records the product questions and a proposed contract so a
later implementation plan can be written. It does **not** ship webhooks.

## Current state

- Promote/rollback: `PATCH /api/v1/apps/{slug}/channels/{channel}` with
  `{ currentVersion }` (`docs/spec.md` App API).
- No webhook/notify code under `src/`.
- View role `content` watches releases but cannot promote
  (`docs/prd/view-roles.md`).
- User-supplied URLs are an SSRF surface (same class as S3 endpoints,
  but webhooks are outbound POST to arbitrary URLs).
- Feature docs live in `docs/prd/*.md`, decisions in `docs/adr/*.md`,
  stable contracts in `docs/spec.md`. `AGENTS.md` forbids implementing
  feature code during 质问.

## Commands you will need

| Purpose | Command | Expected on success |
|---------|---------|---------------------|
| None required | — | — |

Do not run `npm install` or formatters.

## Scope

**In scope**:
- `docs/prd/promote-webhooks.md` (create)
- Optionally `docs/adr/promote-webhook-delivery.md` if you must record
  HMAC + retry vs none

**Out of scope**:
- Any file under `src/`, `tests/`, `drizzle/`, `scripts/`
- Updating `docs/spec.md` (no stable contract yet)
- Slack-specific adapters
- Implementing the feature

## Git workflow

- Branch: `advisor/009-promote-webhooks-spike`
- Commit: `docs: spike PRD for post-promote webhooks`
- Do NOT push.

## Steps

### Step 1: Write the PRD

Create `docs/prd/promote-webhooks.md` with:

- **Problem**: after draft upload, go-live is silent.
- **Users**: admin/developer who promote; content who watches; CI that
  uploaded the draft.
- **Goals**: optional per-app HTTPS URL; fire when `currentVersion`
  actually changes (promote **or** rollback); payload
  `{ app, channel, version, releasedAt, previousVersion }`; HMAC header;
  logged failures; no feed-path latency (async after the transaction).
- **Non-goals**: notify on every finalize/draft; guaranteed exactly-once;
  inbound webhooks; replacing the panel.
- **Open questions** (leave unanswered if you cannot decide from the
  repo — do not invent product answers):
  1. One URL per app vs per channel?
  2. Retry policy (none / 3x / dead letter)?
  3. Secret storage (reuse S3 encryption key vs separate)?
  4. SSRF allowlist (https-only, deny link-local)?
- **Acceptance criteria**: unchecked, testable, no implementation TODOs.

**Verify**: file exists; no `src/` in `git status`.

### Step 2: Stop

Do not implement. NOTES should say this is ready for `$feature-dev`
质问 before any build plan.

**Verify**: `git diff --stat` lists only `docs/prd/promote-webhooks.md`
and optionally one ADR.

## Test plan

None.

## Done criteria

- [ ] `docs/prd/promote-webhooks.md` exists with problem, users, goals,
      non-goals, open questions, unchecked AC
- [ ] No `src/` changes
- [ ] No `docs/spec.md` change

## STOP conditions

- You believe the feature is already specified elsewhere — stop and
  cite the file instead of duplicating.
- You start needing schema or fetch() — stop; that is implementation.

## Maintenance notes

A future build plan must go through `$feature-dev` and update
`docs/spec.md` in the same change as the code. SSRF controls are
mandatory if URLs are user-supplied.
