---
name: critical-interval-security-checker
description: Checks security properties during critical execution intervals such as authentication windows or transaction boundaries. Use when assessing security during auth or transaction flows, when the user asks for critical-interval checks, or when validating security invariants.
license: Apache-2.0
metadata:
  category: "security"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Critical Interval Security Checker

## Purpose

Check security properties during critical intervals (auth, transaction boundaries).

## Workflow

1. **Identify intervals**: Auth flows, transaction boundaries, privilege escalation.
2. **Define invariants**: What must hold (e.g., auth before access, atomicity).
3. **Analyze**: Verify invariants hold during the interval.
4. **Report**: Violations with location and remediation.

## Output

- Invariant check results
- Violations with location
- Remediation suggestions
