---
name: req-to-test
description: Derives executable test cases directly from requirements (user stories, acceptance criteria, specs) by extracting testable conditions, enumerating equivalence classes and boundaries, and producing a traceability map from each test back to its source requirement. Use when building acceptance tests from a spec, when checking whether requirements are covered by existing tests, when translating Gherkin or plain-English criteria into code, or when proving coverage for compliance.
license: Apache-2.0
metadata:
  category: "requirements"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "nl-to-constraints, traceability-matrix-generator, acceptance-test-generator"
---

# Req-to-Test

Turn each requirement into the smallest set of tests that (a) fail if the requirement is violated and (b) pass otherwise. Every test traces back to exactly one requirement. Every requirement has at least one test, or is explicitly marked untestable with the reason.

## Step 1 — Triage each requirement

Not all requirements produce tests the same way. Classify first:

| Type            | Recognition cues                                      | What it yields                          |
| --------------- | ----------------------------------------------------- | --------------------------------------- |
| Functional      | "shall," "the system does X," "when Y then Z"         | Direct input/output test cases          |
| Constraint      | "must not exceed," "within N ms," "at most," "never"  | Boundary and property tests             |
| Conditional     | "if," "unless," "when … otherwise"                    | One test per branch — including the *else* |
| State-based     | "after," "once … then," "until," references to phases | Sequence tests with setup state         |
| Non-functional  | "available," "secure," "scalable," "performant"       | Usually untestable at this level → split or defer |

If a requirement doesn't fit any row, it's probably ambiguous — route to → `ambiguity-detector` before wasting effort deriving tests from mush.

## Step 2 — Extract the testable kernel

Strip filler and normalize to **trigger → condition(s) → expected outcome**. If any of the three is missing, the requirement is incomplete and the gap is your first test.

**Requirement:** *"When a user submits an order with a total exceeding their credit limit, the system shall reject the order and display an error message explaining why."*

**Kernel:**
- Trigger: `order submitted`
- Condition: `order.total > user.creditLimit`
- Outcome 1: `order.status == REJECTED`
- Outcome 2: `error message visible AND mentions credit limit`

Two outcomes → two assertions. They can live in one test (same arrange/act), but both must be checked.

## Step 3 — Enumerate equivalence classes

The condition `order.total > user.creditLimit` splits the input space. Don't test every value — test one representative per class plus every boundary.

| Class                    | Representative                  | Expected     |
| ------------------------ | ------------------------------- | ------------ |
| Well below limit         | `total = limit × 0.5`           | Accepted     |
| At limit (boundary)      | `total = limit`                 | **Check the spec: is "exceeding" `>` or `>=`?** If ambiguous, both readings are tests until clarified. |
| Just over (boundary)     | `total = limit + smallest-unit` | Rejected     |
| Well over                | `total = limit × 2`             | Rejected     |

Boundaries are `0`, `1`, `limit-ε`, `limit`, `limit+ε`, `max`. For each, ask the spec. If the spec doesn't say — **that's a finding**, not a guess.

## Step 4 — Derive negative and abuse cases

Every positive test has a shadow. Enumerate:

1. **Precondition violated.** Requirement assumes a logged-in user → what if there is none? The spec should say; if it doesn't, that's a gap.
2. **Input malformed.** Total is negative, null, a string, NaN. If the spec is silent, the test documents the gap.
3. **Boundary abuse.** Total = 0. Total = `MAX_INT`. Credit limit = 0.
4. **The inverse path.** If rejection at `> limit` is specified, is *acceptance* at `<= limit` specified? Often one direction is implicit — make both explicit.

These aren't padding. A requirement that only specifies the happy path is a half-requirement, and the negative tests are how you prove that.

## Step 5 — Decide test level and emit

| Clue in the requirement        | Test level     | Framework style               |
| ------------------------------ | -------------- | ----------------------------- |
| "the function returns…"        | Unit           | pytest, JUnit, jest           |
| "the API responds with…"       | Integration    | pytest + httpx, supertest, RestAssured |
| "the user sees…", "the page shows…" | Acceptance / E2E | Gherkin → Playwright/Cypress |
| "under N concurrent users…"    | Load           | k6, Locust — note it, don't generate inline |

