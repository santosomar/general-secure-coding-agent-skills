---
name: bug-reproduction-test-generator
description: Creates minimal, reproducible test cases from bug reports to confirm the defect before and after a fix. Use when a bug is reported without a failing test, when the user needs a regression test for a fix, or when the user asks to reproduce a bug as a test.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "bug-localization, flaky-test-diagnoser"
---

# Bug Reproduction Test Generator

## Purpose

Turn a bug report (or failing scenario) into a minimal, executable test that reliably reproduces the defect and can serve as a regression guard.

## Workflow

1. **Input**: Bug description, stack trace, or reproduction steps.
2. **Extract scenario**: Identify the unit under test, inputs that trigger the bug, and expected vs actual behavior.
3. **Generate test**: Create a focused test (unit or integration) that sets up the scenario and asserts the correct behavior.
4. **Minimize**: Reduce the test to the smallest reproducing case (fewer dependencies, minimal setup).
5. **Validate**: Run the test to confirm it fails before the fix and passes after.

## Output

- Executable test code (JUnit, pytest, etc.)
- Setup instructions if needed
- Expected outcome (fail before fix, pass after)
