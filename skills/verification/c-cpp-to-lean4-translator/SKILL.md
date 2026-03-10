---
name: c-cpp-to-lean4-translator
description: Translates C/C++ code into Lean 4 for theorem-proving-based verification. Use when verifying C/C++ in Lean 4, when the user asks to translate C/C++ to Lean, or when preparing for proof-based verification.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# C/C++ to Lean 4 Translator

## Purpose

Translate C/C++ code into Lean 4 for theorem-proving verification.

## Workflow

1. **Parse**: Build representation of C/C++ program.
2. **Translate**: Map to Lean 4 types and definitions.
3. **Annotate**: Add pre/postconditions, invariants.
4. **Output**: Lean 4 file with proof obligations.

## Output

- Lean 4 module
- Translation mapping
- Proof obligation summary
