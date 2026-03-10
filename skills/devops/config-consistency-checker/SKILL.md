---
name: config-consistency-checker
description: Detects inconsistencies or conflicts in configuration files across environments (dev, staging, prod). Use when validating config consistency, when the user asks if configs are aligned, or when troubleshooting environment-specific issues.
license: Apache-2.0
metadata:
  category: "devops"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Config Consistency Checker

## Purpose

Compare configs across environments and flag inconsistencies or conflicts.

## Workflow

1. **Collect configs**: Gather config files for dev, staging, prod (and any variants).
2. **Parse and normalize**: Load configs and normalize structure (keys, types).
3. **Compare**: Diff keys, values, and structure across environments.
4. **Report**: List differences, missing keys, and conflicting values.

## Output

- Diff report across environments
- Missing or extra keys per environment
- Recommendations for alignment
