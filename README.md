<div align="center">

# 🔔 check-pr-labels-on-push-action

**GitHub Action that tells a push workflow whether its PR has all required labels**

This repo **is** Glovo's fork of [`shioyang/check-pr-labels-on-push-action`](https://github.com/shioyang/check-pr-labels-on-push-action), with one change of its own: the action runs on the `node20` runtime instead of `node16` (commit `d7beb17`). Workflows use it as `Glovo/check-pr-labels-on-push-action@v1`. It **is not** a service and **not** a merge gate: it never fails the job because a label is missing. It only sets an output, and the calling workflow decides what to do with it.

[![TypeScript](https://img.shields.io/badge/TypeScript-4.9.4-3178C6?logo=typescript&logoColor=white)](package.json)
[![Node.js](https://img.shields.io/badge/Node.js-20-339933?logo=nodedotjs&logoColor=white)](action.yml)
[![@actions/core](https://img.shields.io/badge/%40actions%2Fcore-1.10.0-2088FF?logo=github&logoColor=white)](package-lock.json)
[![@actions/github](https://img.shields.io/badge/%40actions%2Fgithub-5.1.1-2088FF?logo=github&logoColor=white)](package-lock.json)
[![ncc](https://img.shields.io/badge/ncc-0.36.0-000000?logo=vercel&logoColor=white)](package-lock.json)
<br/>
[![GitHub Action](https://img.shields.io/badge/GitHub_Action-%40v1-2088FF?logo=githubactions&logoColor=white)](action.yml)
<br/>
[![License](https://img.shields.io/badge/license-MIT-blue)](LICENSE)
[![Fork](https://img.shields.io/badge/fork_of-shioyang%2Fcheck--pr--labels--on--push--action-lightgrey?logo=github)](#ownership--support)

</div>

---

## Table of contents

- [Overview](#overview)
- [Build & local development](#build--local-development)
- [Code quality](#code-quality)
- [CI/CD](#cicd)
- [Repository layout](#repository-layout)
- [Contributing & conventions](#contributing--conventions)
- [Ownership & support](#ownership--support)

---

## Overview

A workflow that runs on `push` knows the commit, not the pull request it came from. This action
finds that pull request and checks its labels. It calls the GitHub API for the **pull requests
associated with the pushed commit** (`listPullRequestsAssociatedWithCommit` with
`github.context.sha`), takes the **first** one, and checks that **every** label in the `labels`
input is on it ([`src/index.ts`](src/index.ts)).

It writes `true` or `false` to the `result` output. The next steps in the workflow read that
output to decide whether to run, for example a fast-track build when a PR carries an
`urgent-hotfix` label.

| | |
|---|---|
| Language / build | TypeScript 4.9.4, compiled by `tsc` and bundled by `@vercel/ncc` 0.36.0 into [`dist/index.js`](dist/index.js) |
| Runtime | GitHub Actions `node20` ([`action.yml`](action.yml)); Node 20.11.0 locally ([`.nvmrc`](.nvmrc)) |
| Dependencies | `@actions/core` 1.10.0, `@actions/github` 5.1.1 ([`package-lock.json`](package-lock.json)) |
| Release | git tag `v1`, pointing at `master` HEAD `3495488` (2024-02-28) |
| Used by | `Glovo/remote-gh-actions`: `fast-track/action.yaml` (label `urgent-hotfix`) and `.github/workflows/release-tags.yaml` (label `stable-release`), per a GitHub code search on 2026-10-29 |

### Inputs and outputs

Defined in [`action.yml`](action.yml).

| Kind | Name | Required | Meaning |
|---|---|---|---|
| Input | `github-token` | yes | The repository token, i.e. `secrets.GITHUB_TOKEN` |
| Input | `labels` | yes | A JSON array of label names, e.g. `'["label-1", "label-2"]'` |
| Output | `result` | — | `true` if **all** the given labels are on the PR, else `false` |

### Usage

```yaml
uses: Glovo/check-pr-labels-on-push-action@v1
with:
  github-token: ${{ secrets.GITHUB_TOKEN }}
  labels: '["label-1", "label-2"]'
```

<details><summary><b>Example workflow</b></summary>

Same shape as this repo's own [`.github/workflows/main.yml`](.github/workflows/main.yml), pointed at this fork:

```yaml
on:
  push:
    branches:
      - master

jobs:
  check_pr_labels_job:
    runs-on: ubuntu-latest
    name: A job to check the PR labels contain given labels
    steps:
      - name: Check PR labels action step
        id: check_pr_labels
        uses: Glovo/check-pr-labels-on-push-action@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          labels: '["enhancement"]'
        timeout-minutes: 5
      - name: See result
        run: echo "${{ steps.check_pr_labels.outputs.result }}"
```

</details>

### How `result` is decided

From [`src/index.ts`](src/index.ts):

| Situation | Outcome |
|---|---|
| The first PR for the commit has every label | `result` = `true` |
| A label is missing, or no PR is linked to the commit (e.g. a direct push) | `result` = `false`; the step still **succeeds** |
| Several PRs are linked to the commit | Only the **first** one in the API response is checked |
| Label names differ only in case | No match: the comparison is exact (`===`) |
| `labels` is not valid JSON | The step **fails** with `Action failed with error: …` |

> [!WARNING]
> `labels` must be a JSON **array**. Valid JSON that isn't an array (e.g. `'"urgent-hotfix"'`)
> is treated as an empty list, so `result` is `true` for every push.

---

## Build & local development

> [!NOTE]
> You need **Node 20.11.0** ([`.nvmrc`](.nvmrc)) and npm ([`package-lock.json`](package-lock.json), lockfile v2).

```bash
nvm use        # Node 20.11.0 from .nvmrc
npm ci         # install exact versions from package-lock.json
npm run build  # tsc: src/ → lib/ (git-ignored), then ncc: lib/index.js → dist/index.js
```

### How the bundle is built

| Step | Tool | Input → output | Config |
|---|---|---|---|
| Compile | `tsc` (TypeScript 4.9.4) | `src/` → `lib/` | [`tsconfig.json`](tsconfig.json): `target` esnext, `module` commonjs, `strict` on, `noImplicitAny` off |
| Bundle | `ncc` (`@vercel/ncc` 0.36.0) | `lib/index.js` → `dist/index.js` | `build` script in [`package.json`](package.json) |

> [!IMPORTANT]
> The runner executes **[`dist/index.js`](dist/index.js)** (`runs.main` in [`action.yml`](action.yml)), never
> `src/`. `dist/` is committed (its line in [`.gitignore`](.gitignore) is commented out); `lib/` is ignored.
> A change to `src/` does nothing until you rebuild and commit `dist/index.js`.

`dist/index.js` was last rebuilt in upstream commit `ac67447` (2023-01-07). Glovo's node20 change
touched only `action.yml` and `.nvmrc`.

---

## Code quality

| Tool | Role |
|---|---|
| TypeScript compiler | Type-checks during `npm run build` with the [`tsconfig.json`](tsconfig.json) settings above |
| Prettier | [`.prettierrc.json`](.prettierrc.json) sets single quotes for `*.yml`. Prettier is not in [`package.json`](package.json) and nothing runs it automatically |

There are no git hooks and no tests in the repo.

---

## CI/CD

| Workflow | Trigger | Effect |
|---|---|---|
| [`main.yml`](.github/workflows/main.yml) | Push to `master` | Runs `shioyang/check-pr-labels-on-push-action@v1.0.9` with `labels: '["enhancement"]'` and prints `result` |

> [!WARNING]
> The only workflow runs the **upstream** action at `v1.0.9` (which still uses `node16`), not this
> repo's code. Nothing in CI builds or tests this fork. No workflow rebuilds `dist/` or moves the `v1` tag.

---

## Repository layout

```text
.
├── .github/workflows/main.yml  # push-to-master smoke run (calls upstream v1.0.9)
├── src/
│   └── index.ts                # ⭐ the action: PR lookup and label comparison
├── dist/
│   └── index.js                # generated by `npm run build` (ncc bundle); committed, this is what runs
├── action.yml                  # action contract: inputs, output, node20 runtime, entry point
├── package.json                # dependencies and the `build` script
├── package-lock.json           # npm lockfile (v2)
├── tsconfig.json               # tsc settings: src/ → lib/ (lib/ is git-ignored)
├── .nvmrc                      # Node 20.11.0
├── .prettierrc.json            # Prettier: single quotes in YAML
└── LICENSE                     # MIT, upstream author
```

---

## Contributing & conventions

1. **Rebuild `dist/`** in the same PR as any `src/` change: run `npm run build` and commit `dist/index.js`.
2. **Merging is not releasing.** Consumers pin `@v1`, so they run whatever commit the `v1` tag points to.
   Moving `v1` changes the action for every consumer at once, including both `Glovo/remote-gh-actions` callers.
3. **Bump Node in both places.** Glovo's only change (`d7beb17`) moved `runs.using` in [`action.yml`](action.yml)
   from `node16` to `node20` and [`.nvmrc`](.nvmrc) from 16.19.0 to 20.11.0 together.
4. **Keep the contract stable.** Consumers reference the `github-token` and `labels` inputs and the `result` output by name.
5. **Commit subjects** use Conventional Commit types (`chore:`, `refactor:`, `doc:`) with no ticket prefix.

There is no `.github/CODEOWNERS` file, so GitHub requests no reviewers automatically.

---

## Ownership & support

- **Team:** none recorded. The repo has no `.github/CODEOWNERS` and no `catalog-info.yaml`.
- **Upstream:** [`shioyang/check-pr-labels-on-push-action`](https://github.com/shioyang/check-pr-labels-on-push-action)
  by Tomohiro (Tomo) SHIOYA, MIT licence ([`LICENSE`](LICENSE)).
