---
name: build-ci-migration-assistant
description: Guides and automates migration between CI/CD platforms (e.g., Jenkins → GitHub Actions, CircleCI → GitLab CI). Use when migrating CI/CD platforms, when the user asks to convert a pipeline, or when consolidating pipelines across tools.
license: Apache-2.0
metadata:
  category: "devops"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "ci-pipeline-synthesizer"
---

# Build/CI Migration Assistant

## Purpose

Migrate CI/CD configs from one platform to another while preserving behavior (jobs, triggers, caching, secrets).

## Workflow

1. **Input**: Existing pipeline config (Jenkinsfile, CircleCI config, etc.).
2. **Parse**: Extract jobs, steps, triggers, artifacts, and secrets usage.
3. **Map**: Translate to target platform syntax and semantics.
4. **Generate**: Output equivalent config for target platform.
5. **Diff**: Report differences and manual steps (e.g., secret setup).

## Output

- Target platform config
- Migration notes and manual steps
- Known limitations or behavior differences
