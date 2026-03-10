---
name: java-test-updater
description: Updates existing Java tests to remain valid after code changes (renamed methods, changed signatures, etc.). Use when refactoring broke tests, when the user asks to update Java tests, or when keeping tests in sync with code.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Java Test Updater

## Purpose

Update Java tests after code changes (renames, signature changes).

## Workflow

1. **Input**: Code diff, existing tests.
2. **Analyze**: Find tests affected by changes.
3. **Update**: Fix imports, calls, assertions.
4. **Validate**: Ensure tests pass.
5. **Output**: Updated test code.

## Output

- Updated test code
- List of changes made
- Test run result
