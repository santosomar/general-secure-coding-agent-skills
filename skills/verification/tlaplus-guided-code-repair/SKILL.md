---
name: tlaplus-guided-code-repair
description: Uses TLA+ model checker output (violations, traces) to guide precise code repairs. Use when fixing code based on TLA+ violations, when the user has a TLC counterexample, or when TLA+-specific repair is needed.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# TLA+-Guided Code Repair

## Purpose

Use TLA+ model checker output (violations, traces) to guide code repairs.

## Workflow

1. **Input**: TLC counterexample or violation trace.
2. **Map**: Correlate trace to implementation.
3. **Identify**: Pinpoint the implementation decision that caused the violation.
4. **Repair**: Propose fix to satisfy the TLA+ property.
5. **Cross-ref**: See `verification-model-guided-code-repair` for generic model-guided repair.

## Output

- Code fix with explanation
- Trace-to-code mapping
- Updated TLA+ model if needed
