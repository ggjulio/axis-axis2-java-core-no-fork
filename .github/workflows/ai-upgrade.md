---
on:
  slash_command: ai-upgrade
  workflow_dispatch:
  roles: [admin]
  reaction: rocket

engine:
  id: copilot
  model: claude-opus-4.6

permissions:
  actions: read
  contents: read
  issues: read
  pull-requests: read

tools:
  github:
    toolsets: [issues, pull_requests, actions]

jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/cache@v5
        with:
          path: ~/.m2/repository
          key: copilot-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            copilot-maven-
            maven-
      - uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
      - run: sudo apt-get update && sudo apt-get install -y ant
      - run: mvn -B -q dependency:go-offline -DexcludeReactor=false || true

env:
  MAVEN_OPTS: -Dmaven.wagon.httpconnectionManager.ttlSeconds=25 -Dmaven.wagon.http.retryHandler.count=3

safe-outputs:
  push-to-pull-request-branch:
    target: triggering
    commit-title-suffix: " [ai-upgrade]"
    labels: [ai/generated, ai/claude-opus-4.6]         # require all labels
    if-no-changes: "error"
  add-comment:
---

# AI-Assisted Upgrade

Fix CI build failures on dependency update pull requests.

## Trigger

This workflow is triggered by commenting `/ai-upgrade` on a pull request that has a failing CI build.

## Context

The triggering comment/context is: "${{ needs.activation.outputs.text }}"
The PR number is: ${{ github.event.issue.number }}

## Instructions

1. **Identify the pull request** from the trigger context.
2. **Read the PR description** carefully — dependency update PRs (from Dependabot or Renovate) include release notes, changelogs, and breaking change information. This is critical for major version upgrades.
3. **Find the latest failed CI run** for this PR using the Actions API. Look for failed runs of the "Continuous Integration" workflow on the PR's head SHA.
4. **Download and analyze the failed job logs** to identify the root cause of the build failure.
5. **Fix the code** to make the build pass. This project uses:
   - **Maven** (`mvn`) as the primary build tool.
   - **Java 17** as the base version (also tested on Java 21 and 25).
   - **Ant** for some sub-modules (e.g., `modules/samples/build.xml`, kernel test resources).
7. Common fixes for dependency upgrades include:
   - Updating renamed/moved classes or packages in import statements.
   - Adjusting API calls for changed method signatures.
   - Updating configuration files for changed property names.
   - Fixing version compatibility issues in `pom.xml`.
8. **Verify the build** by running: `mvn -B -e -Papache-release -Dgpg.skip=true -Dmaven.compiler.release=17 verify`
9. If the build passes, commit the fix and push it to the PR branch.

## Style

- Make minimal, targeted changes — only fix what's needed for the build to pass.
- Preserve existing code style and conventions.
- Add comments only if the fix is non-obvious.
