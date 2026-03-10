---
name: code-pattern-extractor
description: Identifies and extracts recurring code patterns or idioms across a codebase. Use when finding common patterns, when the user asks for pattern analysis, or when refactoring to shared abstractions.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Code Pattern Extractor

## Purpose

Identify and extract recurring patterns or idioms across the codebase.

## Workflow

1. **Analyze**: Parse codebase, build AST or IR.
2. **Detect**: Find repeated structures (clone detection, idiom mining).
3. **Extract**: Group similar occurrences into patterns.
4. **Report**: List patterns with locations and frequency.

## Output

- List of patterns with locations
- Frequency and examples
- Refactoring suggestion (e.g., extract to shared function)
