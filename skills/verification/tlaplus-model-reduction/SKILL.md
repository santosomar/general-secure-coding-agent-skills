---
name: tlaplus-model-reduction
description: Simplifies or abstracts TLA+ models to make them tractable for the TLC model checker. Use when a TLA+ model is too large for TLC, when the user asks to reduce model size, or when state explosion is a problem.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# TLA+ Model Reduction

## Purpose

Simplify or abstract TLA+ models for tractable model checking.

## Workflow

1. **Input**: TLA+ module that times out or exhausts memory.
2. **Analyze**: Identify state-space contributors (symmetry, constants, abstraction).
3. **Reduce**: Apply symmetry sets, constant reduction, or abstraction.
4. **Validate**: Ensure reduced model preserves properties of interest.

## Output

- Reduced TLA+ module or config
- Explanation of reduction
- Property preservation argument
