---
name: spring-mvc-to-boot-migrator
description: Automates migration from Spring MVC to Spring Boot, updating configs, annotations, and project structure. Use when migrating Spring MVC to Boot, when the user asks for Spring Boot migration, or when modernizing a Spring app.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Spring MVC to Boot Migrator

## Purpose

Automate migration from Spring MVC to Spring Boot.

## Workflow

1. **Analyze**: Scan Spring MVC config (XML, Java config), controllers, dependencies.
2. **Map**: Identify Boot equivalents (auto-config, starters).
3. **Migrate**: Update configs, annotations, project structure.
4. **Generate**: Produce Boot project with migrated code.
5. **Validate**: Ensure app runs and behavior is preserved.

## Output

- Migrated Spring Boot project
- Config and annotation changes
- Migration notes and manual steps
