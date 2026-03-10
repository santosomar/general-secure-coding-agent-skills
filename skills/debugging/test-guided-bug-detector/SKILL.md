---
name: test-guided-bug-detector
description: Uses failing test results as signals to guide bug search and narrow down candidate fault locations. Use when tests fail and the user wants to locate the fault, when prioritizing which code to inspect, or when the user asks for test-guided fault localization.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Test-Guided Bug Detector

## Purpose

Use failing (and passing) test execution information to guide fault localization and reduce the search space.

## Workflow

1. **Run tests**: Execute the test suite and collect pass/fail and coverage.
2. **Correlate**: Apply spectrum-based or mutation-based fault localization (e.g., Ochiai, DStar) to rank code by suspiciousness.
3. **Narrow**: Focus on code executed by failing tests but not (or less) by passing tests.
4. **Report**: Ranked list of suspicious statements or functions.

## Output

- Ranked fault candidates with suspiciousness scores
- Which tests inform each candidate
- Suggested inspection order
