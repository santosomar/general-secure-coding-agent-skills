---
name: bug-to-patch-generator
description: Automatically synthesizes code patches to fix identified bugs, leveraging the bug location and surrounding context. Use when a bug has been localized and the user wants an automated fix, when generating candidate patches for review, or when the user asks to fix a specific bug.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "bug-localization, patch-correctness-verifier"
---

# Bug-to-Patch Generator

## Purpose

Produce minimal, correct code patches for localized bugs using the fault location, program context, and (when available) failing/passing test signals.

## Workflow

1. **Input**: Fault location (file, function, line) and optional bug description or failing test.
2. **Context extraction**: Gather surrounding code, type information, and call sites.
3. **Patch synthesis**: Apply fix strategies (e.g., condition correction, null checks, boundary fixes, API misuse correction) guided by program semantics.
4. **Validation**: Ensure the patch compiles and, if tests exist, that failing tests pass and passing tests remain passing.
5. **Output**: Minimal diff-style patch with explanation.

## Output

- Unified diff or inline edit
- Brief explanation of the fix
- Any assumptions or limitations
