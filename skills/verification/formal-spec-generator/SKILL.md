---
name: formal-spec-generator
description: Generates formal specifications in multiple target formats from informal descriptions. Use when creating formal specs in various formats, when the user describes a system to formalize, or when targeting multiple verification tools. Cross-ref: verification-tlaplus-spec-generator for TLA+-specific generation.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "tlaplus-spec-generator, dafny-spec-generator, lean4-spec-generator"
---

# Formal Spec Generator

## Purpose

Generate formal specs in multiple formats (TLA+, Alloy, Z, etc.) from informal descriptions.

## Workflow

1. **Input**: Informal or semi-formal system description.
2. **Parse**: Extract state, operations, invariants.
3. **Generate**: Produce spec in target format(s).
4. **Validate**: Ensure specs are well-formed.

## Output

- Formal spec(s) in target format(s)
- Mapping from description to spec
- Tool-specific notes
