---
name: test-case-documentation
description: Writes documentation for test cases — names, docstrings, and comments that explain what behavior is being tested and why, so a failing test tells you what broke without reading the assertion. Use when test names are test_1 through test_47, when tests fail and nobody knows what they mean, or when onboarding needs a readable test suite.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "code-comment-generator"
---

# Test Case Documentation

A test named `test_process_3` failed. What broke? Nobody knows without reading 20 lines of setup. The **test name is the first line of the failure message** — make it count.

## The test name is the spec

| Bad name                  | Why bad                                    | Good name                                        |
| ------------------------- | ------------------------------------------ | ------------------------------------------------ |
| `test_1`                  | Says nothing                               | `test_empty_cart_has_zero_total`                 |
| `test_parse`              | Which parse case?                          | `test_parse_rejects_trailing_comma`              |
| `test_auth_error`         | Which error? Which auth?                   | `test_expired_session_returns_401`               |
| `test_edge_case`          | Which edge?                                | `test_discount_caps_at_100_percent`              |
| `testProcessOrderSuccess` | "Success" — but what *is* success here?    | `testProcessOrder_reservesInventory_thenCharges` |

**Naming pattern:** `test_<scenario>_<expected outcome>`. The scenario is the input/state; the outcome is what you assert.

## The rename test (from `code-comment-generator`)

Before writing a docstring, try renaming the test. If the name says it all, no docstring needed:

```python
def test_3():
    """Tests that refund on a cancelled order returns the full amount."""
    ...
```

→

```python
def test_cancelled_order_refund_is_full_amount():
    ...
```

Docstring deleted. The name *is* the doc.

## When a docstring earns its place

The name says *what*. A docstring says *why* — when "why" isn't obvious:

```python
def test_retry_gives_up_after_5_attempts_not_3():
    """
    Upstream SLA guarantees availability within 5 retries at exponential
    backoff (1s, 2s, 4s, 8s, 16s = 31s total). We previously stopped at 3
    (bug #4812) which gave up during legitimate upstream brownouts.
    """
    ...
```

The name says what (5 not 3). The docstring says why (SLA math, bug ref). Both pull their weight.

| Docstring is worth it when…                         | Example                                           |
| --------------------------------------------------- | ------------------------------------------------- |
| The assertion value has non-obvious origin          | "5 retries because SLA says 31s max wait"         |
| It's a regression test for a specific bug           | "Regression for #4812"                            |
| The setup is unusual for a non-obvious reason       | "Mock returns 503 twice — simulates the Feb outage" |
| The test asserts *absence* of behavior              | "Verifies NO email is sent — silent failure is intentional here" |

## Worked example — before/after

**Before:**

```python
def test_5(self):
    u = User(tier="gold")
    o = Order(items=[Item(100)], user=u)
    o.created_at = datetime(2024, 11, 29)   # black friday
    r = process(o)
    self.assertEqual(r.total, 70)
```

What's 70? Why Black Friday? Why gold?

**After:**

```python
def test_gold_tier_stacks_with_black_friday_discount(self):
    """
    Gold tier (20% off) stacks multiplicatively with Black Friday (12.5% off):
    100 × 0.80 × 0.875 = 70.  Not additive (100 - 20 - 12.5 = 67.5) —
    confirmed with product in #pricing-decisions 2024-10.
    """
    user = User(tier="gold")
    order = Order(items=[Item(price=100)], user=user)
    order.created_at = datetime(2024, 11, 29)   # Black Friday 2024

    result = process(order)

    self.assertEqual(result.total, 70)   # 100 × 0.80 × 0.875
```

Now when this fails with `70 != 67.5`, you immediately know: someone changed stacking from multiplicative to additive. The docstring tells you that was a *deliberate* product decision.

## Inline comments in tests

Three places inline comments help:

```python
def test_concurrent_writes_last_write_wins():
    # Arrange — both clients see version 1
    v1 = store.get("key")
    client_a = Client(initial=v1)
    client_b = Client(initial=v1)

    # Act — A writes, then B writes (B doesn't see A's write)
    client_a.set("key", "from_a")
    client_b.set("key", "from_b")   # B's base is still v1, not A's write

    # Assert — LWW, not conflict error (per ADR-007)
    assert store.get("key") == "from_b"
```

Arrange/Act/Assert markers are fine. The useful comment is `B's base is still v1` — *that's* the scenario. Without it, a reader might think B saw A's write.

## Do not

- **Do not** write docstrings that restate the test name. `test_parse_empty` with docstring "Tests parsing an empty string" — redundant.
- **Do not** document the assertion in the docstring. `"""Should return 70."""` — the `assertEqual(70, ...)` already says that.
- **Do not** leave magic numbers unexplained. `assertEqual(r.total, 70)` with no clue where 70 comes from is the #1 cause of "I don't know if this failure is a bug."
- **Do not** use `# noqa` or `# type: ignore` without a reason comment. `# type: ignore[arg-type]  — mock's type stub is wrong, actual runtime accepts str` — now the suppression is justified.
- **Do not** document *how* the test works internally. Document *what behavior it protects.*

## Output format

```
## Tests reviewed
| File | Tests | Renamed | Docstring added | Magic numbers explained |
| ---- | ----- | ------- | --------------- | ----------------------- |

## Renames
| Before | After | Why the old name was insufficient |
| ------ | ----- | --------------------------------- |

## Docstrings added
| Test | Docstring (summary) | What it explains beyond the name |
| ---- | ------------------- | -------------------------------- |

## Magic numbers
| Test | Value | Explanation added |
| ---- | ----- | ----------------- |

## Tests that should be split
<tests where naming revealed they test two things — note, don't fix here>
```
