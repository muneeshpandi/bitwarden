# Fork Sync & Build Workflow

This document describes how to keep this fork's local changes (`devopsminds/production`)
in sync with the official Bitwarden repository while continuously benefiting from new
upstream features and fixes.

## Repository Layout

| Remote     | URL                                         | Purpose                          |
| ---------- | ------------------------------------------- | -------------------------------- |
| `origin`   | `git@github.com:muneeshpandi/bitwarden.git` | Your fork (backup of your work)  |
| `upstream` | `https://github.com/bitwarden/clients.git`  | Official Bitwarden repo (source) |

> **Fork renamed.** The fork is now `muneeshpandi/bitwarden`; it was previously
> `muneeshpandi/bitwarden-clients`. GitHub redirects the old URL, so existing clones keep
> working, but point `origin` at the new name to avoid surprises:
>
> ```bash
> git remote set-url origin git@github.com:muneeshpandi/bitwarden.git
> git remote -v   # confirm
> ```
>
> The upstream repo is unaffected — it stays `bitwarden/clients`.

| Branch                   | Purpose                                          |
| ------------------------ | ------------------------------------------------ |
| `main`                   | Mirror of upstream `main` (kept clean, no edits) |
| `devopsminds/production` | Your working branch with all your local changes  |

**Prerequisites:** Node `>=24.17.0` and npm `~11` (`engines` in the root `package.json`).
`.nvmrc` pins `v24` and is the source of truth — CI reads it directly, so run `nvm use`
before any npm command. A Node below 24.17.0 fails the `engines` check and produces
confusing install and build errors.

---

## One-Time Setup

Only run these if the remotes are not already configured. Verify with `git remote -v` first.

```bash
# Add the official repo as "upstream" (skip if it already exists)
git remote add upstream https://github.com/bitwarden/clients.git

# Never push to upstream by accident — disable pushing to it
git remote set-url --push upstream DISABLE
```

---

## The Sync Routine (run whenever you want the latest upstream changes)

The strategy is **rebase**: your local commits are lifted and replayed on top of the
newest official `main`. This keeps your changes on top and history linear.

### Step 1 — Save your work first (safety net)

```bash
# Make sure you have no uncommitted changes; commit or stash them
git status

# Option A: commit them
git add -A && git commit -m "wip: local changes before sync"

# Option B: stash them
git stash push -m "before-sync"
```

### Step 2 — Fetch the latest from the official repo

```bash
git fetch upstream --prune
```

### Step 3 — Update your clean mirror of main

```bash
git checkout main
git merge --ff-only upstream/main   # fast-forward only; fails if main has stray commits
git push origin main                # keep your fork's main up to date
```

### Step 4 — Rebase your changes on top of the latest upstream

```bash
git checkout devopsminds/production
git rebase upstream/main
```

If there are conflicts:

```bash
# Edit the conflicted files, then:
git add <resolved-files>
git rebase --continue

# To abort and return to the pre-rebase state at any point:
git rebase --abort
```

If you stashed in Step 1, restore your work:

```bash
git stash pop
```

### Step 5 — Push your updated branch to your fork

Because rebase rewrites history, use a **safe** force push:

```bash
git push --force-with-lease origin devopsminds/production
```

`--force-with-lease` refuses to overwrite work on the remote you haven't seen,
which is much safer than a plain `--force`.

---

## Build After Syncing

Always switch Node first. `.nvmrc` pins `v24` and CI reads the same file, so `nvm use`
keeps local builds aligned with CI.

```bash
nvm use          # reads .nvmrc -> Node 24

# Clean install to match the refreshed lockfile from upstream
npm ci

# If npm ci fails due to lockfile drift, fall back to:
npm install

# Verify the tree is healthy
npm run lint
npm test
```

Then build/run the client you need (each app has its own scripts under `apps/*`), e.g.:

```bash
nvm use

# Web
npm run build --workspace @bitwarden/web-vault

# Browser extension — Firefox (this fork's target)
npm run build:firefox --workspace @bitwarden/browser

# Desktop
npm run build --workspace @bitwarden/desktop

# CLI
npm run build --workspace @bitwarden/cli
```

