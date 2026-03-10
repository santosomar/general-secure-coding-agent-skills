---
name: python-to-dafny-translator
description: Translates Python code into Dafny with accompanying pre/postconditions and invariants. Use when verifying Python in Dafny, when the user asks to translate Python to Dafny, or when preparing Python for formal verification.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Python to Dafny Translator

## Purpose

Translate Python code into Dafny with pre/postconditions and invariants.

## Workflow

1. **Parse**: Build representation of Python program.
2. **Translate**: Map to Dafny (methods, datatypes).
3. **Annotate**: Add requires/ensures, loop invariants.
4. **Output**: Dafny file; run verifier.

## Output

- Dafny module
- Verification result
- Annotation summary
