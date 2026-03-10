---
name: test-case-documentation
description: Generates structured documentation for test cases including purpose, inputs, expected outputs, and rationale. Use when documenting tests, when the user asks for test documentation, or when creating test specs.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Test Case Documentation

## Purpose

Generate structured documentation for test cases.

## Workflow

1. **Input**: Test code or test suite.
2. **Extract**: Derive purpose, inputs, expected outputs from code.
3. **Document**: Produce structured docs (purpose, inputs, outputs, rationale).
4. **Output**: Documentation (Markdown, docstring, etc.).

## Output

- Structured test documentation
- Per-test: purpose, inputs, expected outputs, rationale
- Format suitable for review or spec
