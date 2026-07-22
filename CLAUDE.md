# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

CI/CD pipeline definitions (Jenkins declarative pipelines, written in Groovy) that
automate the release lifecycle of the application repo `WSO2-Project/wso2-apim-app`.
The pipelines themselves run on a Jenkins controller; this repo holds only the
`Jenkinsfile.*` sources and their linting config. There is no application code here.

## Commands

```bash
# Lint the Jenkinsfiles (uses .groovylintrc.json). Requires npm-groovy-lint.
npm-groovy-lint --files "**/Jenkinsfile*"

# Auto-fix lint issues where possible
npm-groovy-lint --fix --files "**/Jenkinsfile*"
```

There is no build or test step — validation happens by linting locally and by the
Jenkins controller parsing/running the pipeline. To validate pipeline syntax against
a live Jenkins, use its "Replay" / "Pipeline Linter" endpoint
(`curl -X POST <JENKINS_URL>/pipeline-model-converter/validate`).

## Release model (how the pipelines fit together)

The process is Gitflow-based. `Jenkinsfile.release` is the **release-cut** job — it does
not build artifacts, it manipulates branches and the `VERSION` file:

1. Checks out `develop` from `wso2-apim-app` via a **GitHub App** checkout
   (`gitHubAppCheckout` / credential id `github-app`) — no PATs are exposed.
2. Reads the `VERSION` file (must be `MAJOR.MINOR.PATCH[-SNAPSHOT]`), strips
   `-SNAPSHOT` to get `RELEASE_VERSION`, and computes `NEXT_DEV_VERSION` (either
   auto-bumping the minor via `AUTO_BUMP_MINOR`, or from `NEXT_DEV_VERSION_OVERRIDE`).
3. Guards that `release/<version>` does not already exist on the remote
   (via a `GitHubHelper` shared-library class).
4. Creates and pushes `release/<version>` with the de-SNAPSHOTted `VERSION`.
5. Bumps `develop` to `<next>-SNAPSHOT` and pushes it.

After the release branch exists, pushes to it are expected to auto-build release
candidates (that RC build lives in a separate pipeline, not in this file yet).
Commits made by the pipeline use `[skip ci]` to avoid retriggering builds.

## Conventions specific to this repo

- **Never introduce raw Git tokens/PATs.** Auth to the app repo goes through the
  GitHub App (`gitHubAppCheckout`, credential id `github-app`). Preserve this when
  adding stages that push or read from GitHub.
- `VERSION` is the single source of truth for versioning and is always validated
  against the `\d+\.\d+\.\d+` regex before use — keep this guard in any new version logic.
- Steps like `gitHubAppCheckout`, `GitHubHelper`, and `cleanWs` come from a Jenkins
  **shared library** configured on the controller, not from this repo. When editing,
  assume these globals exist; don't try to define them here.
- `.groovylintrc.json` extends `recommended-jenkinsfile` and raises
  `maxNestedBlockDepth` to 7 (pipeline/stages/steps/script nesting is deep by nature).
- New pipelines should follow the `Jenkinsfile.<purpose>` naming (e.g.
  `Jenkinsfile.release`) rather than a single monolithic `Jenkinsfile`.
