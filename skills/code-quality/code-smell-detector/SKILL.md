---
name: code-smell-detector
description: Identifies code smells such as long methods, god classes, duplicate code, and other anti-patterns. Use when reviewing code quality, when the user asks for smell detection, or when prioritizing refactoring work.
license: Apache-2.0
metadata:
  category: "code-quality"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "code-refactoring-assistant, code-review-assistant"
---

# Code Smell Detector

## Purpose

Detect code smells: long methods, god classes, duplicate code, feature envy, and other anti-patterns.

## Workflow

1. **Analyze codebase**: Parse and build metrics (LOC, complexity, coupling, cohesion).
2. **Apply rules**: Check for long methods, large classes, duplicated blocks, inappropriate intimacy.
3. **Rank**: Score by severity and refactoring impact.
4. **Report**: List smells with location and suggested refactor.

## Output

- List of code smells (file, symbol, type)
- Metric values that triggered each smell
- Refactoring suggestion
