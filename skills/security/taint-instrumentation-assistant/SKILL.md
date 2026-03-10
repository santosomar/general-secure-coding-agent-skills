---
name: taint-instrumentation-assistant
description: Assists in setting up taint tracking to trace untrusted input through the program and identify dangerous sinks. Use when setting up taint analysis, when the user asks for taint tracking, or when tracing data flow for security.
license: Apache-2.0
metadata:
  category: "security"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Taint Instrumentation Assistant

## Purpose

Set up taint tracking to trace untrusted input to dangerous sinks.

## Workflow

1. **Define sources**: Mark untrusted inputs (HTTP params, file, env).
2. **Define sinks**: Mark dangerous operations (exec, query, eval).
3. **Instrument**: Add taint propagation or configure tool.
4. **Validate**: Run and verify taint flows are detected.

## Output

- Taint configuration or instrumentation
- Source and sink definitions
- Usage instructions
