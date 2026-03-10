---
name: java-test-updater
description: Updates broken JUnit tests after a deliberate code change — distinguishing tests that broke because the behavior changed (update assertion) from tests that broke because they were overcoupled to structure (loosen or delete). Use after API changes, refactors, or intentional behavior changes leave a trail of failing tests.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "python-test-updater, behavior-preservation-checker"
---

# Java Test Updater

A refactor broke 47 tests. Some of them should break — the behavior changed. Some of them broke because they were mocking internals. Don't bulk-update assertions; **triage first.**

## Triage: why did this test break?

| Failure cause                         | Signal                                              | Action                   |
| ------------------------------------- | --------------------------------------------------- | ------------------------ |
| **Behavior changed (intentional)**    | Assertion on return value fails                     | Update assertion         |
| **API signature changed**             | Compilation error — method renamed, param added     | Update call site         |
| **Over-mocked internals**             | `verify(mock).someInternalMethod(...)` fails        | Delete or loosen mock    |
| **Over-specific assertion**           | `assertEquals("User[id=1, name=Joe]", obj.toString())` | Loosen — assert fields, not string |
| **Test was testing a bug**            | Old assertion was *wrong*; new behavior is correct  | Update + document fix    |
| **Regression (unintentional)**        | Assertion fails, but the change wasn't supposed to affect this | **Stop. This is a real bug.** |

The last row is why you don't auto-update. A failing test might be right.

## Step 1 — Separate compile breaks from assertion breaks

```
mvn test 2>&1 | tee test-output.log
```

Compilation errors (`cannot find symbol`, `method X cannot be applied`) are API drift — mechanical fixes. Assertion failures need judgment.

## Step 2 — Compile fixes (mechanical)

| Change                              | Fix                                                       |
| ----------------------------------- | --------------------------------------------------------- |
| Method renamed                      | IDE rename-propagation, or sed. Grep for old name.        |
| Param added                         | Find default/sentinel. Or: was the old method kept as an overload? Use it. |
| Return type changed                 | Update assignment type. If narrowed, cast may be wrong — check. |
| Class moved/renamed                 | Update imports. Bulk: IDE organize-imports.               |
| Checked exception added             | Add `throws` to test method, or wrap in `assertDoesNotThrow` if the test is about the happy path |

## Step 3 — Assertion triage (judgment)

For each failing assertion, read the **change that caused it:**

```java
// Test:
assertEquals(new BigDecimal("27.80"), invoice.total());
// Fails: actual is 27.55

// Git blame the change:
// commit abc123 "Fix: tax rounds half-even, not half-up"
```

Half-even rounding is a bug fix. The old `27.80` was **wrong**. Update:

```java
// Fixed in abc123: half-even rounding. Was 27.80 (half-up bug).
assertEquals(new BigDecimal("27.55"), invoice.total());
```

Versus:

```java
// Test:
verify(taxCalculator).calc(eq(subtotal), eq(Region.US));
// Fails: calc was called with (subtotal, "US") — String, not enum

// Change: TaxCalculator.calc now takes String region, not Region enum
```

The test is verifying an internal call signature. It broke because an internal API changed. **This test is over-coupled.** Fix the immediate break, but also: does this `verify` need to exist? Is the call to `taxCalculator` the behavior, or an implementation detail?

## Over-coupling patterns — loosen or delete

| Pattern                                        | Why fragile                                | Loosen to                                  |
| ---------------------------------------------- | ------------------------------------------ | ------------------------------------------ |
| `verify(mock, times(3)).helper()`              | Call count is implementation               | `verify(mock, atLeastOnce()).helper()` or delete |
| `assertEquals("exact string", obj.toString())` | toString is debug output                   | Assert fields: `assertEquals("Joe", obj.name())` |
| `assertThat(list).containsExactly(a, b, c)`    | Order might not be contractual             | `.containsExactlyInAnyOrder(a, b, c)` if order isn't spec'd |
| Mocking classes in the same package            | Tests structure, not behavior              | Use real objects; mock only boundaries     |
| `PowerMock` on static/private                  | Deepest coupling possible                  | Refactor for testability, delete test      |

## The regression trap

```java
@Test void price_bulk_discount() {
    // 100 units → 10% bulk discount
    assertEquals(new BigDecimal("900.00"), engine.price(bulkOrder(100)));
}
// Now fails: actual 1000.00
```

The change was to `TaxCalculator`. Why did bulk discount break? **This is a regression.** Don't update the assertion. The test is doing its job — it caught a bug introduced by an unrelated change.

`git bisect` to find where, → `regression-root-cause-analyzer` to find why.

## Batch update — only after triage

Once you've triaged all 47 and confirmed none are regressions:

```java
// For snapshot/approval tests: one command updates all
// ApprovalTests: delete .received files after review, rename to .approved
// For explicit assertions: no shortcut — update each with its reason
```

There is no safe bulk-update for explicit assertions. Each one asserts something on purpose.

## Do not

- **Do not** bulk-update assertions to "whatever the code does now." That's deleting tests with extra steps.
- **Do not** update a test without understanding why it failed. A regression looks exactly like an intentional change until you check.
- **Do not** loosen every `verify`. Some call verifications *are* the behavior — "sends an email" is a behavior you verify by checking the email client was called.
- **Do not** delete a failing test because you don't understand it. Figure out what it was protecting first.
- **Do not** add `@Disabled` as a fix. Disabled tests are deleted tests that still clutter the file.

## Output format

```
## Failing tests
Total: <N>  Compile: <N>  Assertion: <N>

## Compile fixes (mechanical)
| Test | Error | Fix | Applied |
| ---- | ----- | --- | ------- |

## Assertion triage
| Test | Old expected | New actual | Cause | Classification | Action |
| ---- | ------------ | ---------- | ----- | -------------- | ------ |
(Classification: intentional-change | bug-fix | over-coupled | REGRESSION)

## Regressions found
<tests that are correctly failing — bugs in the change, not the test>

## Over-coupling cleanup
<verify/toString/PowerMock patterns loosened>

## After
Passing: <N>  Updated: <N>  Loosened: <N>  Deleted: <N>  Regression bugs filed: <N>
```
