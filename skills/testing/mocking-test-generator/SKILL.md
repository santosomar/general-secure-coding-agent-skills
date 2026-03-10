---
name: mocking-test-generator
description: Generates unit tests with appropriate mocks and stubs for external dependencies. Use when generating tests with mocks, when the user asks for mocked tests, or when isolating dependencies.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Mocking Test Generator

## Purpose

Generate unit tests with mocks and stubs for external dependencies.

## Workflow

1. **Input**: Code under test (with dependencies).
2. **Identify deps**: Find external calls (DB, API, file).
3. **Generate mocks**: Create mocks/stubs for each dependency.
4. **Generate tests**: Produce tests with mock setup and assertions.
5. **Output**: Test code with mocks.

## Output

- Test code with mocks/stubs
- Mock setup and verification
- Framework-specific (e.g., Mockito, unittest.mock)
