---
name: multi-version-behavior-comparator
description: Compares the runtime behavior of multiple versions of the same code to identify regressions or behavioral drift. Use when comparing two implementations, when the user asks if behavior changed, or when validating a refactor or port.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Multi-Version Behavior Comparator

## Purpose

Compare runtime behavior of multiple code versions to detect regressions or drift.

## Workflow

1. **Input**: Two or more versions of the same code (or diff).
2. **Generate inputs**: Create test inputs (random, boundary, or from spec).
3. **Execute**: Run both versions on same inputs.
4. **Compare**: Diff outputs, side effects, performance.
5. **Report**: Differences with input that triggered them.

## Output

- Behavioral differences (if any)
- Inputs that produce different output
- Suggested regression test
