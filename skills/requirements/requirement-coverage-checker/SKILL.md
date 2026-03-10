---
name: requirement-coverage-checker
description: Checks whether requirements are fully covered by the existing implementation or test suite. Use when assessing requirement coverage, when the user asks if requirements are implemented, or when preparing for release.
license: Apache-2.0
metadata:
  category: "requirements"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Requirement Coverage Checker

## Purpose

Check if requirements are covered by implementation or tests.

## Workflow

1. **Input**: Requirements, implementation, and/or tests.
2. **Map**: Trace requirements to code and tests.
3. **Assess**: Identify covered, partially covered, uncovered.
4. **Report**: Coverage matrix with gaps.

## Output

- Coverage matrix (requirement → implementation/test)
- Uncovered or partially covered requirements
- Recommendations to close gaps
