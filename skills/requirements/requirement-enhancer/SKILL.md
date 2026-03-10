---
name: requirement-enhancer
description: Rewrites vague or incomplete requirements into precise, testable statements — filling in quantities, actors, conditions, and error behavior while preserving intent. Use after ambiguity-detector flags problems, when a requirement can't be turned into a test, or when engineers keep asking the same clarification questions.
license: Apache-2.0
metadata:
  category: "requirements"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "ambiguity-detector, nl-to-constraints"
---

# Requirement Enhancer

A good requirement is **testable**: you can write a test, run it, and get a definitive pass/fail. Enhancement is rewriting until that's true.

Input: a vague requirement + the answers to → `ambiguity-detector`'s questions. Output: a testable requirement.

## The testability checklist

A requirement is testable when it specifies:

| Component     | Question it answers                       | If missing                          |
| ------------- | ----------------------------------------- | ----------------------------------- |
| **Actor**     | Who/what does this?                       | "Input will be validated" — by whom? |
| **Action**    | What exactly happens?                     | "Handle errors" — how?              |
| **Object**    | On what?                                  | "Validate data" — which data?       |
| **Condition** | When does this apply?                     | Always? Only on POST?               |
| **Criterion** | How do we know it worked?                 | "Respond quickly" — measured how?   |
| **Exception** | What if it fails / doesn't apply?         | Silent fail? Error? Degrade?        |

Every enhanced requirement should answer all six, explicitly or by obvious default.

## Enhancement patterns

| Vague input                        | Enhancement pattern                                           |
| ---------------------------------- | ------------------------------------------------------------- |
| "should be fast"                   | "p{N} latency MUST be under {T} ms at {L} RPS"                |
| "handle errors gracefully"         | "On {error type}, {actor} MUST {log + return HTTP {code} + body {schema}}" |
| "validate input"                   | "{Actor} MUST reject requests where {field} does not match {regex/range/enum}, responding {code}" |
| "support large files"              | "MUST accept files up to {N} MB; files exceeding {N} MB MUST be rejected with {code}" |
| "be secure"                        | Decompose into specific controls — this is 20 requirements, not one |
| "eventually consistent"            | "Writes MUST be visible to all readers within {T} seconds under {conditions}" |
| "retry on failure"                 | "On {error types}, retry up to {N} times with {backoff strategy}; after {N} failures, {final action}" |

## Worked example

**Input (after ambiguity-detector pass, with stakeholder answers in brackets):**

> The search should return relevant results quickly.
> [Stakeholder: "quickly" = 95th percentile under 300ms at 100 RPS. "relevant" = by our existing ranking score, top 20, no threshold cutoff.]

**Enhancement:**

> **REQ-SEARCH-1 (latency):** The `/search` endpoint MUST return a response with p95 latency ≤ 300 ms when serving up to 100 requests per second.
>
> **REQ-SEARCH-2 (result set):** The `/search` endpoint MUST return the top 20 results by `ranking_score` (descending) for the given query. If fewer than 20 documents match, all matching documents MUST be returned.
>
> **REQ-SEARCH-3 (error handling):** If the search index is unavailable, the endpoint MUST return HTTP 503 with body `{"error": "search_unavailable", "retry_after": <seconds>}`.

**What was added:**
- Split into 3 atomic requirements (one claim each → one test each).
- REQ-SEARCH-3 wasn't in the original at all — but "what if it fails" is always a gap. Asked the stakeholder; added.
- "Top 20" raised the edge case "fewer than 20 match" — specified.

## Decomposition — one claim per requirement

"The system MUST authenticate users and authorize access and log all attempts" is three requirements. Split them:

- Tests are 1:1 with requirements. Compound requirements → compound tests → hard to diagnose failures.
- Coverage is per-requirement. You can't be "66% covered" on one requirement.
- Changes are per-requirement. If logging changes, only REQ-LOG changes.

## Preserving intent

Don't change what the requirement *means* — only how precisely it's stated. Red flags that you've drifted:

- You added a number the stakeholder never gave you. ("300 ms" when they said "fast" and you didn't ask.)
- You chose between two valid interpretations without checking. ("all users" → "authenticated users" — maybe, maybe not.)
- You added a clause that changes scope. ("MUST validate on POST" when the original didn't restrict to POST.)

**Every number, every choice, every added clause** should trace to a stakeholder answer or an explicit documented assumption.

## Do not

- **Do not** fill ambiguity with your guess. The enhancement is only as good as the stakeholder input. If you don't have the answer, leave a `{PLACEHOLDER}` and note the open question.
- **Do not** gold-plate. "should log errors" → "MUST log to structured JSON with correlation IDs and ship to SIEM" — unless the stakeholder asked for that, you've inflated scope.
- **Do not** leave compound requirements. "MUST do A and B and C" → three requirements.
- **Do not** use "should" when you mean "must" in the output. RFC 2119: MUST = mandatory, SHOULD = recommended but exceptions exist, MAY = optional. Pick the right one.

## Output format

```
## Original
<verbatim>

## Stakeholder inputs
<Q&A — every number and choice in the enhanced version traces to one of these>

## Enhanced
### REQ-<ID>-<N>
<text — MUST/SHOULD/MAY, actor, action, object, condition, criterion>

### REQ-<ID>-<N+1>
...

## Decomposition rationale
<why the original became N requirements>

## Open questions (placeholders remaining)
<anything still {UNFILLED} — blocks finalization>

## Testability check
| Req | Can write a test? | Test sketch |
| --- | ----------------- | ----------- |
```
