# ci-app-pipelines

Jenkins pipeline definitions for the **WSO2 API Manager** Gitflow release lifecycle.  
This repository automates cutting, finalizing, and hotfixing releases for [`WSO2-Project/wso2-apim-app`](https://github.com/WSO2-Project/wso2-apim-app).

---

## Table of Contents

- [Overview](#overview)
- [Repository structure](#repository-structure)
- [Version format](#version-format)
- [Gitflow branch model](#gitflow-branch-model)
- [Pipelines](#pipelines)
  - [Jenkinsfile.release — Cut a release](#jenkinsfilerelease--cut-a-release)
  - [Jenkinsfile.production — Finalize a release or hotfix](#jenkinsfileproduction--finalize-a-release-or-hotfix)
  - [Jenkinsfile.hotfix — Start a hotfix](#jenkinsfilehotfix--start-a-hotfix)
- [End-to-end workflows](#end-to-end-workflows)
  - [Standard release](#standard-release)
  - [Hotfix](#hotfix)
- [Jenkins jobs](#jenkins-jobs)
- [Credentials](#credentials)
- [Linting](#linting)
- [Protected branches and safety guards](#protected-branches-and-safety-guards)

---

## Overview

```
develop  ──────●──────────────────────────────●─────────────▶
               │  Jenkinsfile.release          │ (bumped to next -SNAPSHOT)
               └──▶ release/X.Y.Z ────────────┤
                                               │ Jenkinsfile.production
               main ◀─────────────────────────┘ (merged + tagged + release branch deleted)
                 │
                 │  Jenkinsfile.hotfix  (branch from tag)
                 └──▶ hotfix/X.Y.Z ──▶ (Jenkinsfile.production)
```

All git operations (merge, tag, push, branch deletion) are performed by Jenkins using a GitHub App credential; no manual git commands are needed once a pipeline is triggered.

---

## Repository structure

```
ci-app-pipelines/
├── Jenkinsfile.release      # Cuts a release branch from develop
├── Jenkinsfile.production   # Merges release/hotfix to main, tags, bumps develop
├── Jenkinsfile.hotfix       # Creates a hotfix branch from a production tag
├── .groovylintrc.json       # npm-groovy-lint configuration (extends recommended-jenkinsfile)
├── .gitignore               # Excludes _data/ (local Jenkins home volume)
```

---

## Version format

Both repositories share a single `VERSION` file with the following format:

```
<WSO2_VERSION>+<CUSTOM_VERSION>[-SNAPSHOT]
```

| Segment | Example | Meaning |
|---|---|---|
| `WSO2_VERSION` | `4.7.0` | Upstream WSO2 API Manager version |
| `CUSTOM_VERSION` | `0.1.0` | Project-specific semantic version |
| `-SNAPSHOT` suffix | present on `develop`, absent on `release/*` and `main` | Indicates an in-development version |

Examples:

| Branch / state | VERSION value |
|---|---|
| `develop` (in progress) | `4.7.0+0.1.0-SNAPSHOT` |
| `release/0.1.0` (cut) | `4.7.0+0.1.0` |
| `main` (after merge) | `4.7.0+0.1.0` |
| `develop` (after bump) | `4.7.0+0.2.0-SNAPSHOT` |
| `hotfix/0.1.1` | `4.7.0+0.1.1` |

---

## Gitflow branch model

| Branch | Purpose | Created from | Merged into |
|---|---|---|---|
| `main` | Production-ready code, tagged releases | — | — |
| `develop` | Integration branch, always `-SNAPSHOT` | — | — |
| `release/X.Y.Z` | Release stabilization | `develop` | `main` + `develop` |
| `hotfix/X.Y.Z` | Critical production fix | production tag on `main` | `main` + `develop` |
| `feature/*` | Feature development | `develop` | `develop` (PR) |

`main` and `develop` are **protected branches**. The pipelines contain explicit guards that refuse any operation targeting them directly.

---

## Pipelines

### `Jenkinsfile.release` — Cut a release

**Purpose:** Create a `release/X.Y.Z` branch from `develop` and immediately advance `develop` to the next `-SNAPSHOT` version.

**Trigger:** Manual — run on the Jenkins *CRB* job.

**Parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `AUTO_BUMP_MINOR` | Boolean | `true` | Bump `develop` to the next MINOR (`x.Y+1.0-SNAPSHOT`). Uncheck to set the version manually. |
| `NEXT_DEV_VERSION_OVERRIDE` | String | _(empty)_ | Explicit next dev version (`MAJOR.MINOR.PATCH`). Only used when `AUTO_BUMP_MINOR` is unchecked. |

**Stages:**

1. **Checkout develop** — cleans workspace, checks out `develop`, fetches `main` and `develop`.
2. **Compute versions** — reads `VERSION`, validates it is a `X.Y.Z+A.B.C-SNAPSHOT`, determines `RELEASE_VERSION` and `NEXT_DEV_VERSION`.
3. **Guard: release branch must not exist** — aborts if `release/X.Y.Z` already exists on the remote.
4. **Create release branch** — creates `release/X.Y.Z`, strips `-SNAPSHOT` from `VERSION`, commits, and pushes.
5. **Bump develop** — switches back to `develop`, writes the next SNAPSHOT version, commits, and pushes.

**Post-run state:**

```
release/X.Y.Z  →  VERSION: 4.7.0+X.Y.Z          (ready for stabilization)
develop        →  VERSION: 4.7.0+X.(Y+1).0-SNAPSHOT
```

---

### `Jenkinsfile.production` — Finalize a release or hotfix

**Purpose:** Merge a `release/X.Y.Z` or `hotfix/X.Y.Z` branch into `main`, create a semver tag, back-merge into `develop`, bump `develop` to the next patch SNAPSHOT, and delete the release/hotfix branch.

**Trigger:** Manual — typically run immediately after stabilization on the release branch is complete.

**Parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `RELEASE_BRANCH` | String | _(required)_ | Branch to finalize, e.g. `release/1.2.3` or `hotfix/1.2.4`. |

**Stages:**

1. **Guard: validate release branch format** — validates the `release/X.Y.Z` or `hotfix/X.Y.Z` pattern; rejects `main`/`develop`.
2. **Guard: release branch must exist** — verifies the branch exists on the remote.
3. **Guard: release tag must NOT exist** — prevents re-finalizing an already-released version.
4. **Checkout and configure git** — sets the Jenkins CI git identity.
5. **Fetch all required branches** — fetches `main`, `develop`, and the release branch without tags.
6. **Resolve versions from release branch** — reads the `VERSION` file on the release branch; expects `WSO2+CUSTOM` (no `-SNAPSHOT`).
7. **Merge release to main** — `--no-ff` merge of the release branch into `main`.
8. **Tag release on main** — creates and pushes an annotated tag `X.Y.Z`.
9. **Push main** — pushes the updated `main`.
10. **Merge release to develop** — `--no-ff` merge into `develop`.
11. **Bump develop to next SNAPSHOT** — increments the patch digit, writes `X.Y.(Z+1)-SNAPSHOT`, commits.
12. **Push develop** — pushes `develop`.
13. **Delete release branch** — deletes the `release/X.Y.Z` or `hotfix/X.Y.Z` branch from the remote.

**Post-run state:**

```
main           →  VERSION: 4.7.0+X.Y.Z  (tagged X.Y.Z)
develop        →  VERSION: 4.7.0+X.Y.(Z+1)-SNAPSHOT
release/X.Y.Z  →  deleted
```

---

### `Jenkinsfile.hotfix` — Start a hotfix

**Purpose:** Create a `hotfix/X.Y.Z` branch from an existing production tag, updating `VERSION` to the hotfix version.

**Trigger:** Manual.

**Parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `BASE_TAG` | String | _(required)_ | Production tag to branch from, e.g. `1.2.3`. |
| `HOTFIX_VERSION_OVERRIDE` | String | _(empty)_ | Explicit hotfix version. Default: `BASE_TAG` patch + 1. |

**Stages:**

1. **Resolve inputs** — validates `BASE_TAG` format; computes `HOTFIX_VERSION` (patch+1 unless overridden); sets `HOTFIX_BRANCH = hotfix/X.Y.Z`.
2. **Guards: tag exists, hotfix branch free** — verifies the base tag exists and the hotfix branch does not.
3. **Create hotfix branch** — checks out the tag via `FETCH_HEAD`, writes the new `VERSION`, commits, and pushes the branch.

**Post-run state:**

```
hotfix/X.Y.Z  →  VERSION: 4.7.0+X.Y.Z  (branched from tag BASE_TAG)
```

Developers commit fixes directly on the hotfix branch. When ready, run `Jenkinsfile.production` with `RELEASE_BRANCH=hotfix/X.Y.Z`.

---

## End-to-end workflows

### Standard release

```
1. Develop on feature/* branches → merge PRs to develop
2. Trigger Jenkinsfile.release
   └─ release/X.Y.Z created, develop bumped to next SNAPSHOT
3. (Optional) stabilize / QA on release/X.Y.Z
4. Trigger Jenkinsfile.production with RELEASE_BRANCH=release/X.Y.Z
   └─ main merged + tagged, develop bumped, release branch deleted
```

### Hotfix

```
1. Identify regression on a production tag (e.g. 1.2.3)
2. Trigger Jenkinsfile.hotfix with BASE_TAG=1.2.3
   └─ hotfix/1.2.4 created from tag 1.2.3
3. Commit fix(es) on hotfix/1.2.4
4. Trigger Jenkinsfile.production with RELEASE_BRANCH=hotfix/1.2.4
   └─ main merged + tagged 1.2.4, develop bumped, hotfix branch deleted
```

---

## Jenkins jobs

| Job name | Jenkinsfile | Type | Description |
|---|---|---|---|
| `CRB` | `Jenkinsfile.release` | Pipeline | Cuts a release branch from `develop` |
| `Multibranch` | `Jenkinsfile` (in app repo) | Multibranch Pipeline | Scans `wso2-apim-app`, builds all branches and PRs |

The `Multibranch` job polls the `WSO2-Project/wso2-apim-app` repository every 3 minutes (180 s) using the GitHub Branch Source plugin. It discovers branches and both origin and fork pull requests.

The `Jenkinsfile.production` and `Jenkinsfile.hotfix` pipelines should each be configured as separate Pipeline jobs pointing to this repository.

---

## Credentials

All pipelines authenticate to GitHub using a **Jenkins credential ID**. Two credential IDs appear across the files:

| Credential ID | Used in | Description |
|---|---|---|
| `wso2-apim-app-ci` | `Jenkinsfile.production` | GitHub App credential for the production finalization pipeline |
| `github-credentials` | `Jenkinsfile.release`, `Jenkinsfile.hotfix` | GitHub App (or username/token) credential |
| `jenkins-github-app` | Job configs (`_data/`) | GitHub App credential used by Jenkins job definitions |

Credentials must be stored in the Jenkins credential store as **Username with password** or **GitHub App** entries and must have push access to `WSO2-Project/wso2-apim-app`.

---

## Linting

Groovy/Jenkinsfile syntax is checked with [npm-groovy-lint](https://github.com/nvuillam/npm-groovy-lint) using the `recommended-jenkinsfile` ruleset. The only override is `maxNestedBlockDepth: 7`.

```bash
npx npm-groovy-lint Jenkinsfile.release Jenkinsfile.production Jenkinsfile.hotfix
```

---

## Protected branches and safety guards

Every pipeline includes explicit runtime guards to prevent accidental operations on `main` or `develop`:

- Branch name format is validated via regex before any git operation.
- Remote existence of required branches/tags is checked via `git ls-remote` before checkout.
- Duplicate tag creation is rejected (idempotency check).
- `[ci skip]` / `[skip ci]` markers are added to all automated commits to avoid triggering downstream CI loops.
- `disableConcurrentBuilds()` is set on all pipelines to prevent parallel runs against the same branches.
