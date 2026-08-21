# Plan 007: Docs, OpenAPI, operator runbook, non-root image

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. Do **not** update `plans/README.md`.
>
> **Drift check (run first)**: `git diff --stat 9afada0..HEAD -- src/server/openapi.ts Dockerfile README.md docs/spec.md docs/prd docs/adr .agents/skills/shukka-ops/references/api.md`
> On mismatch, STOP.

## Status

- **Priority**: P2
- **Effort**: M
- **Risk**: LOW
- **Depends on**: plans/002-auth-hardening.md, plans/005-tauri-publish-and-identity.md
- **Category**: docs
- **Planned at**: commit `9afada0`, 2026-08-21

## Why this matters

`POST /api/v1/upload/init` is missing a requestBody in ReDoc while finalize
has one. Spec cites `docs/prd/container-image.md` and
`docs/adr/ghcr-on-semver-tag.md` — neither file exists. View-roles, theme,
i18n, and updater-adapters PRDs still have unchecked acceptance criteria
for shipped work. Theme-toggle PRD still describes settings tabs that were
superseded by the role menu. The Docker image runs as root and has no
`HEALTHCHECK` even though `GET /api/health` exists. Operators get one
README sentence about backups.

## Current state

`src/server/openapi.ts` lines 163–169: init has `post.responses` only.
Init Zod (`src/routes/api/v1/upload.init.ts:7-13`):
`{ app?, channel, version, createChannel?, files: [{ filename, size? }] }`.

Finalize body is already documented (176–191) — match that style.

`GET /api/health` is implemented (`src/routes/api/health.ts`,
`src/server/health.ts`): 200 `{ status: "ok", db: "ok" }` or 503
`{ status: "degraded", db: "down" }`, `cache-control: no-store`.

`Dockerfile` runtime stage: no `USER`, no `HEALTHCHECK`. Base
`node:24-bookworm-slim` (has a `node` user). `VOLUME ["/data"]`,
`EXPOSE 3000`, `CMD ["node", ".output/server/index.mjs"]`.
`SHUKKA_DATA_DIR=/data`.

`docs/spec.md:98,148-149` reference missing container PRD/ADR.

`docs/prd/view-roles.md`, `theme-toggle.md`, `panel-i18n.md`,
`updater-adapters.md` — unchecked AC. Theme placement decision:
`docs/adr/panel-view-roles.md` + `docs/spec.md:77` (role menu; settings
tabs deferred).

`docs/adr/auth-model.md:20` — password recovery is delete the admin row.

If plan 002 added `rate_limited`, document it here if spec was not already
updated.

## Commands you will need

| Purpose | Command | Expected on success |
|---------|---------|---------------------|
| Docs HTML / i18n | `npm test -- tests/docs-html.test.ts tests/i18n.test.ts` | pass |
| Full suite | `npm test` | pass |
| Lint | `npm run lint` | exit 0 |

No Docker build required (slow, needs qemu). Dockerfile is review-by-read.

## Scope

**In scope**:
- `src/server/openapi.ts`
- `Dockerfile`
- `README.md` (ops table: health, backup, secure cookies, admin reset)
- `docs/spec.md` (only if still missing health/image/rate-limit facts)
- `docs/prd/container-image.md` (create)
- `docs/adr/ghcr-on-semver-tag.md` (create)
- `docs/prd/theme-toggle.md` — status: superseded for settings tabs
- `docs/adr/panel-i18n-and-theme.md` — note theme UX superseded by
  `panel-view-roles.md`
- `docs/prd/view-roles.md`, `docs/prd/panel-i18n.md`,
  `docs/prd/updater-adapters.md` — mark shipped AC `[x]`
- `.agents/skills/shukka-ops/references/api.md` — init body + Tauri
  metadata rules + `updaterKind` on create-app
- `tests/docs-html.test.ts` only if a pinned string breaks

**Out of scope**:
- Rewriting ReDoc renderer
- Multi-user docs
- Implementing webhooks
- Changing `npm ci` (plan 008)
- GSAP removal

## Git workflow

- Branch: `advisor/007-docs-openapi-ops-image`
- Commit: `docs: document init body, image tags, and operator runbook`
- Do NOT push.

## Steps

### Step 1: OpenAPI init requestBody

