---
name: code-optimizer
description: Recommends and applies optimizations for runtime performance, memory usage, or code clarity. Use when optimizing performance, when the user asks for optimization suggestions, or when reducing resource usage.
license: Apache-2.0
metadata:
  category: "code-quality"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Code Optimizer

## Purpose

Recommend and apply optimizations for performance, memory, or clarity.

## Workflow

1. **Analyze**: Identify hotspots, redundant work, allocations, inefficient patterns.
2. **Suggest**: Propose optimizations (e.g., caching, early exit, lazy init).
3. **Apply**: Generate optimized code.
4. **Verify**: Ensure correctness and measure improvement.

## Output

- Optimization suggestions with before/after
- Expected impact (performance, memory)
- Code changes
