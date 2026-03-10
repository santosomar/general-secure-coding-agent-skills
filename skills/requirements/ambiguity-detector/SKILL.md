---
name: ambiguity-detector
description: Detects ambiguity in natural-language requirements — weak words, dangling references, underspecified quantities, conflicting interpretations — before they become implementation bugs. Use when reviewing requirements, when a spec uses words like "appropriate" or "fast", or when two engineers read the same requirement and built different things.
license: Apache-2.0
metadata:
  category: "requirements"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "requirement-enhancer, nl-to-constraints"
---

# Ambiguity Detector

A requirement is ambiguous if two reasonable people can read it and disagree on what it says. Every ambiguity is a future bug, a future argument, or both.

## Ambiguity taxonomy

| Type                   | Signal words/patterns                            | Example                                      |
| ---------------------- | ------------------------------------------------ | -------------------------------------------- |
| **Vague qualifier**    | "fast", "appropriate", "user-friendly", "robust", "efficient", "secure" | "The system should respond quickly" — how quickly? |
| **Dangling reference** | "it", "this", "the above", "as needed"           | "Validate input and log it" — log the input or the validation result? |
| **Unquantified**       | "some", "most", "several", "regularly"           | "Cache most recent entries" — how many?      |
| **Ambiguous scope**    | "all", "any", "each" with unclear domain         | "All users can view reports" — all users, or all *authorized* users? |
| **Underspecified conjunction** | "and/or", "either", "unless"            | "Accept JSON or XML" — both? pick one? configurable? |
| **Passive voice hiding actor** | "will be validated", "must be logged"   | "Input will be validated" — by whom? client? server? both? |
| **Temporal ambiguity** | "eventually", "after", "when", "periodically"    | "Notify the user when the job completes" — immediately? next login? |
| **Negative scope**     | "not ... and/or ..."                             | "Not A and B" — ¬(A∧B) or (¬A)∧B?            |

## The two-interpretations test

For each flagged phrase: can you write **two concrete implementations** that both satisfy the text but behave differently? If yes, it's ambiguous.

**Requirement:** "The search should return relevant results."

- Interpretation A: return everything matching the query, sorted by relevance score.
- Interpretation B: return only results above a relevance threshold (precision over recall).

Both "return relevant results." Different systems. Flag it.

## Worked example

**Requirement as written:**

> The API should handle concurrent requests efficiently. When the load is high, requests may be queued. Large payloads should be rejected with an appropriate error.

**Scan:**

| Phrase                     | Type               | Two interpretations                                          |
| -------------------------- | ------------------ | ------------------------------------------------------------ |
| "efficiently"              | Vague qualifier    | (a) p99 < 100ms under 1k RPS. (b) No more than 2× single-request latency at any load. |
| "the load is high"         | Unquantified       | (a) > 80% CPU. (b) > 500 concurrent requests. (c) queue depth > N. |
| "may be queued"            | Ambiguous modality | (a) queueing is permitted. (b) queueing is what happens (mandatory). |
| "Large payloads"           | Unquantified       | (a) > 1 MB. (b) > 10 MB. (c) configurable.                   |
| "appropriate error"        | Vague qualifier    | (a) HTTP 413. (b) HTTP 400 with message. (c) close the connection. |
| "should" (×2)              | Weak modal         | "should" vs "must" — is this optional?                       |

**Rewrite prompt** (feed to → `requirement-enhancer`):

> The API MUST handle at least {N} concurrent requests with p99 latency under {T} ms. When concurrent requests exceed {N}, additional requests MUST be queued (max queue depth {Q}; beyond that, respond 503). Payloads exceeding {S} bytes MUST be rejected with HTTP 413 and body `{"error": "payload_too_large", "max_bytes": {S}}`.

The `{N}`, `{T}`, `{Q}`, `{S}` are holes for the stakeholder to fill. The ambiguity detector's job is to surface the holes, not fill them.

## Context-dependent false positives

Some "weak" words are fine in context:

| Looks ambiguous            | Actually fine when                                          |
| -------------------------- | ----------------------------------------------------------- |
| "the user"                 | There's only one kind of user in the system                 |
| "the database"             | Single-database architecture                                |
| "standard format"          | A standard is cited elsewhere in the doc (check!)           |
| "as described above"       | "Above" is the immediately preceding numbered item          |

Don't flag mechanically. Check if context resolves it.

## Do not

- **Do not** rewrite the requirement yourself with invented numbers. "quickly → under 200ms" — where did 200 come from? Flag the hole; let the stakeholder fill it.
- **Do not** flag every "should." In RFC 2119 style, SHOULD is a deliberate weakening of MUST. Flag it only if the doc doesn't use RFC 2119 conventions and the intent is unclear.
- **Do not** miss structural ambiguity while hunting word-level ambiguity. "Validate X and Y or Z" — is that (X ∧ Y) ∨ Z or X ∧ (Y ∨ Z)? Parenthesization matters.
- **Do not** treat vague non-functional requirements as less important. "The system should be secure" is the most expensive kind of ambiguity.

## Output format

```
## Requirement (verbatim)
<text>

## Ambiguities
| # | Phrase | Type | Interpretation A | Interpretation B | Resolution needed |
| - | ------ | ---- | ---------------- | ---------------- | ----------------- |

## Questions for stakeholder
1. <concrete question whose answer resolves ambiguity #N>
...

## Suggested restructure
<template with {HOLES} for the stakeholder to fill — not filled in>
```
