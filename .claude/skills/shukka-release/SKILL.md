---
name: shukka-release
description: 'Release a new Shukka version and publish the matching SDKs. Use when asked to release/tag/publish Shukka, bump the app version, create a GitHub Release, regenerate the SDKs from a new contract, or publish npm @shukka/sdk / pub.dev shukka_sdk / crates.io shukka — including 发版、发新版本、发布 SDK. Covers the version bump and Action references, tag → multi-arch Docker image, release notes, manual SDK regeneration, and the trusted-publishing tag flow.'
---

# Release Shukka and its SDKs

One release spans two repositories. Shukka ships first; the SDK is regenerated from
the tag and then published. Everything is tag-driven — registries are fed by GitHub
Actions through trusted publishing (OIDC), never by hand.

## Repositories

Both live in this monorepo as submodules. Run commands from the submodule, not the
monorepo root.

| Path | GitHub | Releases |
|---|---|---|
| `shukka/` | `shukka-app/shukka` | `vX.Y.Z` → Docker image + GitHub Release (product) |
| `sdk/` | `shukka-app/sdk` | `vX.Y.Z` → npm, pub.dev, crates.io (SDKs) |

The `docs/` submodule is not part of the version flow.

## 0. Preconditions

```bash
cd shukka
git status --short          # only untracked plans/ is fine
git pull --ff-only origin main
gh run list --repo shukka-app/shukka --branch main --limit 6
```

`main` must be green (`CI`, `Docker test`, `Action test`). If the API contract changed,
the docs snapshots are already committed in `apps/docs/public/` — if they are not,
regenerate and commit them in the feature PR:

```bash
cd apps/docs && node scripts/sync-openapi.mjs   # writes openapi.json, openapi.zh-CN.json, openapi.v3.0.json
```

## 1. Bump the Shukka version (commit on main)

Semver: additive API/feature → minor, fixes → patch, breaking → major.

