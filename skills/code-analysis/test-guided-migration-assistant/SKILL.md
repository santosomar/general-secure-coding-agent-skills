---
name: test-guided-migration-assistant
description: Uses the existing test suite as a safety harness to guide and validate code migration. Use when migrating code with tests as a guard, when the user asks for test-guided migration, or when ensuring migration preserves behavior.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Test-Guided Migration Assistant

## Purpose

Use the test suite to guide and validate code migration.

## Workflow

1. **Input**: Code to migrate, target (language/framework), existing tests.
2. **Migrate incrementally**: Translate in small steps, run tests after each.
3. **Fix failures**: Use test failures to correct migration.
4. **Validate**: All tests pass on migrated code.

## Output

- Migrated code
- Test run results
- Fixes applied for test failures
