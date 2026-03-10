---
name: smv-model-extractor
description: Extracts SMV (Symbolic Model Verifier) models from source code for NuSMV/nuXmv-based verification. Use when creating SMV models from code, when the user asks for NuSMV/nuXmv models, or when preparing for symbolic model checking.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# SMV Model Extractor

## Purpose

Extract SMV models from source code for NuSMV/nuXmv verification.

## Workflow

1. **Parse**: Build model of program state and transitions.
2. **Translate**: Map to SMV module (variables, INIT, TRANS).
3. **Generate**: Output SMV file with properties.
4. **Validate**: Ensure SMV is well-formed.

## Output

- SMV module file
- Property specifications
- NuSMV/nuXmv run instructions
