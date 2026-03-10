---
name: dependency-resolver
description: Analyzes and resolves dependency conflicts, cycles, or mismatches in project dependency graphs. Use when resolving dependency conflicts, when the user asks about dependency issues, or when upgrading dependencies.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Dependency Resolver

## Purpose

Analyze and resolve dependency conflicts, cycles, and mismatches.

## Workflow

1. **Build graph**: Parse manifest, lockfile, or build config.
2. **Detect**: Find conflicts (version conflicts), cycles, or mismatches.
3. **Resolve**: Propose resolution (upgrade, downgrade, exclude).
4. **Validate**: Ensure resolution is consistent.

## Output

- Conflict report
- Proposed resolution
- Updated manifest or lockfile diff
