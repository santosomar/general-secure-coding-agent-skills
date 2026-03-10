---
name: metamorphic-test-generator
description: Creates metamorphic test relations and test cases that detect bugs without a traditional test oracle. Use when testing without a known oracle, when the user asks for metamorphic tests, or when testing complex or non-deterministic systems.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "test-oracle-generator, property-based-test-generator"
---

# Metamorphic Test Generator

## Purpose

Create metamorphic test relations and cases for systems without a traditional oracle.

## Workflow

1. **Input**: Program or function under test.
2. **Identify relations**: Find metamorphic relations (e.g., if f(x)=y then f(f(x))=f(y)).
3. **Generate**: Create test pairs that satisfy the relation.
4. **Output**: Test code that checks relations hold.

## Output

- Metamorphic relations
- Test cases that check relations
- Test code
