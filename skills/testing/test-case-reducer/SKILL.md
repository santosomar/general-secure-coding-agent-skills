---
name: test-case-reducer
description: Minimizes failing test cases to their smallest reproducing form for easier debugging. Use when minimizing a failing test, when the user has a large failing test to reduce, or when creating minimal reproducers.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Test Case Reducer

## Purpose

Minimize failing test cases to smallest reproducing form.

## Workflow

1. **Input**: Failing test (or test + failing input).
2. **Reduce**: Apply delta debugging or similar to shrink inputs.
3. **Verify**: Ensure reduced test still fails.
4. **Output**: Minimal failing test.

## Output

- Minimal failing test
- Reduction steps
- Verification that it still fails
