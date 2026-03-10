---
name: requirement-comparison-reporter
description: Compares two versions of a requirements document and reports additions, removals, semantic changes, and scope drift — distinguishing clerical edits from meaning changes. Use when a spec was revised, when checking if a new version of a standard affects you, or when the user asks what changed between spec versions.
license: Apache-2.0
metadata:
  category: "requirements"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "requirement-summarizer, requirement-coverage-checker"
---

# Requirement Comparison Reporter

A text diff of two specs is noise. Every reworded sentence shows as changed. The useful question is: **what changed in meaning?**

## Change classification

| Change class          | Meaning impact | Example                                                  |
| --------------------- | -------------- | -------------------------------------------------------- |
| **Added**             | New obligation | v2 adds: "MUST support TLS 1.3"                          |
| **Removed**           | Dropped obligation | v1 had: "MUST support TLS 1.0" — gone in v2         |
| **Strengthened**      | Tighter constraint | "SHOULD encrypt" → "MUST encrypt"; "under 1s" → "under 500ms" |
| **Weakened**          | Looser constraint  | "MUST validate" → "SHOULD validate"; "all inputs" → "user-facing inputs" |
| **Scope change**      | Same constraint, different applicability | "Applies to all endpoints" → "applies to public endpoints" |
| **Clerical**          | None           | Typo fix, renumbering, example added                     |

## Step 1 — Align

Match v1 requirements to v2 requirements. Don't diff line-by-line; diff **requirement-by-requirement**:

- Same ID → same requirement (easy case).
- No IDs → match by content similarity (cosine on embeddings, or just longest common subsequence of key terms).
- Unmatched in v1 → removed. Unmatched in v2 → added.

## Step 2 — Classify each matched pair

For each (v1_req, v2_req) pair, extract the claim structure:

| Component        | Extract                                              |
| ---------------- | ---------------------------------------------------- |
| Modal            | MUST / SHOULD / MAY / MUST NOT                       |
| Subject          | Who/what the requirement is about                    |
| Predicate        | What they must/should/may do                         |
| Condition        | "When X", "if Y", "unless Z"                         |
| Quantity         | Numbers, ranges, thresholds                          |

Diff component-by-component. Modal change = strengthen/weaken. Quantity change = strengthen/weaken (direction depends). Condition change = scope change.

## Worked example

**v1 §4.2:** "The server SHOULD log authentication failures. Logs should be retained for at least 30 days."

**v2 §4.2:** "The server MUST log all authentication events (success and failure) with timestamp, source IP, and user identifier. Logs MUST be retained for 90 days in tamper-evident storage."

**Component diff:**

| Component    | v1                          | v2                                          | Delta                  |
| ------------ | --------------------------- | ------------------------------------------- | ---------------------- |
| Modal        | SHOULD                      | MUST                                        | **Strengthened**       |
| Subject      | server                      | server                                      | Same                   |
| Predicate    | log auth failures           | log all auth events + required fields       | **Scope expanded** (failures → all events) + **Added** (field requirements) |
| Quantity     | 30 days                     | 90 days                                     | **Strengthened** (longer retention) |
| Condition    | (none)                      | tamper-evident storage                      | **Added** (storage constraint) |

**Impact:** Existing implementation that logs failures only, 30-day retention, to a regular file: now **non-compliant** on 4 points. This is not a clerical revision.

## The "hidden weakening" pattern

Watch for weakening disguised as addition:

**v1:** "All API responses MUST be validated against the schema."
**v2:** "All API responses MUST be validated against the schema, except during maintenance windows."

Text diff shows an *addition*. Semantic diff shows a *weakening* — the exception carves out a case where the constraint doesn't hold.

## Do not

- **Do not** report reworded-but-equivalent requirements as changes. "The system must" → "The server must" (when they're the same thing) is clerical. Check that the subject *actually* changed.
- **Do not** miss renumbering. v1 §4.2 might be v2 §5.1. Align by content, not section number.
- **Do not** treat added examples as added requirements. "For example, a user with role X can..." is illustrative, not normative. (Unless the doc says examples are normative.)
- **Do not** miss requirements that moved between sections. A requirement deleted from §4 and added to §7 is a *move*, not a remove+add. The section move might itself be a scope change (§4 = mandatory, §7 = optional).

## Output format

```
## Documents
v1: <name/version/date>
v2: <name/version/date>

## Summary
| Class        | Count |
| ------------ | ----- |
| Added        |       |
| Removed      |       |
| Strengthened |       |
| Weakened     |       |
| Scope change |       |
| Clerical     | <not itemized> |

## Added
| v2 ref | Text | Impact |
| ------ | ---- | ------ |

## Removed
| v1 ref | Text | Was anyone relying on this? |
| ------ | ---- | --------------------------- |

## Strengthened
| v1 ref | v2 ref | v1 | v2 | Delta | Implementations affected |
| ------ | ------ | -- | -- | ----- | ------------------------ |

## Weakened
<same columns — flag "hidden weakenings" via exception clauses>

## Scope changes
<same>

## Compliance impact
<for a known implementation: which changes break compliance, prioritized>
```
