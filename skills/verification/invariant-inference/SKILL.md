---
name: invariant-inference
description: Automatically infers loop invariants and program invariants to support formal proofs. Use when inferring invariants for verification, when the user asks for invariant inference, or when automating proof discovery.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Invariant Inference

## Purpose

Automatically infer loop and program invariants for formal proofs.

## Workflow

1. **Input**: Program with loops or stateful components.
2. **Infer**: Use template-based or constraint-based inference.
3. **Validate**: Verify inferred invariants hold.
4. **Output**: Invariants with proof obligations.

## Output

- Inferred invariants
- Validation result
- Proof obligations if any
