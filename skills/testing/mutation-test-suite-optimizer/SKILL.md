---
name: mutation-test-suite-optimizer
description: Optimizes test suites to maximize mutation score with minimum redundant tests. Use when optimizing a test suite for mutation testing, when the user asks to reduce tests while keeping mutation score, or when improving test efficiency.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Mutation Test Suite Optimizer

## Purpose

Optimize test suite to maximize mutation score with minimum redundant tests.

## Workflow

1. **Input**: Test suite, code under test.
2. **Run mutation**: Generate mutants, run tests.
3. **Analyze**: Identify redundant tests (same mutant kills).
4. **Optimize**: Select minimal subset that preserves mutation score.
5. **Output**: Optimized test suite.

## Output

- Optimized test list
- Mutation score before/after
- Removed redundant tests