Add `requestBody` to `POST /api/v1/upload/init` matching the Zod schema.
Describe in the summary that Electron requires at least one `.yml` and
Tauri requires `latest.json` and/or artifact+`.sig` pairs (see spec).

**Verify**: `rg -n "requestBody" src/server/openapi.ts` shows init and
finalize. `npm test -- tests/docs-html.test.ts` pass.

### Step 2: Missing image PRD + ADR

Write short files (not essays):

`docs/prd/container-image.md`: problem (need a runnable image), users
(operators), goals (GHCR on `vMAJOR.MINOR.PATCH` tags; PR does not
publish; `main` publishes `latest` + `sha-*`), non-goals (Helm chart),
acceptance pointing at `.github/workflows/docker.yml` tag rules.

`docs/adr/ghcr-on-semver-tag.md`: context, decision (docker metadata
action tags as in `docker.yml` lines 44–49), alternatives (only latest),
trade-offs (major 0 does not publish `:0`).

Keep vocabulary aligned with `docs/spec.md` Runtime image section.

**Verify**: files exist; spec links resolve.

### Step 3: Mark shipped PRDs and supersede theme tabs

- Check boxes that match shipped UI (role menu, adapters, i18n). Do not
  invent completed AC you did not verify — if unsure, add a one-line
  **Status: shipped** header and leave boxes rather than lying.
- `theme-toggle.md`: add Status: **Superseded for settings Appearance
  / Account tabs** by `docs/prd/view-roles.md` / `docs/adr/panel-view-roles.md`.
  Theme toggle lives in the role menu; settings page remains password +
  about.
- `docs/adr/panel-i18n-and-theme.md`: add a Status note that theme
  placement was superseded; i18n cookie/SSR parts still stand.

**Verify**: no new unchecked AC that contradicts spec.

### Step 4: Operator README + healthcheck + non-root

README table / "Run it" section, add:

| Variable | Default | Purpose |
| `SHUKKA_SECURE_COOKIES` | unset | Set `1` to force `Secure` on the session cookie (or terminate TLS and forward `X-Forwarded-Proto: https`) |

Backup: whole `SHUKKA_DATA_DIR` (SQLite + `encryption.key`). Restore =
copy the directory back; a DB without the key cannot decrypt S3 secrets.

Admin reset: delete the singleton `admin` row (id=1) and re-open `/setup`.
Do **not** invent a CLI. Quote that this is the ADR recovery path.

`Dockerfile` runtime:

```
RUN mkdir -p /data && chown -R node:node /app /data
USER node
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:'+(process.env.PORT||3000)+'/api/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
```

`node:24-bookworm-slim` includes `node`. Do not add curl. `USER node`
must come after `chown`. `VOLUME ["/data"]` stays.

**Verify**: read the Dockerfile — `USER node` appears once in the runtime
stage; `HEALTHCHECK` present; build stage unchanged.

### Step 5: api.md Tauri + init + updaterKind

Update `.agents/skills/shukka-ops/references/api.md`:
- upload init file rules per `updaterKind`
- create-app payload includes `updaterKind: "electron" | "tauri"`
- notes GET as already specified

**Verify**: `npm test` → pass. `npm run lint` → 0.

## Test plan

No new product tests unless OpenAPI generation is snapshotted — do not
add a brittle full-spec snapshot. `tests/docs-html.test.ts` is the pin.

## Done criteria

- [ ] OpenAPI init has requestBody with channel, version, files
- [ ] `docs/prd/container-image.md` and `docs/adr/ghcr-on-semver-tag.md` exist
- [ ] Theme-toggle PRD marked superseded for settings tabs
- [ ] Dockerfile runtime is non-root and has HEALTHCHECK against `/api/health`
- [ ] README documents backup, admin reset, Secure cookies
- [ ] api.md mentions Tauri and `updaterKind`
- [ ] `npm test` exits 0
- [ ] No files outside scope

## STOP conditions

- Health route path is not `/api/health` in this tree.
- `USER node` cannot write `/data` without a different uid — STOP and
  report rather than switching to a custom user with a guessed uid.
- You would need to rewrite `docs/spec.md` wholesale — only patch the
  cited gaps.

## Maintenance notes

Reviewer: image `chown /app` after `COPY` — if a later COPY overwrites
ownership, `USER node` may not read `.output`. Keep `chown` **after**
the last COPY in the runtime stage.