> **Firefox is not the default.** In `apps/browser/package.json`, the bare `build` script
> aliases `build:chrome`, so `npm run build --workspace @bitwarden/browser` produces a
> Chrome MV3 bundle. Name the Firefox script explicitly every time.

### Firefox browser extension scripts

Run these from the repo root with `--workspace @bitwarden/browser`, or from
`apps/browser` without the workspace flag.

| Goal                       | Script                    | Notes                                             |
| -------------------------- | ------------------------- | ------------------------------------------------- |
| Dev build                  | `build:firefox`           | MV2; unpacked output in `apps/browser/build/`     |
| Dev build, rebuild on save | `build:watch:firefox`     | MV2 watch mode                                    |
| Dev build, MV3             | `build:watch:firefox:mv3` | Sets `MANIFEST_VERSION=3`                         |
| Production build           | `build:prod:firefox`      | Sets `NODE_ENV=production`                        |
| Packaged zip               | `dist:firefox`            | Prod build + `apps/browser/dist/dist-firefox.zip` |
| Packaged zip, MV3          | `dist:firefox:mv3`        | Prod build + MV3 manifest                         |

`webpack.base.js` defaults to manifest v2 unless `MANIFEST_VERSION=3` is set, and it
defaults `BROWSER` to `chrome` when unset — so both variables matter.

```bash
nvm use

# One-off Firefox build
npm run build:firefox --workspace @bitwarden/browser

# Iterating: rebuild on every file change
npm run build:watch:firefox --workspace @bitwarden/browser

# Shippable zip
npm run dist:firefox --workspace @bitwarden/browser
```

Load the result in Firefox via `about:debugging` → This Firefox → Load Temporary Add-on,
then pick `apps/browser/build/manifest.json`.

The commercial (`bit-`) variants exist for each of the above — `build:bit:firefox`,
`build:bit:watch:firefox`, `build:bit:prod:firefox`, `dist:bit:firefox` — and build with
`bitwarden_license/bit-browser/webpack.config.js`.

> Check the individual `apps/<client>/package.json` for the exact `build`, `build:prod`,
> and `dev`/`watch` script names for the client you are targeting.

---

## Quick Reference (copy/paste the whole routine)

```bash
git fetch upstream --prune
git checkout main && git merge --ff-only upstream/main && git push origin main
git checkout devopsminds/production
git rebase upstream/main
git push --force-with-lease origin devopsminds/production
nvm use
npm ci && npm run lint && npm test
npm run build:firefox --workspace @bitwarden/browser
```

---

## Merge Alternative (if you dislike rewriting history)

If you'd rather not force-push, use merge instead of rebase. Your history will contain
merge commits, but no force-push is ever needed.

```bash
git fetch upstream --prune
git checkout devopsminds/production
git merge upstream/main          # resolve conflicts if any, then commit
git push origin devopsminds/production
```

**Rebase vs. merge:** rebase gives a clean, linear history with your changes always on
top (recommended for a personal production branch), but requires `--force-with-lease`.
Merge never rewrites history and never force-pushes, at the cost of merge commits.

---

## Troubleshooting

- **`merge --ff-only` failed on `main`:** you accidentally committed to `main`. Reset it
  to match upstream (this discards local `main` commits — make sure they live on your
  working branch first):
  ```bash
  git checkout main
  git reset --hard upstream/main
  git push --force-with-lease origin main
  ```
- **Repeated conflicts during rebase:** consider keeping local changes small and focused,
  or maintain them as a separate patch/feature set to reduce overlap with upstream.
- **`npm ci` errors after sync:** upstream changed dependencies. Run `npm install` to
  regenerate `package-lock.json`, then commit the updated lockfile.
- **Install or build fails in a confusing way:** check `node -v` before blaming the code.
  Several Node versions are installed via nvm on this machine, so run `nvm use` (or
  `nvm install 24.17.0 && nvm use 24.17.0` if the patch matters) and retry.
- **Build came out as Chrome instead of Firefox:** you ran the bare `build` script, which
  aliases `build:chrome`. Use `build:firefox` — see
  [Firefox browser extension scripts](#firefox-browser-extension-scripts).

```

```
