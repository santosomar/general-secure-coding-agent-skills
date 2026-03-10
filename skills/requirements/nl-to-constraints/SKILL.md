---
name: nl-to-constraints
description: Translates natural language requirements into formal constraints (OCL, Z, or similar) for use in verification. Use when formalizing requirements, when the user asks for formal constraints, or when preparing for model checking.
license: Apache-2.0
metadata:
  category: "requirements"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "ambiguity-detector, req-to-test, requirement-to-tlaplus-property-generator"
---

# NL to Constraints

## Purpose

Translate natural language requirements into formal constraints (OCL, Z, etc.).

## Workflow

1. **Input**: Natural language requirement(s).
2. **Parse**: Extract conditions, invariants, pre/postconditions.
3. **Translate**: Map to formal notation (OCL, Z, Alloy).
4. **Validate**: Ensure constraints are well-formed.

## Output

- Formal constraint(s)
- Mapping from NL to formal
- Tool compatibility notes
