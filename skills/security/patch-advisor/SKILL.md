---
name: patch-advisor
description: Recommends specific code changes or dependency upgrades to remediate identified security vulnerabilities. Use when remediating vulnerabilities, when the user asks how to fix a vulnerability, or when planning security patches.
license: Apache-2.0
metadata:
  category: "security"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "vulnerability-root-cause-analyzer, cve-advisor"
---

# Security Patch Advisor

## Purpose

Recommend code changes or dependency upgrades to remediate vulnerabilities.

## Workflow

1. **Input**: Vulnerability (location, type, CVE if applicable).
2. **Recommend**: For code issues: suggest fix (e.g., parameterize, sanitize). For deps: suggest upgrade or alternative.
3. **Generate**: Produce patch or upgrade instructions.
4. **Validation**: Ensure fix does not introduce new issues.

## Output

- Recommended fix (code patch or upgrade)
- Explanation of the fix
- Verification steps
