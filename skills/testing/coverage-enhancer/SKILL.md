---
name: coverage-enhancer
description: Raises test coverage by identifying uncovered code regions, ranking them by risk, and generating targeted tests that hit them — prioritizing branches and conditions over raw line count. Use when coverage is below target, when untested code is blocking a release, or when deciding which tests to write next.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "unit-test-generator, mutation-test-suite-optimizer, test-suite-prioritizer"
---

# Coverage Enhancer

Line coverage is a floor, not a goal. 100% lines with 50% branches means half your `if` conditions never ran the other way. This skill targets the uncovered *decisions*, not just uncovered *lines*.

## Coverage levels — weakest to strongest

| Metric              | What it measures                              | Gaps it misses                                    |
| ------------------- | --------------------------------------------- | ------------------------------------------------- |
| Line                | Each line executed at least once              | `if a or b:` — only one of a/b tested             |
| Branch              | Each if/else arm taken                        | `if a and b:` — never tested a=T, b=F             |
| Condition           | Each boolean sub-expression both T and F      | Masking: short-circuit hides some                 |
| MC/DC               | Each condition independently flips the outcome | Nothing — this is the aviation standard           |
| Mutation            | Tests kill injected bugs                      | → `mutation-test-suite-optimizer`                 |

**Default target:** branch coverage. MC/DC for safety-critical. Mutation for "is this suite actually good."

## Step 1 — Get the gap report

| Ecosystem | Branch coverage command                                       |
| --------- | ------------------------------------------------------------- |
| Python    | `pytest --cov=src --cov-branch --cov-report=term-missing`     |
| Java      | JaCoCo — `mvn jacoco:report`, look at "branches missed"       |
| JS/TS     | `jest --coverage` (Istanbul) — "% Branch" column              |
| Go        | `go test -cover` is line-only; use `-covermode=count` + gocov for branches |
| C/C++     | `gcov -b` / llvm-cov                                          |

Output: file → uncovered lines + uncovered branches. The branches are the interesting part.

## Step 2 — Rank the gaps

Not all uncovered code is equal. Rank by risk × effort:

| Factor                            | Weight | How to measure                                   |
| --------------------------------- | ------ | ------------------------------------------------ |
| Code churn (recently changed)     | High   | `git log --since="3 months ago" --oneline -- <file> | wc -l` |
| Complexity (cyclomatic)           | High   | Radon/SonarQube/etc — complex & untested = risky |
| Error handling path               | High   | `except:`, `if err != nil` — errors hide here    |
| Public API surface                | Medium | Externally reachable                             |
| Auto-generated / boilerplate      | -High  | `__repr__`, getters — low value                  |
| Defensive dead code               | -High  | `if x is None:` that's never None by construction |

**Top of the list:** complex error-handling in recently-churned public APIs. **Bottom:** generated `__eq__` methods.

## Step 3 — Construct the hitting test

For each targeted gap, reverse-engineer the input:

**Uncovered branch:** `pricing.py:47 — if customer.tier == "enterprise" and region == "eu": [TRUE branch never taken]`

**Path to this line:** called from `calculate_price(customer, items)` → `_apply_discounts(customer, subtotal)` → line 47.

**What needs to be true:** `customer.tier == "enterprise"` AND `region == "eu"`. Where does `region` come from? `customer.billing_address.country` mapped through `COUNTRY_TO_REGION`.

**Test:**

```python
def test_enterprise_eu_discount_applied():
    customer = Customer(
        tier="enterprise",
        billing_address=Address(country="DE"),  # DE → eu in COUNTRY_TO_REGION
    )
    price = calculate_price(customer, items=[Item(base=100.0)])
    # Branch was dead because no test had an enterprise customer in the EU.
    # The discount is 15% per the body of the branch.
    assert price == 85.0
```

The assertion isn't `assert True` — it checks what the branch *does*, not just that it ran.

## Step 4 — Uncoverable code

Some gaps shouldn't be closed:

| Pattern                                        | Verdict                                         |
| ---------------------------------------------- | ----------------------------------------------- |
| `if sys.platform == "win32":`                  | Covered on Windows CI. `# pragma: no cover` on other platforms. |
| `raise AssertionError("unreachable")`          | It's unreachable. That's the point. Exclude.    |
| `except MemoryError:`                          | Can't reliably trigger. Document the manual test. |
| Logging, `__repr__`, debug helpers             | Low value. Configurable exclusion.              |
| `if TYPE_CHECKING:`                            | Never runs at runtime. Exclude.                 |

Mark these explicitly (`# pragma: no cover` + a reason). Don't chase 100%.

## Do not

- **Do not** write tests that hit lines without asserting behavior. `call_function(); assert True` raises coverage and tests nothing. Every new test asserts the *effect* of the code it covers.
- **Do not** chase 100% line coverage. Diminishing returns past ~85% — the last 15% is boilerplate, defensive code, and platform-specific paths. Chase branch coverage and mutation score instead.
- **Do not** test auto-generated code. `@dataclass` `__eq__` works. Don't test the language.
- **Do not** ignore *why* a branch was uncovered. Sometimes it's dead code (→ `dead-code-eliminator`). Sometimes it's a condition that's always False by construction — that's a bug or a simplification opportunity, not a test gap.

## Output format

```
## Current coverage
Line: <%>  Branch: <%>  (<tool + config>)

## Gap ranking
| File:line | Type | Risk | Churn | Complexity | Priority |
| --------- | ---- | ---- | ----- | ---------- | -------- |

## Targeted tests
### Gap: <file:line> — <description>
Path condition: <what must be true to reach this>
Test:
<code>
Expected coverage delta: +<N> branches

## Excluded
| File:line | Why uncoverable | Marked with |
| --------- | --------------- | ----------- |

## Projected
After these tests: Line <%> → <%>, Branch <%> → <%>
```
