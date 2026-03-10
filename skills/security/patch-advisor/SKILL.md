---
name: patch-advisor
description: Recommends the specific code change to remediate a detected vulnerability by dispatching on CWE to the matching Project CodeGuard rule's prescribed fix pattern. Use after a finding has been confirmed and located, when the user asks how to fix a vulnerability, or when generating remediation PRs.
license: Apache-2.0
metadata:
  category: "security"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  upstream: "cosai-oasis/project-codeguard"
  codeguard-version: "1.3.0"
  related: "static-vulnerability-detector, vulnerability-pattern-matcher"
---

# Patch Advisor

This skill **delegates to Project CodeGuard** for remediation patterns. Every CodeGuard rule includes an "Implementation Checklist" and concrete before→after code; this skill is the CWE→rule→fix lookup.

**Upstream:** <https://github.com/cosai-oasis/project-codeguard/tree/main/skills/software-security>

## Dispatch (CWE → CodeGuard rule → fix section)

| CWE    | CodeGuard rule                              | Fix pattern                           |
| ------ | ------------------------------------------- | ------------------------------------- |
| 89     | `codeguard-0-input-validation-injection`    | PreparedStatement / parameterized query examples |
| 78     | `codeguard-0-input-validation-injection`    | ProcessBuilder / structured-exec + arg allow-list |
| 79     | `codeguard-0-client-side-web-security`      | Context-aware encoding, DOMPurify, Trusted Types |
| 502    | `codeguard-0-xml-and-serialization`         | `yaml.safe_load`, `ObjectInputStream` allow-list, `TypeNameHandling=None` |
| 611    | `codeguard-0-xml-and-serialization`         | `disallow-doctype-decl`, `DtdProcessing.Prohibit`, `defusedxml` |
| 22     | `codeguard-0-file-handling-and-uploads`     | Canonicalize-then-prefix-check; value allow-list |
| 798    | `codeguard-1-hardcoded-credentials`         | KMS/vault extraction; env injection at runtime |
| 327    | `codeguard-1-crypto-algorithms`             | Algorithm substitution table (MD5→SHA-256, AES-ECB→AES-GCM) |
| 862    | `codeguard-0-authorization-access-control`  | User-scoped query; middleware enforce; DTO allow-list |

## Workflow

1. Take CWE + language from the upstream finding.
2. Look up the rule; extract the fix pattern for that language.
3. Emit: the minimal diff, the rule ID it satisfies, and the rule's test-plan line for verification.
4. If the CWE isn't in the table, fall back to the CodeGuard language→rules map and apply the closest rule's Implementation Checklist.
