# Plan 006: Batch dashboard reads and cache feed metadata

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. Do **not** update `plans/README.md`.
>
> **Drift check (run first)**: `git diff --stat 9afada0..HEAD -- src/server/dashboard.ts src/server/feed.ts src/server/updaters/tauri.ts src/server/channels.ts src/server/releases.ts tests`
> On mismatch, STOP.

## Status

- **Priority**: P2
- **Effort**: M
- **Risk**: MED
- **Depends on**: plans/001-characterization-tests.md
  (and land **003 before this** if both change `dashboard.ts`)
- **Category**: perf
- **Planned at**: commit `9afada0`, 2026-08-21

## Why this matters

`appSummaries` and `appDetail` issue one query per channel and per version
(artifacts). The panel sidebar and app page pay this on every load. The
Electron feed `getObjectText`s S3 on every client update check; Tauri
`generateFeedDocument` can read `latest.json` plus every `.sig`. Metadata
is immutable per version id.

Do **not** make hit recording async (ADR hit-trends: same transaction as
the counter). Do **not** paginate the App API in this plan (contract
change). Batching + cache only.

## Current state

`src/server/dashboard.ts` `appSummaries` (32–42):
`.map` → `listChannels(app.id)` → `listVersions(channel.id)`.

`appDetail` (69–77): `listVersions` then `listArtifacts(version.id)` per
version.

Helpers: `listChannels` (`src/server/channels.ts:16-18`),
`listVersions` (`:61-67`), `listArtifacts` (`src/server/releases.ts:205-207`).

If plan 003 added `includeKeys`, keep that option.

`src/server/feed.ts` 54–59: Electron metadata path calls `getObjectText`
then `recordHit`. Tauri generated document is 37–51.

`src/server/updaters/tauri.ts` `generateFeedDocument` uses `getText` for
json/sigs.

JSON response **shape** of `appDetail` / `appSummaries` must stay the same
(same fields, same ordering: versions by `createdAt DESC, id DESC`;
channels by `createdAt, id`; artifacts by filename).

## Commands you will need

| Purpose | Command | Expected on success |
|---------|---------|---------------------|
| Existing tests | `npm test -- tests/app-api.test.ts tests/release-flow.test.ts tests/tauri-updater.test.ts tests/hits.test.ts` | pass |
| Full suite | `npm test` | pass |
| Lint | `npm run lint` | exit 0 |

## Scope

**In scope**:
- `src/server/dashboard.ts`
- `src/server/channels.ts` — add batch list helpers if needed
- `src/server/releases.ts` — add `listArtifactsForVersions(ids: number[])`
- `src/server/feed.ts`
- `src/server/updaters/tauri.ts` — only if cache is cleaner at `getText`
- `src/lib/object-cache.ts` (create) — optional tiny Map cache
- Tests that break because of import/order — fix assertions only if order
  was unspecified; do not weaken them

**Out of scope**:
- Pagination / dropping artifacts from historical versions
- SSR loader calling dashboard functions instead of HTTP
- GSAP / ReDoc / fonts
- Hit-bucket indexes
- `lastUsedAt` throttling

## Git workflow

- Branch: `advisor/006-dashboard-and-feed-perf`
- Commit: `perf: batch dashboard queries and cache feed metadata`
- Do NOT push.

## Steps

### Step 1: Batch dashboard queries

Implement (names may vary) in `channels.ts` / `releases.ts`:

- `listChannelsForApps(appIds: number[]): Channel[]` —
  `where(inArray(channels.appId, appIds))` then group in JS.
- `listVersionsForChannels(channelIds: number[]): Version[]` —
  `inArray(versions.channelId, channelIds)`.
- `listArtifactsForVersions(versionIds: number[]): Artifact[]` —
  `inArray(artifacts.versionId, versionIds)`.

Empty id arrays must not generate invalid SQL — return `[]` without
querying.

Rewrite `appSummaries` and `appDetail` to use these. Preserve sort order
exactly as today's helpers.

**Verify**: `npm test -- tests/app-api.test.ts tests/release-flow.test.ts`
plus any test that snapshots app detail. If none exist, add one
characterization test: create an app with two channels and two versions,
`appDetail` returns the same channel/version/artifact counts and
`currentVersion` as before (compare against a fixture you record from the
old logic **in the test**, not from memory).

### Step 2: In-process metadata cache

Create `src/lib/object-cache.ts`:

```
const cache = new Map<string, string>()
export function cachedText(key: string, load: () => Promise<string>): Promise<string>
export function clearObjectCache(): void  // tests
```

Key for feed metadata: `meta:${app.id}:${artifact.s3Key}` or
`meta:${versionId}:${filename}`. Version+filename is enough (immutable).

In `feed.ts` Electron path, wrap `getObjectText`. In Tauri
`generateFeedDocument`, wrap each `getText` call (the adapter already
receives `getText` — wrap at the `feed.ts` call site:

```
getText: (key) => cachedText(`s3:${app.id}:${key}`, () => getObjectText(s3, key))
```

so you do not edit Tauri adapter logic if possible).

Memory: unbounded Map is OK for self-host (dozens of versions). Do not
add LRU unless it is <20 lines.

**Verify**: `tests/tauri-updater.test.ts` and `tests/hits.test.ts` still
pass (hits still increment — cache must not skip `recordHit`). Add a test
with a mocked `getObjectText` counter: two `resolveFeedRequest` for the
same latest.yml increment the mock once, `recordHit` twice.

### Step 3: Full suite

**Verify**: `npm test` → pass. `npm run lint` → 0. `npm run typecheck` → 0.

## Test plan

- App detail shape/counts unchanged
- Feed cache: one S3 read, two hits
- Empty app list / app with zero versions does not crash (`inArray` empty)

## Done criteria

- [ ] `appSummaries` / `appDetail` do not call `listArtifacts` / `listChannels`
      / `listVersions` inside a per-item `.map` (grep the `.map` bodies)
- [ ] Feed metadata S3 read is cached per process
- [ ] `recordHit` still runs every request
- [ ] `npm test` exits 0
- [ ] JSON field names unchanged
- [ ] No files outside scope

## STOP conditions

- `inArray` empty-array behavior is unsafe and you cannot special-case
  empty without changing results — STOP.
- Caching would serve metadata from a **different** version than
  `currentVersionId` — the cache key must include version id or s3 key.
- Plan 003's `includeKeys` is present and you cannot keep it — STOP and
  rebase onto 003 rather than dropping it.

## Maintenance notes

Reviewer: confirm delete-version / promote do not need cache invalidation
because keys include version id or full s3 key (old version keys simply
go unused). If someone overwrites an S3 object in place (unsupported),
cache can be stale until process restart — acceptable.
