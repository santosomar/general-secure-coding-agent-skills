---
name: behavior-preservation-checker
description: Verifies that a refactoring or transformation has not changed the observable behavior of the code. Use when refactoring, when the user asks to verify behavior preservation, or when validating refactoring safety.
license: Apache-2.0
metadata:
  category: "code-quality"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Behavior Preservation Checker

## Purpose

Verify that refactoring or transformation preserves observable behavior.

## Workflow

1. **Input**: Original and transformed code (or diff).
2. **Compare**: Analyze I/O, side effects, error paths, edge cases.
3. **Report**: Flag any behavioral differences.
4. **Optional**: Run tests on both versions to confirm.

## Output

- Pass/fail for behavior preservation
- List of potential differences if any
- Suggested test to add
