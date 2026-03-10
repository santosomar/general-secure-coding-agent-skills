---
name: test-deduplicator
description: Identifies and removes redundant or near-duplicate tests to keep suites lean and fast. Use when deduplicating tests, when the user asks to remove redundant tests, or when speeding up the test suite.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Test Deduplicator

## Purpose

Identify and remove redundant or near-duplicate tests.

## Workflow

1. **Input**: Test suite.
2. **Analyze**: Compare tests (structure, coverage, behavior).
3. **Cluster**: Group similar or redundant tests.
4. **Recommend**: Suggest removals or merges.
5. **Output**: Deduplicated suite or removal list.

## Output

- List of redundant or duplicate tests
- Recommended removals
- Coverage impact (if any)
