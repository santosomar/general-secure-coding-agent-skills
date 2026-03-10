---
name: design-pattern-suggestor
description: Identifies opportunities to apply well-known design patterns (Factory, Observer, Strategy, etc.) to improve structure. Use when improving structure, when the user asks for pattern suggestions, or when reducing coupling.
license: Apache-2.0
metadata:
  category: "code-quality"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "code-refactoring-assistant, code-review-assistant"
---

# Design Pattern Suggestor

## Purpose

Identify opportunities to apply design patterns (Factory, Observer, Strategy, etc.) to improve structure.

## Workflow

1. **Analyze**: Detect code that could benefit from a pattern (e.g., many conditionals → Strategy).
2. **Suggest**: Propose pattern with rationale and example transformation.
3. **Apply**: Generate refactored code using the pattern.

## Output

- Pattern suggestion with location
- Before/after code
- Rationale for the pattern
