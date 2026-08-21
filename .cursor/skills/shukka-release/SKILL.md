---
name: shukka-release
description: Cuts a shukka release with bumpp (version bump, commit, v*.*.* tag, push; no npm publish) so GitHub Actions can build the GHCR image. Use when releasing, tagging, versioning, or shipping shukka, or when the user asks for a patch, minor, or major.
---

# Shukka release

Release lives in the `shukka` submodule (`github.com/shukka-app/shukka`), not this monorepo. The artifact is the GHCR image from `.github/workflows/docker.yml` on `v*.*.*` tags. Never `npm publish`.

Unset `GIT_DIR` and `GIT_WORK_TREE` before every git/bumpp command in `shukka/` — the monorepo workspace can point them at the parent repo.

```bash
cd shukka
unset GIT_DIR GIT_WORK_TREE
```

## Ask first

If the user did not already name **patch**, **minor**, or **major**, ask with `AskQuestion` (those three options only). Do not guess.

## Preconditions

Stop if any fail:

1. `git status -sb` is on `main`, tracking `origin/main`, and clean.
2. `git pull --ff-only origin main`
3. `git fetch origin --tags`
4. `origin` is `https://github.com/shukka-app/shukka.git`

Do not release from a feature branch or with a dirty tree.

## Next version

Git tags are the source of truth. `package.json` can lag.

1. Read `version` from `package.json`.
2. List remote tags: `git ls-remote --tags origin 'v*'`
3. Parse `vMAJOR.MINOR.PATCH` (ignore pre-release unless the user asked for one).
4. `base` = the highest tag. Tell the user the tag list and that `base`.
5. If `package.json` ≠ `base`, sync `package.json` and the root `version` fields in `package-lock.json` to `base`, then commit only those files (`chore: sync package.json version to <base>`). Do not tag that commit.
6. Apply the requested kind to `base`. If `v{next}` already exists, increment until free (same kind).

## Bump

```bash
npx bumpp <exact-version> --yes
```

Use the exact version, not `patch`/`minor`/`major` — those follow stale `package.json`. bumpp already commits, tags `v%s`, and pushes. Do not add `npm publish`.

If bumpp cannot push, `git push origin HEAD && git push origin "v<exact-version>"`.

## After

1. Confirm `origin` has `v<exact-version>` on the just-pushed `main` commit.
2. Give the user the tag URL and that `Docker` should build `ghcr.io/shukka-app/shukka:<version>`.
3. Do not create a GitHub Release unless asked.
4. Do not commit the monorepo submodule pointer unless asked.
