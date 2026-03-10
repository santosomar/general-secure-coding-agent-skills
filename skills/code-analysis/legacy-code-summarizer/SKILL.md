---
name: legacy-code-summarizer
description: Specializes in producing summaries for legacy systems lacking comments or documentation. Use when understanding legacy code, when the user asks about undocumented code, or when onboarding to an inherited codebase.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "code-summarizer, code-translator"
---

# Legacy Code Summarizer

## Purpose

Summarize legacy code that lacks comments or documentation.

## Workflow

1. **Input**: Legacy code (file, module, or subsystem).
2. **Analyze**: Infer behavior from structure, naming, and patterns.
3. **Summarize**: Produce description despite missing docs.
4. **Flag**: Note ambiguities or inferred assumptions.

## Output

- Summary of inferred behavior
- Assumptions and uncertainties
- Suggested areas for clarification
