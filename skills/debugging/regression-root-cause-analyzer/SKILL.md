---
name: regression-root-cause-analyzer
description: Traces regressions to the specific commit, change, or code path that introduced the behavioral breakage. Use when a previously passing test or feature now fails, when the user asks what change caused a regression, or when bisecting a regression across commits.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "bug-localization, git-bisect-automator"
---

# Regression Root Cause Analyzer

## Purpose

Identify the commit, diff, or code path that introduced a regression by correlating VCS history with failing behavior.

## Workflow

1. **Define regression**: Failing test or observable behavior that used to pass.
2. **Bisect or analyze**: Use git bisect, blame, or semantic diff to narrow the introducing change.
3. **Correlate**: Map the regression to specific lines or functions in the introducing commit.
4. **Explain**: Summarize what change caused the break and why.

## Output

- Introducing commit (hash, message)
- Changed files and relevant hunks
- Explanation of how the change caused the regression
