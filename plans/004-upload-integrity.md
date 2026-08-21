# Plan 004: Make upload init/finalize and S3 errors honest

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. Do **not** update `plans/README.md`.
>
> **Drift check (run first)**: `git diff --stat 9afada0..HEAD -- src/server/releases.ts src/lib/storage.ts src/lib/crypto.ts src/db/schema.ts src/lib/errors.ts tests/release-flow.test.ts tests/crypto.test.ts drizzle`
> On mismatch with "Current state" excerpts, STOP.

## Status

- **Priority**: P1
- **Effort**: M
- **Risk**: MED
- **Depends on**: plans/001-characterization-tests.md
- **Category**: bug
- **Planned at**: commit `9afada0`, 2026-08-21

## Why this matters

`initUpload` only rejects when a **version row** already exists. Two CI jobs
can init the same `(channel, version)`, PUT the same S3 keys, and race
finalize. The loser hits `UNIQUE` on `versions_channel_version_unique` and
becomes `internal_error` 500. `headObject` treats every AWS failure as
"not uploaded". `deleteObjects` swallows all errors so DB rows vanish while
objects remain. `decryptSecret` throws a bare `Error` → 500 on the feed
when the encryption key does not match the ciphertext.

## Current state

`src/server/releases.ts` `initUpload` (lines 62–69) checks `versions` only.
Pending insert is lines 81–90. No lookup of `pendingUploads` for the same
channel+version.

`finalizeUpload` insert (165–176) has no UNIQUE catch. `jsonError` maps
unknown errors to 500.

`src/db/schema.ts` `pendingUploads` (153–166): no unique on
`(channelId, version)`.

`src/lib/storage.ts`:

```
export async function headObject(...): Promise<{ size: number } | null> {
  try {
    const result = await client(s3).send(new HeadObjectCommand(...))
    return { size: result.ContentLength ?? 0 }
  } catch {
    return null
  }
}

export async function deleteObjects(...): Promise<void> {
  await Promise.all(
    keys.map((Key) => s3Client.send(new DeleteObjectCommand(...)).catch(() => undefined)),
  )
}
```

`src/lib/crypto.ts` `decryptSecret` line 40:
`if (!iv || !tag || !ciphertext) throw new Error('Malformed encrypted secret')`.
GCM auth failure also throws a raw Node error.

`deleteVersion` (`releases.ts:223`), `deleteChannel` (`channels.ts:53`),
`deleteApp` (`apps.ts:140`) all `await deleteObjects(...)` then delete DB
rows. After this plan, a storage failure must **prevent** the DB delete
(throw `storage_error`). That is the intended behavior.

Tests: `tests/release-flow.test.ts` mocks `headObject` / `deleteObjects`.
Update the mock only if the real function's contract changes in a way the
mock must simulate (e.g. `headObject` throwing). Keep the mock returning
`null` for missing keys.

Migrations: next file is `drizzle/0005_*.sql`. Generate with
`npm run db:generate` after schema change, or write a small SQL file and
snapshot if drizzle-kit requires a TTY — prefer `npm run db:generate`.
If generate wants interactive names, write:

```
CREATE UNIQUE INDEX `pending_uploads_channel_version_unique`
  ON `pending_uploads` (`channel_id`,`version`);
```

Expired rows are deleted on each init (`releases.ts:80`) **before** insert,
so a unique index on all pending rows is OK: expired rows are gone first.
If a unique insert races, map to `conflict`.

## Commands you will need

| Purpose | Command | Expected on success |
|---------|---------|---------------------|
| Tests | `npm test -- tests/release-flow.test.ts tests/crypto.test.ts` | all pass |
| Generate migration | `npm run db:generate` | new `drizzle/0005_*.sql` |
| Full suite | `npm test` | all pass |
| Lint | `npm run lint` | exit 0 |

## Scope

**In scope**:
- `src/server/releases.ts`
- `src/lib/storage.ts`
- `src/lib/crypto.ts`
- `src/db/schema.ts`
- `drizzle/0005_*.sql` + `drizzle/meta/*` if generated
- `tests/release-flow.test.ts`
- `tests/crypto.test.ts`
- `src/lib/errors.ts` only if you add a helper `isUniqueConstraint(error)`

**Out of scope**:
- Feed filename collision ORDER BY (rollback-by-filename is by design)
- Orphan cleanup cron for expired pending S3 objects
- Changing presigned URL TTLs
- Dashboard / panel

## Git workflow

- Branch: `advisor/004-upload-integrity`
- Commit: `fix: reject concurrent pending uploads and map S3 errors`
- Do NOT push.

## Steps

### Step 1: Reject a live pending for the same version

In `initUpload`, after the existing version clash check and **after**
deleting expired pending rows, query `pendingUploads` for
`channelId + version`. If a row exists, throw
`ShukkaError('conflict', `Version ${version} already has a pending upload`)`.

