---
name: requirement-summary
description: Alias for requirement-summarizer. Produces a structured summary of a requirements document — the key obligations, grouped by actor and concern, with the MUST/SHOULD/MAY breakdown. Use when onboarding to a large spec, when deciding what to implement first, or when the user asks what a standard actually requires.
license: Apache-2.0
metadata:
  category: "requirements"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "requirement-summarizer"
---

# Requirement Summary

**This skill is an alias.** → `requirement-summarizer` for the full workflow.

The two names exist because "summary" (the noun — the artifact you want) and "summarizer" (the verb — the thing that produces it) both get asked for. They mean the same thing. Follow the link.

## Quick reference

What the target skill does:

| Input                    | Output                                                   |
| ------------------------ | -------------------------------------------------------- |
| Requirements document    | Obligations table: actor → MUST / SHOULD / MAY           |
| RFC / standard           | Normative statements extracted, informative prose stripped |
| Long spec                | Grouped by concern (auth, storage, transport, …)         |

## When you're here by mistake

| You actually wanted…                                       | Go to                                 |
| ---------------------------------------------------------- | ------------------------------------- |
| To compare two versions of requirements                    | → `requirement-comparison-reporter`   |
| To check if code covers requirements                       | → `requirement-coverage-checker`      |
| To make one vague requirement precise                      | → `requirement-enhancer`              |
| To turn a requirement into concrete test scenarios         | → `scenario-generator`                |

Otherwise: → `requirement-summarizer`.
