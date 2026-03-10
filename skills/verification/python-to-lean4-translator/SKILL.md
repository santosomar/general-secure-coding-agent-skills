---
name: python-to-lean4-translator
description: Translates Python code into Lean 4 for use in formal proof environments. Use when verifying Python in Lean 4, when the user asks to translate Python to Lean, or when preparing Python for theorem proving.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Python to Lean 4 Translator

## Purpose

Translate Python code into Lean 4 for formal proof.

## Workflow

1. **Parse**: Build representation of Python program.
2. **Translate**: Map to Lean 4 definitions and proofs.
3. **Annotate**: Add pre/postconditions, invariants.
4. **Output**: Lean 4 file with proof obligations.

## Output

- Lean 4 module
- Translation mapping
- Proof obligation summary