Also add `uniqueIndex('pending_uploads_channel_version_unique').on(t.channelId, t.version)`
on `pendingUploads` in `schema.ts`. Generate migration 0005.

If insert still hits UNIQUE (race), catch and throw the same `conflict`.

**Verify**: in `tests/release-flow.test.ts`, init twice for `1.0.1` without
finalize → second throws `ShukkaError` with code `conflict`. After finalize
(or after manually expiring the pending row), a new init for that version
still conflicts on the **version** row (existing test). After expire+delete
of pending without a version row, re-init succeeds — add that case by
setting `expiresAt` in the past and calling init again (init's expiry
cleanup should remove it).

### Step 2: Finalize UNIQUE → conflict

Wrap the version insert (or the whole transaction) so a SQLite unique
failure becomes `ShukkaError('conflict', 'Version already exists')`.
Detect via `error.code` / message containing `UNIQUE` — better-sqlite3
uses `code: 'SQLITE_CONSTRAINT_UNIQUE'`. If the shape differs, read the
thrown error in a failing test and match that. Do not swallow other errors.

**Verify**: finalize the same `uploadId` twice (or two pendings if you can
bypass step 1 in the test by inserting a version row mid-flight). Second
finalize: first pending already deleted → `not_found` is OK; the UNIQUE
path can be tested by inserting a version row with the same channel+version
after init and before finalize.

### Step 3: Honest `headObject`

Return `null` only for not-found. Treat as not-found when:
- `error.name === 'NotFound'`
- `error.$metadata?.httpStatusCode === 404`
- `error.name === 'NoSuchKey'`

Any other error: throw `ShukkaError('storage_error', 'Cannot reach storage', /* do not put raw SDK message in details if plan 002 redacted details — omit details */)`.

Update the storage mock in `release-flow.test.ts` only if tests break.

**Verify**: unit-test `headObject` with a mocked `S3Client` **or** extract
`isS3NotFound(error)` to a small exported function and test that. Do not
add `@aws-sdk` test deps if you can test the predicate.

### Step 4: `deleteObjects` must fail loudly

Remove `.catch(() => undefined)`. Use `Promise.all` and let failures throw,
wrapped as `ShukkaError('storage_error', 'Failed to delete objects from storage')`.

Empty `keys` array: no-op (already fine).

`verifyWritable`'s delete of the probe object may still ignore delete
failure after a successful put (probe leftover is acceptable). **Do not**
change that `.catch` on the probe delete unless you have to for types.

**Verify**: `tests/release-flow.test.ts` delete-version cases still pass
(mock `deleteObjects` still resolves). Add a test that if the mock
`deleteObjects` rejects, `deleteVersion` throws and the version row
**remains**.

### Step 5: Typed `decryptSecret`

Import `ShukkaError` in `src/lib/crypto.ts`. On malformed payload or
decipher failure, throw
`ShukkaError('storage_error', 'Stored S3 secret cannot be decrypted')`.
Do not include the ciphertext in the message.

**Verify**: extend `tests/crypto.test.ts` — garbage string and a secret
encrypted then decoded with a broken payload both throw `ShukkaError` with
code `storage_error`.

### Step 6: Full suite

**Verify**: `npm test` → pass. `npm run lint` → 0. `npm run typecheck` → 0.

## Test plan

- Double init same version while pending → 409/conflict
- Re-init after pending expiry → allowed if no version row
- Finalize when version row already exists → conflict not 500
- `deleteVersion` when `deleteObjects` rejects → version still in DB
- `decryptSecret` malformed → `storage_error`
- Pattern: `tests/release-flow.test.ts`, `tests/crypto.test.ts`

## Done criteria

- [ ] Second `initUpload` for an in-flight pending is `conflict`
- [ ] Unique version insert surfaces as `conflict`
- [ ] `headObject` does not map credential/network errors to "not uploaded"
- [ ] `deleteObjects` no longer swallows errors (grep
      `deleteObjects` / `.catch(() => undefined)` in `storage.ts` — only
      the probe delete may still swallow)
- [ ] `decryptSecret` throws `ShukkaError`
- [ ] Migration 0005 exists
- [ ] `npm test` exits 0
- [ ] No files outside scope

## STOP conditions

- `npm run db:generate` would rewrite unrelated migrations.
- better-sqlite3 unique error shape cannot be detected without catching
  **all** errors — STOP and report the error object rather than swallowing.
- Existing release-flow tests fail in a way that requires changing feed
  rollback-by-filename behavior.

## Maintenance notes

Reviewer: deleting a version now fails closed if S3 is down. Operators may
see `storage_error` instead of a silent leak — that is intended.
Expired pending S3 orphans are still not cleaned (deferred).
