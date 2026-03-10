---
name: pseudocode-extractor
description: Extracts language-agnostic pseudocode representations from source code to communicate logic clearly. Use when explaining logic without language specifics, when the user asks for pseudocode, or when creating algorithm documentation.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Pseudocode Extractor

## Purpose

Extract language-agnostic pseudocode from source code.

## Workflow

1. **Input**: Source code (function or algorithm).
2. **Parse**: Extract control flow, data flow, key operations.
3. **Abstract**: Remove language-specific syntax, use generic constructs.
4. **Output**: Pseudocode (e.g., structured English, algorithmic notation).

## Output

- Pseudocode representation
- Mapping from pseudocode to source lines
- Assumptions about data structures
