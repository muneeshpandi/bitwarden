# Node version

Always state the Node version to use before giving any command that runs Node, npm,
or a script from `package.json` (install, build, test, lint, storybook, etc.).

## Required version

| Source | Value |
| --- | --- |
| `.nvmrc` | `v24` |
| `engines.node` (root `package.json`) | `>=24.17.0` |
| `engines.npm` (root `package.json`) | `~11` |

Effective requirement: **Node 24, at minimum 24.17.0**, with **npm 11.x**.

`.nvmrc` is the source of truth. CI reads it directly and strips the leading `v`
(see `.github/workflows/chromatic.yml` and `.github/workflows/sdk-update.yml`),
so local and CI versions stay aligned by using `nvm use`.

## How to state it

Prefix Node-dependent command blocks with the switch, so the command is copy-pasteable:

```bash
nvm use          # reads .nvmrc -> Node 24
npm ci
```

If the exact patch matters (a build or engine-check failure), be explicit:

```bash
nvm install 24.17.0 && nvm use 24.17.0
```

## Rules

- Do not suggest Node 20 or 22 for this repo. Other versions are installed on this
  machine via nvm, so `node -v` alone is not proof the right one is active.
- Verify with `node -v` before concluding that a build or test failure is a code
  problem. A version below 24.17.0 fails `engines` checks and can produce
  confusing install and build errors.
- When editing `.nvmrc` or `engines`, change both together and mention that CI
  picks up `.nvmrc` automatically.
