---
name: specification-to-temporal-logic-generator
description: Translates specifications into temporal logic formulas (LTL/CTL) for use with model checkers. Use when generating LTL/CTL from specs, when the user asks for temporal logic formulas, or when preparing for NuSMV/SPIN.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Specification-to-Temporal-Logic Generator

## Purpose

Translate specifications into LTL/CTL formulas for model checkers.

## Workflow

1. **Input**: Specification (informal or semi-formal).
2. **Extract**: Safety and liveness conditions.
3. **Translate**: Produce LTL or CTL formulas.
4. **Validate**: Ensure formulas are well-formed.

## Output

- LTL/CTL formulas
- Mapping from spec to formula
- Model-checker compatibility notes
