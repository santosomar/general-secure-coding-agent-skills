---
name: requirement-summarizer
description: Generates concise summaries of lengthy requirements documents for rapid understanding. Use when summarizing requirements, when the user asks for a requirements summary, or when onboarding to a requirements set.
license: Apache-2.0
metadata:
  category: "requirements"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "requirement-comparator, requirement-enhancer"
---

# Requirement Summarizer

## Purpose

Generate concise summaries of lengthy requirements documents.

## Workflow

1. **Input**: Requirements document(s).
2. **Parse**: Extract key requirements, structure, dependencies.
3. **Summarize**: Produce concise overview (executive summary, key points).
4. **Output**: Summary in target format.

## Output

- Concise summary
- Key requirements list
- Optional: stakeholder-focused variant (use --audience stakeholder)
