---
name: semantic-szz-analyzer
description: Extends classic SZZ with semantic code understanding to reduce false positives and improve accuracy of bug-introducing commit identification. Use when SZZ results are noisy, when the user wants higher-confidence bug-introduction analysis, or when refining defect prediction inputs.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Semantic SZZ Analyzer

## Purpose

Improve SZZ by using semantic analysis (e.g., AST diff, refactoring detection) to filter false positives and better identify true bug-introducing changes.

## Workflow

1. **Run SZZ**: Get initial bug-introducing candidates from classic SZZ.
2. **Semantic filter**: Exclude refactorings, renames, and formatting-only changes using AST or diff analysis.
3. **Context check**: Verify the introducing change actually added the faulty logic, not just touched the line.
4. **Report**: Refined list of bug-introducing commits with higher confidence.

## Output

- Filtered bug-introducing commits
- Rationale for exclusions
- Confidence or score per candidate
