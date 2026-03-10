---
name: szz-bug-identifier
description: Applies the SZZ algorithm to VCS history to identify which commits introduced bugs by correlating bug-fix commits with earlier changes. Use when analyzing bug introduction history, when the user asks which commit introduced a bug, or when building defect prediction or technical debt metrics.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# SZZ Bug Identifier

## Purpose

Identify bug-introducing commits by tracing from bug-fix commits backward through the VCS using the SZZ (Śliwerski–Zimmermann–Zeller) algorithm.

## Workflow

1. **Identify fix commits**: Find commits that fix bugs (e.g., via issue links, "fix" in message).
2. **Blame and diff**: For each fixed line, use blame to find the last modifying commit.
3. **Filter**: Exclude commits that only added the fixed code (did not introduce the bug).
4. **Report**: List bug-introducing commits with affected files and fix linkage.

## Output

- Bug-introducing commits for each fix
- Affected files and lines
- Fix-to-introduction mapping
