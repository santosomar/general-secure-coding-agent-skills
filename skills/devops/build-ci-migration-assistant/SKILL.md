---
name: build-ci-migration-assistant
description: Assists migrating a build or CI pipeline from one system to another — Jenkins to GitHub Actions, Travis to GitLab CI, Makefile to Bazel — preserving semantics and surfacing untranslatable constructs. Use when switching CI providers, when modernizing a legacy build, or when the user pastes a Jenkinsfile and asks for the GitHub Actions equivalent.
license: Apache-2.0
metadata:
  category: "devops"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "ci-pipeline-synthesizer, cd-pipeline-generator"
---

# Build/CI Migration Assistant

Migrating CI is a **translation problem with losses.** Every CI system has the same core (stages, steps, artifacts, conditions) and each has idioms that don't translate cleanly. The job is to translate what maps, and flag what doesn't.

## Concept map — the common core

All CI systems implement these. Translate via the concept, not the syntax:

| Concept               | Jenkins                       | GitHub Actions                | GitLab CI                  | CircleCI                 |
| --------------------- | ----------------------------- | ----------------------------- | -------------------------- | ------------------------ |
| Pipeline file         | `Jenkinsfile`                 | `.github/workflows/*.yml`     | `.gitlab-ci.yml`           | `.circleci/config.yml`   |
| Unit of parallelism   | `stage` / `parallel`          | `job`                         | `job` (in `stage`)         | `job`                    |
| Sequential grouping   | `stage`                       | `needs:`                      | `stage:`                   | `requires:`              |
| Reusable fragment     | Shared library, `load()`      | Composite action / reusable workflow | `extends:` / `include:` | Orb / command |
| Run on                | `agent { label }`             | `runs-on:`                    | `tags:`                    | `executor:`              |
| Conditional           | `when { ... }`                | `if:`                         | `rules:`                   | `when:` (filter)         |
| Env var               | `environment { }`             | `env:`                        | `variables:`               | `environment:`           |
| Secret                | `credentials()`               | `${{ secrets.X }}`            | CI/CD variables (masked)   | Context                  |
| Artifact hand-off     | `stash`/`unstash`, `archiveArtifacts` | `upload-artifact` / `download-artifact` | `artifacts:` | `persist_to_workspace`   |
| Cache                 | Plugin-dependent              | `actions/cache`               | `cache:`                   | `save_cache`/`restore_cache` |
| Trigger               | `triggers { }` / webhooks     | `on:`                         | `rules:` / `workflow:`     | `triggers:` (filters)    |

## Step 1 — Extract the dependency graph

Before writing any target syntax, draw the DAG: what runs, what depends on what, what runs in parallel. This is the invariant. Syntax changes; the graph doesn't.

```
            ┌─ lint ──┐
checkout ──▶├─ test ──┤──▶ build ──▶ deploy-staging ──▶ deploy-prod
            └─ audit ─┘
```

## Step 2 — Translate node-by-node through the concept map

For each node in the graph, map the source construct → concept → target construct. Most steps are just shell commands inside a wrapper — those translate 1:1.

## Step 3 — Flag the untranslatables

Things that **do not map cleanly** — inventory and escalate:

| Source feature                                  | Why it doesn't translate                          | What to do                                    |
| ----------------------------------------------- | ------------------------------------------------- | --------------------------------------------- |
| Jenkins `input` step (human prompt mid-pipeline)| GHA/GitLab don't pause for input                  | Split into two workflows, use environment approvals |
| Jenkins Groovy shared libraries                 | Arbitrary code, not declarative                   | Rewrite as composite actions / scripts; may need refactoring |
| Self-hosted agent with local state              | Cloud runners are ephemeral                       | Move state to a cache or artifact             |
| Implicit workspace sharing between stages       | GHA jobs get fresh filesystems                    | Explicit `upload-artifact`/`download-artifact` |
| Polling triggers (`pollSCM`)                    | Push-based systems don't poll                     | Use webhook triggers (`on: push`) — usually what you wanted anyway |
| Jenkins `post { always/failure/success }`       | GHA has `if: always()` per step, not per stage block | Duplicate the post-step on each job with `if:` guards |

## Worked example

**Source (Jenkinsfile):**

```groovy
pipeline {
  agent any
  stages {
    stage('Test') {
      steps { sh 'npm ci && npm test' }
    }
    stage('Build') {
      when { branch 'main' }
      steps {
        sh 'npm run build'
        archiveArtifacts artifacts: 'dist/**'
      }
    }
  }
  post {
    failure { slackSend channel: '#ci', message: "Build failed: ${env.BUILD_URL}" }
  }
}
```

**DAG:** `Test → Build (main only)`, plus a failure notification.

**Target (GitHub Actions):**

```yaml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm test

  build:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  notify-failure:
    needs: [test, build]
    if: failure()
    runs-on: ubuntu-latest
    steps:
      - run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -d '{"text":"Build failed: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"}'
```

**Flagged:** The `post { failure }` became a separate job — Jenkins runs `post` in the same agent with the workspace intact; here it's a fresh runner. If the Slack message needed build artifacts, they'd have to be uploaded first. **Also:** `npm ci` runs twice now (once per job — fresh filesystems). Could consolidate test+build into one job, or cache `node_modules`.

## Do not

- **Do not** translate line-by-line. Translate through the concept map. A `stage` is not always a `job`.
- **Do not** silently drop untranslatable features. Every Groovy shared library call you skip is a behavior the new pipeline is missing.
- **Do not** assume the old pipeline was correct. Migration is a good time to delete steps nobody remembers the purpose of — but *ask* first.
- **Do not** migrate and switch over in one step. Run both pipelines in parallel for a few builds; diff the results.

## Output format

```
## Dependency graph (extracted)
<ascii dag>

## Translated pipeline
<target file path>
<code block>

## Untranslated / degraded
| Source construct | Issue | Proposed workaround |
| ... | ... | ... |

## Verification plan
- Run both pipelines on the same commit; diff: <what to compare>
```
