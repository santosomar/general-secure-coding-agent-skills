---
name: traceability-matrix-generator
description: Generates requirements traceability matrices (RTM) linking requirements to design elements, code, and tests. Use when building traceability, when the user asks for an RTM, or when preparing for compliance or audit.
license: Apache-2.0
metadata:
  category: "requirements"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "req-to-test, coverage-analyzer"
---

# Traceability Matrix Generator

## Purpose

Generate RTM linking requirements to design, code, and tests.

## Workflow

1. **Input**: Requirements, design/code, tests.
2. **Trace**: Map requirements to design elements, code, tests.
3. **Build matrix**: Create RTM (requirement × artifact).
4. **Report**: Matrix with coverage and gaps.

## Output

- Traceability matrix (table or export)
- Coverage summary
- Gaps and recommendations