1. `apps/shukka/package.json` → `"version": "X.Y.Z"` (shown in the panel's Settings page).
2. Every `shukka-app/shukka@vOLD` reference must become `@vX.Y.Z`:
   - `README.md`
   - `.agents/skills/shukka-ops/SKILL.md`
   - `apps/docs/content/docs/en-US/ci.md`, `apps/docs/content/docs/zh-CN/ci.md`
   - `apps/shukka/docs/prd/monorepo-layout.md`
   - `apps/shukka/src/features/apps/integration-snippets.ts`
   - `apps/shukka/src/features/apps/integration-panel.tsx`
   - `apps/shukka/tests/integration-snippets.test.ts`

   ```bash
   sed -i '' 's|shukka-app/shukka@vOLD|shukka-app/shukka@vNEW|g' <files…>
   cd apps/shukka && npx vitest run tests/integration-snippets.test.ts && pnpm typecheck
   ```

3. Commit `chore: release vX.Y.Z` and push to `main`.

## 2. Tag Shukka

```bash
git tag -a vX.Y.Z -m "Release vX.Y.Z"
git push origin vX.Y.Z
```

`.github/workflows/docker.yml` builds `linux/amd64` + `linux/arm64` and pushes
`ghcr.io/shukka-app/shukka:{X.Y.Z, X.Y, X}` (plus `latest` from the `main` build).

```bash
gh run list --workflow docker.yml --limit 3
gh run watch <run-id> --exit-status
docker buildx imagetools inspect ghcr.io/shukka-app/shukka:X.Y.Z \
  --format '{{json .Manifest}}' | jq -r .digest
```

Optional smoke test of the published artifact: run it on a spare port and curl
`/api/health` for `{"status":"ok","db":"ok"}`.

## 3. Create the GitHub Release

```bash
gh release create vX.Y.Z --title "vX.Y.Z — <theme>" --notes "$(cat <<'EOF'
## What's new

- ...

## Upgrade

Docker image: `ghcr.io/shukka-app/shukka:X.Y.Z` (`linux/amd64`, `linux/arm64`).

Image digest: `sha256:…`.

GitHub Action: `shukka-app/shukka@vX.Y.Z`.

No database migration. Existing feeds and clients are unaffected.

**Full changelog:** https://github.com/shukka-app/shukka/compare/vPREV...vX.Y.Z
EOF
)"
```

Notes style: user-visible bullets only (`What's new`), then `Upgrade` with image, digest,
Action ref, migration status, compare link. Confirm it is the latest release:

```bash
gh api repos/shukka-app/shukka/releases/latest --jq .tag_name
```

## 4. Regenerate the SDKs (manual — deliberately not automated)

There is no `repository_dispatch`; do not add one (it needs a cross-repo PAT).

```bash
gh workflow run generate.yml --repo shukka-app/sdk \
  -f spec_base_url=https://raw.githubusercontent.com/shukka-app/shukka/vX.Y.Z/apps/docs/public
```

Pin to the tag, not `main`: the generated code must match the release. Watch it:

```bash
gh run list --repo shukka-app/sdk --workflow generate.yml --limit 1
gh run watch <run-id> --exit-status
```

- Drift → it opens the `regenerate/openapi` PR. Review `spec/openapi.lock.json` first,
  then the generated packages (`packages/typescript`, `packages/dart`; Rust and Swift
  embed `openapi.json` and generate at build time). Merge it.
- No drift → no PR. Skip step 5: the contract did not change, so there is nothing to
  publish.

## 5. Release the SDKs

SDK versions are independent of Shukka's (currently `0.1.x`); bump patch for
regenerated code, minor for new capabilities, major for breaking contract changes
(renamed `operationId`, removed fields).

```bash
cd sdk          # from this monorepo
git pull --ff-only origin main
git tag -a vX.Y.Z -m "Release vX.Y.Z"
git push origin vX.Y.Z
```

`.github/workflows/publish.yml` runs `verify` (npm pack, `dart pub publish --dry-run`,
`cargo publish --dry-run`, `swift package describe`) then publishes in parallel:

| Registry | Package | Auth |
|---|---|---|
| npm | `@shukka/sdk` | trusted publishing, `--provenance` |
| pub.dev | `shukka_sdk` | automated publishing, tag `v{{version}}` |
| crates.io | `shukka` | trusted publishing |

Watch the run. If a job fails, fix the cause and rerun the failed jobs
(`gh run rerun <run-id> --failed`) — **never re-push a tag whose version a registry
already accepted**.

Verify all three:

```bash
npm view @shukka/sdk version
curl -s https://pub.dev/api/packages/shukka_sdk | jq -r .latest.version
curl -s https://crates.io/api/v1/crates/shukka | jq -r .crate.max_version
npm view @shukka/sdk@X.Y.Z dist.attestations   # provenance: npm publish + SLSA
```

## One-time trusted-publishing setup (already done)

- npm: `npm trust github @shukka/sdk --repo shukka-app/sdk --file publish.yml --allow-publish`
- pub.dev → Automated publishing: repo `shukka-app/sdk`, tag pattern `v{{version}}`,
  workflow `publish.yml`, no required environment.
- crates.io → Trusted Publishing: repo `shukka-app/sdk`, workflow `publish.yml`, no environment.

Each registry only accepts a trusted publisher after the package's first
hand-published version.

## Gotchas

- Never push the tag before the version-bump commit — the image build checks out the tag.
- A release PR that changes the contract must include the regenerated
  `apps/docs/public/openapi*.json`, or the SDK will regenerate stale models.
- `operationId` renames break every generated SDK; treat them as SDK-major.
- Docker build needs both architectures; a missing QEMU step fails only at push.
- `npm view` can report the previous version for a few minutes after a successful publish
  (registry CDN lag, seen on 0.1.1 and 0.1.3). Re-check before assuming the job failed;
  `npm view @shukka/sdk@X.Y.Z dist.attestations` returning provenance is the real confirmation.
- The panel's Settings page version comes from `apps/shukka/package.json` — it lags
  behind the tag until this bump is pushed.
- After a release, bump the submodule pointers in the monorepo (this repo):
  `git -C shukka … && git add shukka sdk && git commit -m "chore: point the submodules at vX.Y.Z"`.
