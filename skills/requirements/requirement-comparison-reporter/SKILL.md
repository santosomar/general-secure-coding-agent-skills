---
name: requirement-comparison-reporter
description: Compares two versions of a requirements document and reports additions, removals, and changes. Use when comparing requirement versions, when the user asks what changed in requirements, or when tracking requirement evolution.
license: Apache-2.0
metadata:
  category: "requirements"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Requirement Comparison Reporter

## Purpose

Compare two requirement versions and report additions, removals, and changes.

## Workflow

1. **Input**: Two requirement documents (or versions).
2. **Parse**: Extract requirements from each.
3. **Diff**: Match and compare (added, removed, modified).
4. **Report**: Structured diff with impact notes.

## Output

- Added requirements
- Removed requirements
- Modified requirements with before/after
- Impact summary
