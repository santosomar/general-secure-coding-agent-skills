---
name: dead-code-eliminator
description: Detects and safely removes unreachable, unused, or obsolete code. Use when cleaning up dead code, when the user asks to remove unused code, or when reducing codebase size.
license: Apache-2.0
metadata:
  category: "code-quality"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "code-refactoring-assistant, coverage-analyzer"
---

# Dead Code Eliminator

## Purpose

Find and safely remove unreachable, unused, or obsolete code.

## Workflow

1. **Detect**: Find unreachable code, unused exports, dead branches.
2. **Verify**: Confirm no dynamic references (reflection, etc.) before removal.
3. **Propose**: Generate diff removing dead code.
4. **Validate**: Run tests to ensure no regressions.

## Output

- List of dead code candidates with location
- Proposed removal diff
- Verification that removal is safe
