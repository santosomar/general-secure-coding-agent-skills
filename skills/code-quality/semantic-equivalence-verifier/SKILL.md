---
name: semantic-equivalence-verifier
description: Formally or heuristically checks that two code versions are semantically equivalent. Use when comparing two implementations, when verifying refactoring correctness, or when the user asks if two code versions are equivalent.
license: Apache-2.0
metadata:
  category: "code-quality"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Semantic Equivalence Verifier

## Purpose

Check that two code versions are semantically equivalent (formally or heuristically).

## Workflow

1. **Input**: Two code versions (e.g., before/after refactor).
2. **Analyze**: Compare control flow, data flow, invariants.
3. **Verify**: Use formal methods (if available) or heuristics to establish equivalence.
4. **Report**: Equivalence result and any differences found.

## Output

- Equivalence result (equivalent / not equivalent / unknown)
- Differences if not equivalent
- Confidence or proof sketch
