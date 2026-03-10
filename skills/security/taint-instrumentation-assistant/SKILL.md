---
name: taint-instrumentation-assistant
description: Sets up taint tracking by defining sources, sinks, and sanitizers from Project CodeGuard's input-validation taxonomy, then configures the target tool (CodeQL, Semgrep, custom instrumentation). Use when wiring taint analysis into CI, when the user asks for taint tracking, or when you need a source/sink catalog for a specific language.
license: Apache-2.0
metadata:
  category: "security"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  upstream: "cosai-oasis/project-codeguard"
  codeguard-version: "1.3.0"
  related: "static-vulnerability-detector"
---

# Taint Instrumentation Assistant

This skill **delegates to Project CodeGuard** for its source/sink/sanitizer taxonomy — specifically `codeguard-0-input-validation-injection`, which defines the trust boundaries (HTTP params, env, files, IPC) and dangerous sinks (query execution, shell, eval, filesystem) per language.

**Upstream:** <https://github.com/cosai-oasis/project-codeguard/tree/main/skills/software-security>

## Dispatch

| Taint component | CodeGuard source                                               |
| --------------- | -------------------------------------------------------------- |
| Sources         | `codeguard-0-input-validation-injection` → "Core Strategy" trust boundaries, per-framework request-object tables |
| Sinks           | Same rule → SQL/LDAP/OS-command sections; plus `codeguard-0-xml-and-serialization` for deserialization sinks |
| Sanitizers      | Same rule → parameterization APIs, escaping functions, allow-list validators listed as "primary defense" |

## Workflow

1. Pull the language-specific source/sink/sanitizer lists from the CodeGuard rules above.
2. Translate into the target tool's format: CodeQL `.ql` source/sink predicates, Semgrep `pattern-sources`/`pattern-sinks`, or inline annotations.
3. Sanitizers are **allow-listed**, not inferred — only CodeGuard-recognized sanitizers suppress a flow. A regex-based "sanitizer" does not count.
4. Run and verify: inject a known-tainted flow and confirm it's flagged before trusting the configuration.
