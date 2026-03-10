---
name: cpp-to-dafny-translator
description: Translates C++ code into Dafny for specification and automated verification. Use when verifying C++ in Dafny, when the user asks to translate C++ to Dafny, or when preparing for Dafny verification.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# C++ to Dafny Translator

## Purpose

Translate C++ code into Dafny for specification and verification.

## Workflow

1. **Parse**: Build representation of C++ program.
2. **Translate**: Map to Dafny methods and types.
3. **Annotate**: Add requires/ensures, invariants.
4. **Output**: Dafny file; run verifier.

## Output

- Dafny module
- Verification result
- Unresolved proof obligations if any
