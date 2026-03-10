---
name: code-translation
description: Translates code between programming languages (e.g., Java ↔ Python, C++ ↔ Rust). Use when porting code to another language, when the user asks to translate code, or when migrating implementations.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Code Translation

## Purpose

Translate code between programming languages while preserving behavior.

## Workflow

1. **Input**: Source code and target language.
2. **Parse**: Build representation of source.
3. **Map**: Translate constructs to target language idioms.
4. **Generate**: Output translated code.
5. **Validate**: Check syntax and suggest tests.

## Output

- Translated code in target language
- Mapping notes and idioms used
- Limitations or manual review suggestions
