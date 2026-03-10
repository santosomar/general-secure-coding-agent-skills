---
name: technical-debt-analyzer
description: Quantifies technical debt, identifies high-debt modules, and prioritizes remediation efforts. Use when assessing codebase health, when the user asks for debt metrics, or when planning refactoring sprints.
license: Apache-2.0
metadata:
  category: "code-quality"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "code-smell-detector, code-review-assistant"
---

# Technical Debt Analyzer

## Purpose

Quantify technical debt and prioritize remediation by module or file.

## Workflow

1. **Collect metrics**: Complexity, duplication, test coverage, violations, age.
2. **Compute debt**: Score modules or files by debt indicators.
3. **Prioritize**: Rank by impact (e.g., high-debt, high-churn areas first).
4. **Report**: Debt summary with recommendations.

## Output

- Debt score per module/file
- Prioritized remediation list
- Suggested actions (refactor, test, document)
