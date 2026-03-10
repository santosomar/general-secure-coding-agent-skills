---
name: rollback-strategy-advisor
description: Recommends rollback procedures and strategies for failed deployments based on the deployment method and infrastructure. Use when planning rollback procedures, when a deployment fails, or when the user asks how to roll back.
license: Apache-2.0
metadata:
  category: "devops"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Rollback Strategy Advisor

## Purpose

Recommend rollback procedures for failed deployments based on deployment method and infrastructure.

## Workflow

1. **Input**: Deployment method (K8s, ECS, VM, serverless), artifact type, stateful components.
2. **Identify rollback steps**: Version revert, traffic shift, config rollback, DB migration revert.
3. **Generate procedure**: Steps for safe rollback with verification.
4. **Document**: Preconditions, rollback commands, verification checks.

## Output

- Step-by-step rollback procedure
- Commands or config changes for rollback
- Verification and rollback completion criteria
