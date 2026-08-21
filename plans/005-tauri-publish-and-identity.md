# Plan 005: First-class Tauri publish + Action identity + notes snippet

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. Do **not** update `plans/README.md`.
>
> **Drift check (run first)**: `git diff --stat 9afada0..HEAD -- scripts/shukka-upload.mjs action.yml src/features/apps/integration-snippets.ts README.md .agents/skills/shukka-ops tests/shukka-upload.test.ts`
> On mismatch, STOP.

## Status

- **Priority**: P1
- **Effort**: M
- **Risk**: LOW
- **Depends on**: none (if 001 already added `tests/shukka-upload.test.ts`,
  update the `latest.json` case from "throws" to "reads version")
- **Category**: bug
- **Planned at**: commit `9afada0`, 2026-08-21

## Why this matters

The server has a full Tauri adapter. The GitHub Action and
`scripts/shukka-upload.mjs` still fail with "No electron-updater .yml
metadata file" unless `version` / `SHUKKA_VERSION` is set. Integration
snippets and README still say `uses: akarachen/shukka@main` and describe
an Electron-only product. Release notes have a public API but no
Integration example.

## Current state

`scripts/shukka-upload.mjs` `versionFromMetadata` (lines 49–54):

```
const metadata = files.find((file) => /\.ya?ml$/i.test(file.filename))
if (!metadata) fail('No electron-updater .yml metadata file found in the directory')
const match = (await readFile(metadata.path, 'utf8')).match(/^version:\s*(.+)$/m)
```

`action.yml`:
- `description`: 'Publish an electron-builder output directory…'
- `directory` description mentions `latest*.yml` only
- `version` description: 'read from the .yml metadata when omitted'
- `author: 'akarachen'`

`src/features/apps/integration-snippets.ts` githubAction + agentCli use
`akarachen/shukka` (lines 54, 83, 121, 150).

`README.md` title/lede is Electron-only; Action example is
`uses: akarachen/shukka@main`.

Canonical repo in `package.json` is `github.com/shukka-app/shukka`.
GHCR image is `ghcr.io/shukka-app/shukka`. Use
`uses: shukka-app/shukka@v1` (or `@v1.0.2` if you want a pin — prefer
`@v1` floating major once v1 exists; package version is `1.0.2`).

Public notes: `GET /api/v1/apps/{appSlug}/channels/{channel}/notes?from&to&locale`
(`docs/spec.md` Release notes section). No auth.

Do **not** implement webhooks (plan 009).

## Commands you will need

| Purpose | Command | Expected on success |
|---------|---------|---------------------|
| Uploader tests | `npm test -- tests/shukka-upload.test.ts` | pass |
| i18n if snippet strings change | `npm test -- tests/i18n.test.ts` | pass |
| Full suite | `npm test` | pass |
| Lint | `npm run lint` | exit 0 |

## Scope

**In scope**:
- `scripts/shukka-upload.mjs`
- `action.yml`
- `src/features/apps/integration-snippets.ts`
- `README.md` (product lede + Action example + one Tauri bullet)
- `.agents/skills/shukka-ops/SKILL.md` and
  `.agents/skills/shukka-ops/references/api.md` — Action owner + Tauri
  `latest.json` version detection + notes GET example
- `skills/shukka-publish/SKILL.md` if it mentions yml-only version or
  `akarachen/shukka`
- `tests/shukka-upload.test.ts` (create if 001 did not)
- `docs/spec.md` GitHub Action bullet if it still says yml-only

**Out of scope**:
- Server Tauri adapter logic
- Changing default `release: false`
- Marketplace publish
- Sparkle

## Git workflow

- Branch: `advisor/005-tauri-publish-and-identity`
- Commit: `feat: detect Tauri latest.json version in the publish action`
- Do NOT push.

## Steps

### Step 1: Version detection

In `versionFromMetadata`:

