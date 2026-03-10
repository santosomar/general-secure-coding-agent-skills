---
name: semantic-bug-detector
description: Detects logical/semantic bugs by understanding program intent — catches issues that syntax-only tools miss. Use when analyzing code for logic errors, off-by-one mistakes, or incorrect business logic, or when the user asks for semantic or logic-level bug detection.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Semantic Bug Detector

## Purpose

Find bugs that stem from incorrect logic, intent violations, or semantic misuse rather than syntax or type errors.

## Workflow

1. **Understand intent**: Infer expected behavior from names, comments, and patterns.
2. **Analyze semantics**: Check for logic errors (wrong condition, inverted logic, missing case), boundary issues, and invariant violations.
3. **Cross-reference**: Compare implementation against common patterns and domain expectations.
4. **Report**: List suspected semantic bugs with location and suggested correction.

## Output

- List of semantic issues with file:line
- Description of the suspected bug
- Suggested fix or invariant to add
