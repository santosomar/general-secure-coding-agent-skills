---
name: module-level-code-translator
description: Translates entire modules between languages, preserving structure, naming conventions, and semantics. Use when porting a module to another language, when the user asks for module-level translation, or when migrating a subsystem.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Module-Level Code Translator

## Purpose

Translate entire modules between languages, preserving structure and semantics.

## Workflow

1. **Input**: Source module (file or package) and target language.
2. **Parse**: Build full module representation.
3. **Translate**: Map module structure, types, and behavior.
4. **Generate**: Output translated module with equivalent structure.
5. **Validate**: Check consistency and suggest integration steps.

## Output

- Translated module(s)
- Structure mapping
- Integration notes
