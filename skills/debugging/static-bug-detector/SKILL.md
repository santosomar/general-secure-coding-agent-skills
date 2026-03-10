---
name: static-bug-detector
description: Identifies bugs through static code analysis (null dereferences, type mismatches, control flow issues) without executing the program. Use when scanning code for defects before running tests, when the user asks for static analysis, or when integrating with CI for defect detection.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Static Bug Detector

## Purpose

Detect defects via static analysis: null/undefined dereferences, type mismatches, unreachable code, and control-flow anomalies.

## Workflow

1. **Parse and analyze**: Build AST/IR and run data-flow and control-flow analyses.
2. **Check for defects**: Null dereference, type errors, dead code, uninitialized variables, resource leaks.
3. **Rank by severity**: Classify findings by likelihood and impact.
4. **Report**: Output findings with location and remediation hint.

## Output

- List of static defects (file:line, rule, severity)
- Brief explanation and suggested fix
- Optional integration format (SARIF, etc.)
