---
name: abstract-invariant-generator
description: Generates abstract invariants over program state for use in verification proofs. Use when generating invariants for verification, when the user asks for loop or program invariants, or when strengthening specifications.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Abstract Invariant Generator

## Purpose

Generate abstract invariants over program state for verification.

## Workflow

1. **Input**: Program or specification.
2. **Analyze**: Identify loops, state variables, and relationships.
3. **Generate**: Propose invariants (loop, class, module).
4. **Validate**: Check that invariants hold (induction, model check).

## Output

- Invariant candidates with location
- Proof sketch or validation result
- Strengthening suggestions
