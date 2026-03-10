---
name: cd-pipeline-generator
description: Creates continuous delivery/deployment pipeline configs including staging, approval gates, and production deployment steps. Use when setting up CD, when the user asks for deployment pipelines, or when adding staging or approval gates.
license: Apache-2.0
metadata:
  category: "devops"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "ci-pipeline-synthesizer, release-manager, rollback-strategy-generator"
---

# CD Pipeline Generator

## Purpose

Generate CD pipeline configs with staging, approval gates, and production deployment steps.

## Workflow

1. **Gather requirements**: Target environments (dev, staging, prod), deployment method (K8s, ECS, VM), approval needs.
2. **Design stages**: Build → test → staging deploy → approval → prod deploy.
3. **Generate config**: Output pipeline with stages, gates, secrets handling, and rollback hooks.
4. **Document**: Explain each stage and how to configure approval.

## Output

- CD pipeline config (YAML or equivalent)
- Stage definitions and triggers
- Approval and rollback instructions
