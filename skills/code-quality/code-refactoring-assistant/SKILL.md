---
name: code-refactoring-assistant
description: Suggests and applies targeted refactoring transformations (extract method, rename, simplify conditionals, etc.). Use when refactoring code, when the user asks for refactoring suggestions, or when improving code structure.
license: Apache-2.0
metadata:
  category: "code-quality"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "code-smell-detector, dead-code-eliminator"
---

# Code Refactoring Assistant

## Purpose

Suggest and apply refactorings: extract method, rename, simplify conditionals, replace magic numbers, etc.

## Workflow

1. **Identify opportunity**: Detect smell or user-selected region.
2. **Suggest refactor**: Propose transformation (extract method, rename, etc.) with before/after.
3. **Apply**: Generate the refactored code.
4. **Verify**: Ensure behavior is preserved (tests, if available).

## Output

- Refactoring suggestion with rationale
- Before/after code
- Optional: test run to confirm behavior