1. If any `.ya?ml` exists, keep today's yml parse (Electron).
2. Else if `latest.json` exists, `JSON.parse` and read `.version` as a
   string. On missing/invalid JSON, `fail` with a clear message
   (`Could not read "version" from latest.json`).
3. Else `fail` mentioning both `latest*.yml` and `latest.json`.

Precedence: explicit `version` input / `SHUKKA_VERSION` still wins
(existing `main()` line ~100).

**Verify**: tests:
- yml `version: 2.0.0` → `2.0.0`
- only `latest.json` `{"version":"1.4.2","platforms":{}}` → `1.4.2`
- empty dir of metadata → fail (no throw of uncaught TypeError)

### Step 2: `action.yml` copy

Update `description`, `directory`, and `version` input descriptions to
name Electron `latest*.yml` **or** Tauri `latest.json`. Keep `author`
unless you also change it to `shukka-app` — do that.

**Verify**: `npm run lint:actions` if `actionlint` is installed; if the
binary is missing, skip and note it. Do not add a dependency for this.

### Step 3: Canonical Action identity

Replace `akarachen/shukka@main` with `shukka-app/shukka@v1` in:
- `src/features/apps/integration-snippets.ts` (all four sites)
- `README.md` publish example
- agent skill files listed in scope
- `skills/shukka-publish/SKILL.md` if present

Agent CLI tarball URL: `https://github.com/shukka-app/shukka/archive/${skillRef}.tar.gz`

**Verify**: `rg -n 'akarachen/shukka' README.md src/features/apps/integration-snippets.ts .agents skills` → no matches
(except historical ADR `docs/adr/composite-github-action.md`, which is
**out of scope** — do not edit superseded ADRs).

### Step 4: Release-notes Integration snippet

Add an optional fifth snippet **or** append a short commented block to
`httpApi` / `mainProcess` for both Electron and Tauri:

```
# Release notes (public, no auth) — only if the app enabled Release log
curl "${serverUrl}/api/v1/apps/${slug}/channels/${channel}/notes?from=1.0.0&locale=en-US"
```

Do **not** add a new Integration UI step number unless the panel already
renders a dynamic list of snippets — read
`src/features/apps/integration-panel.tsx` first. If it maps a fixed
`builderConfig | mainProcess | githubAction | httpApi | agentCli`,
append to `httpApi` only. Do not expand the TypeScript union unless the
panel iterates `Object.values`.

**Verify**: `npm test` still passes. If i18n captions mention "four
snippets", update en/zh together.

### Step 5: README product sentence

Change the lede from Electron-only to Electron **and** Tauri. Add one
bullet that Tauri feed is JSON at `/api/update/{app}/{channel}`.

**Verify**: `npm test` → pass. `npm run lint` → 0.

## Test plan

- `tests/shukka-upload.test.ts` as in step 1
- No new panel tests unless i18n keys change (`tests/i18n.test.ts`)

## Done criteria

- [ ] Tauri directory without `SHUKKA_VERSION` gets version from `latest.json`
- [ ] Electron yml path unchanged
- [ ] No `akarachen/shukka` in README, snippets, or ops/publish skills
- [ ] Notes curl appears in Integration HTTP snippet
- [ ] `npm test` exits 0
- [ ] No files outside scope

## STOP conditions

- `latest.json` in the wild uses a different version field — if tests in
  `tests/tauri-updater.test.ts` show the server reads another field, match
  **that** field, not this plan's guess. Read `src/server/updaters/tauri.ts`
  `parseMetadata` before implementing.
- Integration panel cannot accept an extra snippet without a large UI
  rewrite — then append to `httpApi` only.

## Maintenance notes

Reviewer: `@v1` requires the Action repo to have a `v1` tag/release.
If only `v1.0.2` exists, pin `@v1.0.2` and say so in NOTES.
Historical ADR `composite-github-action.md` will still mention
`akarachen` — leave it.
