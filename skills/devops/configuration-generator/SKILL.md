---
name: configuration-generator
description: Generates infrastructure and application configuration files (YAML, TOML, env files) based on project requirements. Use when creating config files for a new service, when the user asks for config templates, or when standardizing config across environments.
license: Apache-2.0
metadata:
  category: "devops"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Configuration Generator

## Purpose

Produce config files (YAML, TOML, .env) for infrastructure and application based on requirements.

## Workflow

1. **Gather requirements**: Environment (dev/staging/prod), service type, integrations (DB, cache, queue).
2. **Select format**: Choose config format (YAML, TOML, env) per project.
3. **Generate**: Output config with placeholders and documented defaults.
4. **Validate**: Ensure syntax is valid and required keys are present.

## Output

- Config file(s) per environment
- Documentation of keys and defaults
- Secret placeholders (never real values)
