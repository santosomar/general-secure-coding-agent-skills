---
name: python-test-updater
description: Updates existing Python tests to stay aligned with code changes. Use when refactoring broke tests, when the user asks to update Python tests, or when keeping tests in sync with code.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Python Test Updater

## Purpose

Update Python tests after code changes.

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
