---
name: model-guided-code-repair
description: Repairs implementation code guided by a formal model to correct violations of specified properties. Use when fixing code to satisfy a formal model, when the user has a model violation to fix, or when model-guided repair is needed.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Model-Guided Code Repair

## Purpose

Repair implementation code guided by a formal model to fix property violations.

## Workflow

1. **Input**: Implementation, formal model, violation trace.
2. **Analyze**: Map violation to implementation location.
3. **Repair**: Propose code change to align with model.
4. **Verify**: Re-check model or run tests.

## Output

- Proposed code fix
- Mapping from model to implementation
- Verification result
