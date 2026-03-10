---
name: ci-pipeline-synthesizer
description: Generates CI pipeline configs by analyzing a repo's structure, language, and build needs — GitHub Actions, GitLab CI, or other platforms. Use when bootstrapping CI for a new repo, when porting from one CI to another, when the user asks for a pipeline that builds and tests their project, or when wiring in security gates.
license: Apache-2.0
metadata:
  category: "devops"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "cd-pipeline-generator, build-ci-migration-assistant, containerization-assistant"
---

# CI Pipeline Synthesizer

Generate a CI config that **actually runs**, by reading the repo rather than asking the user twenty questions.

## Step 1 — Detect the build

Read the repo, not the user's description. Signals:

| File present                          | Ecosystem        | Build command                    | Test command                  |
| ------------------------------------- | ---------------- | -------------------------------- | ----------------------------- |
| `package.json`                        | Node/JS          | `npm ci` then `npm run build`    | `npm test`                    |
| `pyproject.toml` / `setup.py`         | Python           | `pip install -e .` or `uv sync`  | `pytest` / `python -m pytest` |
| `Cargo.toml`                          | Rust             | `cargo build --release`          | `cargo test`                  |
| `go.mod`                              | Go               | `go build ./...`                 | `go test ./...`               |
| `pom.xml`                             | Java/Maven       | `mvn -B package`                 | `mvn -B test`                 |
| `build.gradle` / `build.gradle.kts`   | Java/Gradle      | `./gradlew build`                | `./gradlew test`              |
| `Makefile` (and nothing else)         | Make             | `make`                           | `make test` (if target exists)|
| `Dockerfile` (and no build system)    | Container-only   | `docker build .`                 | — (look for test stage)       |

Read the actual file. `package.json` might not have a `build` script. `Makefile` might not have a `test` target. Adapt.

## Step 2 — Choose the platform target

| Signal                           | Emit                     |
| -------------------------------- | ------------------------ |
| `.github/` dir exists            | GitHub Actions           |
| `.gitlab-ci.yml` referenced in docs, or remote is `gitlab.*` | GitLab CI |
| User specifies                   | Whatever they said       |
| No signal                        | GitHub Actions (default — largest install base) |

## Step 3 — Assemble the pipeline

Every CI pipeline has the same skeleton. Fill the slots:

```
[trigger]          — push to main + PRs. Nothing fancier unless asked.
  [checkout]       — always
  [setup runtime]  — language-specific setup action, version pinned to whatever the repo uses
  [cache deps]     — keyed on lockfile hash
  [install deps]   — from Step 1
  [lint]           — only if a lint config exists (.eslintrc, ruff.toml, .golangci.yml)
  [build]          — from Step 1 (skip for interpreted langs with no build step)
  [test]           — from Step 1
  [security]       — optional, see below
```

**Pin versions.** `actions/checkout@v4` not `@main`. `python-version: '3.11'` not `'3.x'`. Floating versions are reproducibility bugs waiting to happen.

## Security gates

Ask or infer whether to include (opt-in — they add runtime and noise):

- **Dependency audit** (`npm audit`, `pip-audit`, `cargo audit`) — cheap, include by default as a non-blocking step
- **SAST** — only if the repo already has a config for one (`.semgrep.yml`, CodeQL workflow); otherwise suggest but don't inject
- **Secret scanning** — usually a platform feature, not a pipeline step — mention, don't add

## Worked example

**Repo contents:** `pyproject.toml` (uses `hatchling`), `uv.lock`, `tests/`, `.github/` exists but is empty.

**Output (`.github/workflows/ci.yml`):**

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - uses: astral-sh/setup-uv@v3
      - run: uv sync --frozen
      - run: uv run pytest
```

Six steps. No matrix (not asked for). No coverage upload (not asked for). `--frozen` because `uv.lock` exists — fail if lockfile is stale. Python `3.12` because `pyproject.toml` said `requires-python = ">=3.12"`.

## Edge cases

- **Monorepo with multiple languages:** Emit one job per language, with `paths:` filters so they only run when their subtree changes. Don't run the Python tests because someone touched the Go service.
- **No tests exist:** Still emit a build-and-lint pipeline. Add a commented-out `# TODO: add test step when tests exist` — don't silently skip it.
- **Build needs a service (DB, Redis):** Add a `services:` block. Detect from test fixtures or docker-compose files.
- **The repo already has a CI config:** Do NOT overwrite. Read it, report what's missing, offer a diff.

## Do not

- **Do not** emit a 200-line pipeline for a 10-file repo. Match complexity to the project.
- **Do not** add matrix builds (multiple OS/versions) unless the project is a library that claims compatibility with a range. Applications run on one thing.
- **Do not** add steps for tools the repo doesn't use. No `eslint` step if there's no `.eslintrc`.
- **Do not** cargo-cult `continue-on-error: true`. If a step can fail, decide: is it blocking or not? If not blocking, why is it there?
- **Do not** inline secrets. `${{ secrets.X }}` only, and list which secrets need to be configured.

## Output format

```
## Detected
Language: <lang>  Build: <cmd>  Test: <cmd>  Platform: <ci>

## Pipeline
<file path>
<code block with full pipeline>

## Secrets to configure
- <name>: <what it's for>

## Optional next steps
- <matrix build / coverage / security gate — only what's relevant>
```
