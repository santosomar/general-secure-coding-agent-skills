---
name: verified-pseudocode-extractor
description: Extracts algorithmically verified pseudocode from formal specifications or verified implementations. Use when extracting pseudocode from verified code, when the user asks for verified pseudocode, or when documenting verified algorithms.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Verified Pseudocode Extractor

## Purpose

Extract verified pseudocode from formal specs or verified implementations.

## Workflow

1. **Input**: Formal spec (TLA+, Dafny) or verified implementation.
2. **Extract**: Derive high-level pseudocode preserving verified properties.
3. **Annotate**: Include pre/postconditions and invariants.
4. **Output**: Human-readable pseudocode with verification annotations.

## Output

- Pseudocode with annotations
- Mapping to original spec/implementation
- Property summary
