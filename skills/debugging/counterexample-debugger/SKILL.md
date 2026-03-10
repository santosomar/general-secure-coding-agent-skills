---
name: counterexample-debugger
description: Uses counterexamples produced by formal verifiers or model checkers to debug the precise failing execution trace. Use when a model checker or verifier reports a violation, when the user has a counterexample trace to analyze, or when debugging formal verification failures.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Counterexample Debugger

## Purpose

Interpret counterexamples from model checkers (TLC, NuSMV, etc.) or verifiers and map them to concrete execution traces for debugging.

## Workflow

1. **Input**: Counterexample trace (state sequence, variable valuations) from a model checker.
2. **Map to implementation**: Correlate model states with program locations and variables.
3. **Reconstruct execution**: Build a human-readable execution trace showing the path to the violation.
4. **Identify root cause**: Pinpoint the decision or transition that leads to the failure.
5. **Suggest fix**: Propose changes to avoid the violating trace.

## Output

- Human-readable execution trace
- Mapping from model to source
- Root cause and suggested fix
