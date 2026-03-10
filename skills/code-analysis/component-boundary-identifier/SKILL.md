---
name: component-boundary-identifier
description: Detects natural component or service boundaries within a monolith, aiding modularization and microservices extraction. Use when decomposing a monolith, when the user asks for component boundaries, or when planning modularization.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "architecture-recoverer, dependency-graph-builder"
---

# Component Boundary Identifier

## Purpose

Detect natural component or service boundaries in a monolith.

## Workflow

1. **Analyze**: Build dependency graph, coupling metrics.
2. **Cluster**: Identify cohesive modules with low inter-module coupling.
3. **Boundaries**: Propose component boundaries (packages, services).
4. **Report**: Boundaries with dependency map and extraction order.

## Output

- Proposed component boundaries
- Dependency graph between components
- Suggested extraction order
