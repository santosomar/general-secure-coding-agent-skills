---
name: change-log-generator
description: Automatically generates changelogs following formats like Keep a Changelog or Conventional Commits from VCS history. Use when maintaining a changelog, when the user asks for a changelog, or when automating release documentation.
license: Apache-2.0
metadata:
  category: "devops"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Changelog Generator

## Purpose

Generate changelogs from VCS history using conventions like Keep a Changelog or Conventional Commits.

## Workflow

1. **Parse history**: Read commits since last tag or version.
2. **Classify**: Map commits to types (feat, fix, chore, etc.) via convention or heuristics.
3. **Group**: Organize by version and category.
4. **Output**: Produce changelog in target format (Markdown, etc.).

## Output

- Changelog file (CHANGELOG.md style)
- Optional: per-version or per-release sections
- Unreleased section for current work
