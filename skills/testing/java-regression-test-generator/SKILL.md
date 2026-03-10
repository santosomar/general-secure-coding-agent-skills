---
name: java-regression-test-generator
description: Generates regression test suites specifically for Java codebases. Use when generating Java regression tests, when the user asks for Java regression tests, or when protecting Java code from regressions.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Java Regression Test Generator

## Purpose

Generate regression test suites for Java codebases.

## Workflow

1. **Input**: Java code (class, package, or module).
2. **Analyze**: Extract behavior, invariants, public API.
3. **Generate**: Create JUnit/TestNG tests for regression.
4. **Output**: Java test code.

## Output

- Java regression test code (JUnit/TestNG)
- Coverage of public API
- Edge case handling
