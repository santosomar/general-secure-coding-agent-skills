---
name: coverage-enhancer
description: Analyzes coverage gaps and generates additional tests to increase line, branch, or path coverage. Use when improving coverage, when the user asks for coverage-enhancing tests, or when closing coverage gaps.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Coverage Enhancer

## Purpose

Analyze coverage gaps and generate tests to increase coverage.

## Workflow

1. **Input**: Code, existing tests, coverage report.
2. **Identify gaps**: Find uncovered lines, branches, or paths.
3. **Generate**: Create tests targeting gaps.
4. **Validate**: Run and verify coverage increases.
5. **Output**: New tests and updated coverage.

## Output

- New tests for coverage gaps
- Coverage before/after
- Remaining gaps if any
