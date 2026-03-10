---
name: ci-pipeline-synthesizer
description: Generates complete CI pipeline configurations (e.g., GitHub Actions, GitLab CI, Jenkins) tailored to the project's language and structure. Use when setting up CI for a new project, when the user asks for a CI pipeline, or when migrating to a new CI platform.
license: Apache-2.0
metadata:
  category: "devops"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "cd-pipeline-generator, containerization-assistant"
---

# CI Pipeline Synthesizer

## Purpose

Produce CI pipeline configs (GitHub Actions, GitLab CI, Jenkinsfile) that match the project's language, build system, and structure.

## Workflow

1. **Analyze project**: Detect language, package manager, build tool, test framework.
2. **Select template**: Choose pipeline structure (lint, build, test, artifact) for the stack.
3. **Generate config**: Output YAML or Jenkinsfile with jobs, caching, and matrix if applicable.
4. **Validate**: Ensure syntax is valid and steps are executable.

## Output

- Complete CI config file
- Brief explanation of each job
- Caching and optimization recommendations
