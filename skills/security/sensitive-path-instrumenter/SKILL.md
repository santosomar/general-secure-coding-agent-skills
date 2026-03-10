---
name: sensitive-path-instrumenter
description: Instruments code paths that handle sensitive data (auth, encryption, PII) for monitoring and audit. Use when adding security monitoring, when the user asks to instrument sensitive paths, or when preparing for audit.
license: Apache-2.0
metadata:
  category: "security"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Security-Sensitive Path Instrumenter

## Purpose

Instrument code paths for auth, encryption, PII for monitoring and audit.

## Workflow

1. **Identify**: Find sensitive paths (auth, crypto, PII access).
2. **Instrument**: Add logging (non-sensitive), metrics, or audit hooks.
3. **Generate**: Output instrumented code or config.
4. **Document**: Explain what is logged and retention.

## Output

- Instrumented code or config
- List of instrumented paths
- Logging/audit documentation
