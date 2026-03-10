---
name: java-regression-test-generator
description: Generates JUnit regression tests that lock in current behavior before a refactor, capturing observed outputs as assertions so that any behavioral change trips a test. Use before large refactors, when inheriting untested legacy Java, or when the spec is "whatever it does now."
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "python-regression-test-generator, test-guided-migration-assistant, unit-test-generator"
---

# Java Regression Test Generator

Regression tests don't test correctness — they test **stability**. Run the code, record what it does, assert it keeps doing that. If behavior changes, a test fails, and you decide: bug or intentional?

## When regression tests are the right tool

| Situation                                      | Regression tests?                          |
| ---------------------------------------------- | ------------------------------------------ |
| About to refactor legacy code with no tests    | Yes — lock current behavior first          |
| Behavior is the spec ("match the old system")  | Yes — this *is* the spec                   |
| Code is known-buggy                            | Capture current behavior, but **mark** known-wrong assertions as "tracking, not endorsing" |
| Writing tests for new code                     | No — → `unit-test-generator`. New code needs correctness tests. |

## Step 1 — Pick capture points

Public methods are obvious. Also capture at **seams** — places where the call graph narrows:

```java
public class PricingEngine {
    public Invoice price(Order order) {           // ← capture here (public)
        BigDecimal subtotal = computeSubtotal(order);
        BigDecimal tax = taxCalculator.calc(subtotal, order.region());  // ← and here (seam to collaborator)
        return new Invoice(subtotal, tax, subtotal.add(tax));
    }
    private BigDecimal computeSubtotal(Order o) { ... }   // don't capture — implementation detail
}
```

Capturing `computeSubtotal` directly couples tests to private structure. Capture `price()` and let `computeSubtotal` be covered transitively.

## Step 2 — Generate characterizing inputs

You need inputs that exercise different paths. Sources:

| Source                          | How                                                          |
| ------------------------------- | ------------------------------------------------------------ |
| Production samples              | Capture real inputs (sanitized) from logs/DB                 |
| Boundary analysis               | Empty list, single element, max size, null fields            |
| Coverage-guided                 | Run with JaCoCo, find uncovered branches, craft inputs → `coverage-enhancer` |
| Fixture inference               | Look at existing integration tests, pull fixtures            |

Aim for **branch** coverage of the capture point, not line.

## Step 3 — Capture and emit

Run each input. Record output. Emit as a JUnit 5 test:

```java
// Input: Order with 2 line items, US region, no discount
// Captured: 2024-01-15 against PricingEngine@a3f2c1
@Test
void price_twoItems_usRegion_noDiscount() {
    Order order = new Order(
        List.of(new LineItem("sku-1", 2, new BigDecimal("10.00")),
                new LineItem("sku-2", 1, new BigDecimal("5.50"))),
        Region.US,
        Discount.NONE
    );

    Invoice invoice = engine.price(order);

    assertEquals(new BigDecimal("25.50"), invoice.subtotal());
    assertEquals(new BigDecimal("2.30"), invoice.tax());     // 9% US rate — as observed
    assertEquals(new BigDecimal("27.80"), invoice.total());
}
```

The comment says "as observed" — not "as specified." This is a regression test. If tax changes to `2.55`, the test fails, and you decide: did someone break tax, or did the rate change on purpose?

## Step 4 — Handle non-determinism

Some outputs aren't stable across runs:

| Non-determinism       | Handling                                                      |
| --------------------- | ------------------------------------------------------------- |
| Timestamps            | Inject a fixed `Clock`: `Clock.fixed(Instant.parse("2024-01-15T00:00:00Z"), UTC)` |
| UUIDs / random IDs    | Inject a seeded `Random`, or assert format not value: `assertTrue(id.matches("[0-9a-f]{8}-..."))` |
| `HashMap` iteration order | Convert to sorted list before assertion, or use `assertThat(...).containsExactlyInAnyOrder(...)` |
| Floating point        | `assertEquals(expected, actual, 1e-9)`                        |
| External calls        | Mock (→ `mocking-test-generator`) or record-replay (WireMock) |

## Parameterized tests for many inputs

Fifty tests that differ only in data → one parameterized test:

```java
static Stream<Arguments> regressionCases() {
    return Stream.of(
        arguments(order(items(2, "10.00", 1, "5.50"), US, NONE), invoice("25.50", "2.30", "27.80")),
        arguments(order(items(1, "100.00"),           EU, TEN_PCT), invoice("90.00", "18.00", "108.00")),
        // ... 48 more
    );
}

@ParameterizedTest
@MethodSource("regressionCases")
void price_regression(Order input, Invoice expected) {
    assertEquals(expected, engine.price(input));
}
```

For truly large case counts, load from a resource file (CSV/JSON) — keeps the test class readable.

## Known-wrong behavior

Sometimes current behavior is a bug. Still capture it, but **flag it:**

```java
@Test
// TODO(bug-1234): This asserts the CURRENT buggy behavior — tax is computed
// on post-discount subtotal, spec says pre-discount. Tracking, not endorsing.
// When fixed, expected tax changes from 8.10 → 9.00.
void price_discount_taxOnWrongBase_TRACKING_BUG() {
    ...
    assertEquals(new BigDecimal("8.10"), invoice.tax());  // wrong, but current
}
```

When the bug is fixed, this test fails — good, that's the signal. Update the assertion and delete the comment.

## Do not

- **Do not** generate regression tests for code you're about to delete. Waste of time.
- **Do not** assert on `toString()` output unless `toString()` is the contract. Refactoring a debug string shouldn't break tests.
- **Do not** capture internal state via reflection (`setAccessible(true)`). That's not behavior — it's structure. Capture observable outputs.
- **Do not** forget to pin the clock. A test that passes in January and fails in February is not a regression test, it's a time bomb.
- **Do not** treat regression tests as permanent. Once you've refactored and written *real* tests with *intentional* assertions, delete the regression tests that are now redundant.

## Output format

```
## Capture target
<class/method(s) — why these seams>

## Input set
| # | Input summary | Branches hit | Source |
| - | ------------- | ------------ | ------ |

## Non-determinism handling
| Field | Strategy |
| ----- | -------- |

## Generated tests
<JUnit 5 — @Test or @ParameterizedTest + @MethodSource>

## Known-wrong captures
| Test | Bug ref | Expected change when fixed |
| ---- | ------- | -------------------------- |

## Coverage
Before: <%>  After: <%>  (branch)
```