**Output template** (per test):

```
TEST: rejects_order_exceeding_credit_limit
  Requirement: REQ-042 (clause 2: "shall reject the order")
  Level: Integration
  Classes covered: just-over-boundary

  ARRANGE  user.creditLimit = 1000.00
           order.total      = 1000.01
  ACT      POST /orders
  ASSERT   response.status == 422
           response.body.error.code == 'CREDIT_LIMIT_EXCEEDED'

TEST: shows_credit_limit_error_message
  Requirement: REQ-042 (clause 3: "display an error message explaining why")
  Level: Acceptance
  ...
```

Follow with a **traceability map**:

```
REQ-042  →  rejects_order_exceeding_credit_limit   (clause 2, just-over)
         →  rejects_order_well_over_limit          (clause 2, well-over)
         →  accepts_order_at_limit                 (clause 2, boundary — assumes "exceeding" = strictly greater)
         →  shows_credit_limit_error_message       (clause 3)

REQ-043  →  [UNCOVERED — non-functional, "the system shall be responsive"]
            Action: split into measurable sub-requirements or defer to perf suite.
```

Explicitly list every uncovered requirement with a reason. Silence on coverage reads as "forgot," not "deliberately deferred."

## Worked example (Gherkin in, tests out)

**Input:**
```gherkin
Scenario: Free shipping threshold
  Given a cart with subtotal $50 or more
  When the user proceeds to checkout
  Then shipping cost is $0
```

**Analysis:**
- Condition: `subtotal >= 50` — "or more" is `>=`, not `>`. Boundary is 50 exactly.
- Outcome: `shipping == 0`
- Inverse is implicit. What *is* shipping below $50? The scenario doesn't say. That's a gap — but we can still test "not zero."

**Derived:**

| Test                            | Subtotal | Expected shipping | Covers                    |
| ------------------------------- | -------- | ----------------- | ------------------------- |
| `free_shipping_at_threshold`    | 50.00    | 0.00              | Boundary, inclusive       |
| `free_shipping_above_threshold` | 75.00    | 0.00              | Well-above class          |
| `paid_shipping_just_below`      | 49.99    | > 0.00            | Boundary-minus; exact value unspecified → assert non-zero only |
| `shipping_at_zero_subtotal`     | 0.00     | ??? — **GAP**     | Empty cart: is checkout even allowed? Flag to spec owner. |

Four tests, one gap surfaced. Done.

## Edge cases

- **Requirements that reference each other** ("as defined in REQ-017"): resolve the reference first. If REQ-017 is ambiguous, every derived test inherits the ambiguity. Fix upstream.
- **"The system shall be secure":** Untestable as written. Either decompose into testable sub-requirements ("shall reject requests without a valid token," "shall hash passwords with argon2id") or mark `UNTESTABLE — too broad` and kick back to the author.
- **Conflicting requirements:** REQ-A says "round up," REQ-B says "round to nearest." Write a test for each. One will fail. That's the point — the tests expose the conflict more convincingly than a meeting would.
- **"Shall log X":** Testable — assert on log output. Don't skip these as untestable; they're often the only spec for observability.
- **Temporal requirements** ("within 200ms"): Derive the test but flag it as environment-sensitive. Recommend running against a budget, not an exact number, and isolating from noisy neighbors.

## Do not

- **Don't write tests for behavior the requirement doesn't state.** If the spec says "rejects over-limit orders" and nothing else, don't invent a test for "and sends an email" because it seems reasonable. That's gold-plating the spec with your assumptions.
- **Don't collapse the boundary.** `total = 50.00` and `total = 50.01` are different classes when the operator could be `>` vs `>=`. Test both or prove the spec picks one.
- **Don't derive tests from ambiguous requirements.** You'll bake your interpretation into the suite, and six months later someone will "fix" the code to match your interpretation instead of the real intent. Flag ambiguity → `ambiguity-detector`.
- **Don't skip the negative/inverse path** just because the requirement only states the positive. The implicit inverse is where bugs live.
- **Don't emit a single monolithic test** per requirement. One assertion concern per test — so a failure tells you *which* part of the requirement broke.
