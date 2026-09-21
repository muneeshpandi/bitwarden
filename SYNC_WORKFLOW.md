# Fork Sync & Build Workflow

This document describes how to keep this fork's local changes (`devopsminds/production`)
in sync with the official Bitwarden repository while continuously benefiting from new
upstream features and fixes.

## Repository Layout

| Remote     | URL                                             | Purpose                          |
| ---------- | ----------------------------------------------- | -------------------------------- |
| `origin`   | `git@github.com:muneeshpandi/bitwarden-clients.git` | Your fork (backup of your work)  |
| `upstream` | `https://github.com/bitwarden/clients.git`      | Official Bitwarden repo (source) |

| Branch                  | Purpose                                            |
| ----------------------- | -------------------------------------------------- |
| `main`                  | Mirror of upstream `main` (kept clean, no edits)   |
| `devopsminds/production`| Your working branch with all your local changes    |

**Prerequisites (from `package.json`):** Node `>=24.17.0`, npm `~11`.

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

```bash
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
# Web
npm run build --workspace @bitwarden/web-vault

# Browser extension
npm run build --workspace @bitwarden/browser

# Desktop
npm run build --workspace @bitwarden/desktop

# CLI
npm run build --workspace @bitwarden/cli
```

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
npm ci && npm run lint && npm test
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
```
