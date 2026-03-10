---
name: requirement-to-tlaplus-property-generator
description: Converts software requirements into TLA+ temporal properties for model-based verification. Use when deriving TLA+ properties from requirements, when the user asks to formalize requirements, or when adding properties to a TLA+ model.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "tlaplus-spec-generator, nl-to-constraints"
---

# Requirement-to-TLA+ Property Generator

## Purpose

Convert requirements into TLA+ temporal properties for model checking.

## Workflow

1. **Input**: Requirements (natural language or structured).
2. **Parse**: Extract safety and liveness conditions.
3. **Translate**: Map to TLA+ expressions (invariants, temporal formulas).
4. **Output**: TLA+ property definitions.

## Output

- TLA+ property definitions
- Mapping from requirement to property
- Suggested model-checker config
