---
name: code-search-assistant
description: Helps find relevant code within large codebases using semantic or structural queries. Use when searching for code by meaning, when the user asks where something is implemented, or when navigating a large codebase.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Code Search Assistant

## Purpose

Find relevant code using semantic or structural queries.

## Workflow

1. **Input**: Query (natural language, pattern, or structural).
2. **Search**: Use semantic search, grep, or AST-based search.
3. **Rank**: Order results by relevance.
4. **Output**: List of matches with context.

## Output

- Ranked list of matching locations
- Snippet and file path
- Brief relevance explanation
