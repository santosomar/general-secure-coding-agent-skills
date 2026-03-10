---
name: program-to-tlaplus-spec-generator
description: Translates an existing program into a TLA+ formal specification for model checking. Use when creating a TLA+ model from code, when the user asks to formalize a program, or when preparing for model checking.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Program-to-TLA+ Spec Generator

## Purpose

Translate a program into a TLA+ specification for model checking.

## Workflow

1. **Parse program**: Extract state, transitions, and invariants.
2. **Abstract**: Map to TLA+ state machine (variables, Init, Next).
3. **Generate**: Output TLA+ module with properties.
4. **Validate**: Ensure spec is well-formed and checkable.

## Output

- TLA+ module file
- Mapping from program to spec
- Suggested properties to verify
