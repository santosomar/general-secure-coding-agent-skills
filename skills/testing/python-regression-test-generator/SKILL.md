---
name: python-regression-test-generator
description: Generates regression test suites specifically for Python codebases. Use when generating Python regression tests, when the user asks for Python regression tests, or when protecting Python code from regressions.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Python Regression Test Generator

## Purpose

Generate regression test suites for Python codebases.

## Workflow

1. **Input**: Python code (module, class, or function).
2. **Analyze**: Extract behavior, invariants, public API.
3. **Generate**: Create pytest/unittest tests for regression.
4. **Output**: Python test code.

## Output

- Python regression test code (pytest/unittest)
- Coverage of public API
- Edge case handling
