---
name: counterexample-to-test-generator
description: Converts counterexamples produced by model checkers into executable test cases to reproduce and guard against failures. Use when converting a model checker counterexample to a test, when the user has a counterexample to turn into a test, or when adding regression tests from verification.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Counterexample-to-Test Generator

## Purpose

Convert model checker counterexamples into executable regression tests.

## Workflow

1. **Input**: Counterexample trace (states, transitions).
2. **Map**: Translate trace to program inputs and sequence.
3. **Generate**: Produce test that exercises the violating path.
4. **Assert**: Add assertion for the violated property.

## Output

- Executable test code
- Mapping from trace to test
- Expected outcome (fail before fix, pass after)
