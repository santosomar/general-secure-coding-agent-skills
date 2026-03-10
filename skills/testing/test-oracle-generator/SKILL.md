---
name: test-oracle-generator
description: Generates test oracles (assertions, invariant checks) that define correct output for a given input. Use when creating test oracles, when the user asks for assertions, or when defining expected behavior for tests.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "unit-test-generator, metamorphic-test-generator"
---

# Test Oracle Generator

## Purpose

Generate test oracles (assertions, invariant checks) for given inputs.

## Workflow

1. **Input**: Code under test, or specification.
2. **Analyze**: Extract expected behavior (output, invariants, postconditions).
3. **Generate**: Produce assertions or oracle functions.
4. **Output**: Oracle code (assertions, property checks).

## Output

- Assertions or oracle functions
- Mapping from input to expected output
- Invariant checks if applicable
