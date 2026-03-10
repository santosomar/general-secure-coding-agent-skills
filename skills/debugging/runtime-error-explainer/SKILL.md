---
name: runtime-error-explainer
description: Translates cryptic runtime errors (stack overflows, segfaults, exceptions) into clear, human-readable explanations with suggested fixes. Use when the user encounters a runtime error, when explaining a crash or exception to a developer, or when the user asks what an error means.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Runtime Error Explainer

## Purpose

Turn raw runtime errors (exceptions, segfaults, stack overflows) into plain-language explanations and actionable fixes.

## Workflow

1. **Input**: Error message, stack trace, or crash log.
2. **Parse**: Extract exception type, message, stack frames, and context.
3. **Explain**: Describe what went wrong in plain language (e.g., null dereference, index out of bounds, recursion limit).
4. **Suggest**: Propose fixes (null checks, bounds validation, base case, resource cleanup).
5. **Context**: If source is available, point to the relevant lines.

## Output

- Plain-language explanation of the error
- Root cause summary
- Suggested fixes with code snippets when appropriate
- Relevant source locations if available
