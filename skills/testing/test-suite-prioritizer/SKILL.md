---
name: test-suite-prioritizer
description: Reorders test execution to surface failures faster, prioritizing high-risk or recently changed code. Use when prioritizing tests, when the user asks to reorder tests, or when optimizing CI feedback time.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Test Suite Prioritizer

## Purpose

Reorder tests to surface failures faster.

## Workflow

1. **Input**: Test suite, code under test, optional history.
2. **Score**: Prioritize by risk, recent changes, coverage.
3. **Order**: Produce execution order.
4. **Output**: Prioritized test list or config.

## Output

- Prioritized test order
- Rationale and scores
- Optional: time-to-failure estimate
