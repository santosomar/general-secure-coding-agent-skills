---
name: smart-mutation-operator-generator
description: Generates domain-aware mutation operators tailored to the codebase's language, patterns, and known fault types. Use when creating custom mutation operators, when the user asks for domain-specific mutations, or when improving mutation testing effectiveness.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Smart Mutation Operator Generator

## Purpose

Generate domain-aware mutation operators for a codebase.

## Workflow

1. **Input**: Codebase, language, patterns.
2. **Analyze**: Identify language constructs, idioms, fault types.
3. **Generate**: Propose mutation operators (e.g., boundary, operator, logic).
4. **Output**: Custom mutation operators or config.

## Output

- Mutation operators (or config)
- Rationale per operator
- Integration notes for mutation tool
