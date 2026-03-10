---
name: code-comment-generator
description: Generates accurate, meaningful inline comments and docstrings for functions, classes, and modules. Use when adding documentation to code, when the user asks for comments or docstrings, or when improving code readability.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Code Comment Generator

## Purpose

Generate inline comments and docstrings for functions, classes, and modules.

## Workflow

1. **Input**: Code without or with sparse comments.
2. **Analyze**: Understand purpose, parameters, return values, invariants.
3. **Generate**: Add docstrings (params, returns, raises) and inline comments for non-obvious logic.
4. **Format**: Follow language/framework conventions (e.g., JSDoc, reStructuredText).

## Output

- Docstrings for public APIs
- Inline comments for complex logic
- Format-compliant with project style
