---
name: unit-test-generator
description: Generates unit tests for a function or class by analyzing branches, boundaries, and error paths — then emits test code in the project's existing framework and style. Covers happy path, edge cases, and failure modes with mocks for external dependencies. Use when writing tests for new code, when backfilling coverage on untested functions, when the user asks to generate tests, or when a coverage report shows specific gaps.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "coverage-enhancer, mocking-test-generator, test-oracle-generator"
---

# Unit Test Generator

Generate tests that fail when the code is wrong and pass when it's right — not tests that pass because they assert what the code currently does. The purpose of a test is to pin *intended* behavior, not *observed* behavior.

## Step 1 — Detect the target framework and conventions

Before writing anything, find an existing test file in the project and match it:

| Check                          | Look for                                                                  |
| ------------------------------ | ------------------------------------------------------------------------- |
| Framework                      | `pytest` / `unittest` (Py), `jest` / `vitest` / `mocha` (JS/TS), `JUnit 4/5` / `TestNG` (Java), `go test` (Go), `xUnit` / `NUnit` (.NET), `RSpec` / `Minitest` (Ruby) |
| Assertion style                | `assert x == y` vs `assertThat(x).isEqualTo(y)` vs `expect(x).toBe(y)` vs `x.should eq(y)` |
| File location / naming         | `tests/test_foo.py`, `__tests__/foo.test.ts`, `src/foo_test.go`, `FooTest.java` |
| Mocking library                | `unittest.mock`, `jest.fn()`, Mockito, `gomock`, Moq, Sinon               |
| Parameterization style         | `@pytest.mark.parametrize`, `test.each`, `@ParameterizedTest`, table-driven |
| Fixture / setup idiom          | `conftest.py`, `beforeEach`, `@BeforeEach`, `TestMain`, `let(:x)`         |

If no tests exist at all, ask which framework — or default to the ecosystem standard (pytest, jest, JUnit 5, `go test`) and say so explicitly in the output.

## Step 2 — Enumerate what to cover

Read the function and extract a coverage checklist. Every row becomes at least one test.

| Category         | How to find them                                          | What to test                               |
| ---------------- | --------------------------------------------------------- | ------------------------------------------ |
| Happy path       | The straight-line execution with typical inputs           | Exactly one test. This is your baseline.   |
| Branches         | Every `if`/`elif`/`else`, `switch`/`match` arm, ternary, short-circuit `&&`/`\|\|`, `?.` | One test per arm that takes that arm       |
| Boundaries       | Comparison operators: `<`, `<=`, `>`, `>=`, `==`; loops with explicit bounds; length/size checks | The boundary value itself, one below, one above |
| Error paths      | `raise`/`throw`, functions documented to throw, error-returning branches | One test per distinct error, asserting the *type* and *message* |
| Input edges      | Per parameter type (see table below)                      | Representative of each class               |
| Side effects     | Writes to anything outside local scope: DB, file, HTTP, global, logger | Assert the effect happened — or was skipped when it should be |

**Per-type input edges:**

| Parameter type  | Must cover                                            |
| --------------- | ----------------------------------------------------- |
| Collection      | `[]`, `[x]`, `[x, y, ...]`; if order matters, also reversed |
| Optional / nullable | Present, absent/`None`/`null`/`undefined`         |
| String          | `""`, whitespace-only, typical, very long if there's a length check anywhere downstream |
| Number          | `0`, negative, positive, boundary of any internal comparison, float-vs-int if the code divides or compares |
| Dict / map      | `{}`, key present, key absent                         |
| Enum / union    | **Every** variant (no skipping "unlikely" ones)       |

Not every cell applies to every function. Prune the ones that provably can't affect behavior — but do it *consciously*, not by default.

## Step 3 — Find the oracle for each test

The oracle is *how you know the expected value is correct*. This is the difference between a test and a tautology.

**Valid oracles,** in order of preference:
1. **A spec, docstring, or requirement** says what the output should be.
2. **A mathematical identity**: `reverse(reverse(xs)) == xs`, `decode(encode(x)) == x`, `sum([]) == 0`.
3. **A worked example you compute by hand** — for `tax(100, 0.08)`, you did the arithmetic yourself and got `8.00`.
4. **An independent reference implementation** — a slow/simple version that's obviously correct.
5. **A known-good snapshot** that a human reviewed once — weakest, use sparingly.

**Invalid oracle:** running the code under test and asserting whatever it returns. That's `assert f(x) == f(x)` with extra steps. If you catch yourself doing this — if you ran the function to find out what to assert — stop. You don't have a test, you have a change detector. Either find a real oracle or hand off to → `test-oracle-generator`.

## Step 4 — Isolate external dependencies

A unit test tests one unit. Anything that crosses a process boundary — network, disk, DB, clock, randomness, environment — gets replaced.

| Dependency        | Default replacement                                | Use the real thing when…                  |
| ----------------- | -------------------------------------------------- | ----------------------------------------- |
| HTTP client       | Mock returning a canned response                   | Never, in a unit test                     |
| Database          | In-memory fake, or mock the repository method      | It's an integration test — different file |
| Clock / `now()`   | Freeze to a fixed instant (freezegun, `jest.useFakeTimers`, `Clock.fixed`) | Never — nondeterministic tests are flaky tests |
| Random            | Seed it, or mock to return a fixed value           | Never                                     |
| Filesystem        | `tmp_path` / `tmpdir` fixture, or in-memory FS     | The path logic itself is the unit under test |
| Env var / config  | Patch for the test's duration                      | Never — leaks between tests               |
| Another class you own | **Don't mock by default** — use the real one  | It's expensive to construct, or you're testing the interaction specifically |

**Mock at the boundary, not in the interior.** If `OrderService` uses `TaxCalculator` uses `TaxRateLookup` uses `HttpClient` — mock the HTTP call, not `TaxCalculator`. Mocking your own internal code couples the test to implementation details; refactoring breaks the test without breaking the behavior.

## Step 5 — Emit, matching house style

**Naming:** `test_<what>_<condition>_<expected>`. `test_parse_rejects_empty_string` beats `test_parse_2`. A failed test's name should tell you what broke without opening the file.

**Structure:** Arrange–Act–Assert, one of each. If you need two Acts, that's two tests.

**Parameterize** only when the assertion logic is identical across cases and the cases differ only in data:

```python
@pytest.mark.parametrize("raw,expected", [
    ("1,2,3",  [1, 2, 3]),
    ("",       []),
    ("  7  ",  [7]),       # whitespace trimmed
    ("1,,3",   [1, 3]),    # empty segments dropped
])
def test_parse_csv_ints(raw, expected):
    assert parse_csv_ints(raw) == expected
```

**Don't parameterize** when different cases need different assertions — you'll end up with `if`/`else` inside the test body, which defeats the point.

## Worked example

**Code under test:**

```python
def apply_discount(price: Decimal, code: str | None, clock=time) -> Decimal:
    if code is None:
        return price
    discount = DISCOUNTS.get(code)
    if discount is None:
        raise InvalidCodeError(code)
    if discount.expires_at < clock.time():
        raise ExpiredCodeError(code)
    return (price * (1 - discount.rate)).quantize(Decimal("0.01"))
```

**Enumeration:**

| Category   | Case                                | Oracle                    |
| ---------- | ----------------------------------- | ------------------------- |
| Happy      | Valid unexpired code applied        | Hand arithmetic           |
| Branch     | `code is None` → return unchanged   | Identity                  |
| Branch     | Code not in `DISCOUNTS`             | Spec: raises `InvalidCodeError` |
| Branch     | Code expired                        | Spec: raises `ExpiredCodeError` |
| Boundary   | `expires_at == clock.time()`        | Spec says `<`, so at-the-instant is still valid |
| Edge       | `price = 0`                         | `0 × anything = 0`        |
| Edge       | `rate = 1.0` (100% off)             | Result is `0.00`          |
| Precision  | `price = 9.99`, `rate = 0.333`      | Hand-compute the quantized result |

**Output** (pytest, matching a project that uses fixtures and `pytest.raises`):

```python
from decimal import Decimal
import pytest

@pytest.fixture
def clock_at():
    class _Fixed:
        def __init__(self, t): self._t = t
        def time(self): return self._t
    return _Fixed

@pytest.fixture
def discounts(monkeypatch):
    table = {}
    monkeypatch.setattr("pricing.DISCOUNTS", table)
    return table

def test_applies_valid_unexpired_discount(discounts, clock_at):
    discounts["SUMMER10"] = Discount(rate=Decimal("0.10"), expires_at=2000)
    result = apply_discount(Decimal("50.00"), "SUMMER10", clock=clock_at(1000))
    assert result == Decimal("45.00")  # 50 × 0.9, hand-computed

def test_returns_price_unchanged_when_code_is_none(clock_at):
    assert apply_discount(Decimal("50.00"), None, clock=clock_at(0)) == Decimal("50.00")

def test_rejects_unknown_code(discounts, clock_at):
    with pytest.raises(InvalidCodeError) as exc:
        apply_discount(Decimal("50.00"), "NOPE", clock=clock_at(0))
    assert "NOPE" in str(exc.value)

def test_rejects_expired_code(discounts, clock_at):
    discounts["OLD"] = Discount(rate=Decimal("0.10"), expires_at=999)
    with pytest.raises(ExpiredCodeError):
        apply_discount(Decimal("50.00"), "OLD", clock=clock_at(1000))

def test_code_at_exact_expiry_instant_is_still_valid(discounts, clock_at):
    # Spec: condition is `<`, not `<=` — expiry at t=1000 is valid at t=1000
    discounts["EDGE"] = Discount(rate=Decimal("0.10"), expires_at=1000)
    result = apply_discount(Decimal("50.00"), "EDGE", clock=clock_at(1000))
    assert result == Decimal("45.00")

def test_zero_price_yields_zero(discounts, clock_at):
    discounts["X"] = Discount(rate=Decimal("0.50"), expires_at=2000)
    assert apply_discount(Decimal("0"), "X", clock=clock_at(0)) == Decimal("0.00")

def test_quantizes_to_two_decimal_places(discounts, clock_at):
    discounts["THIRD"] = Discount(rate=Decimal("0.333"), expires_at=2000)
    # 9.99 × 0.667 = 6.66333 → quantize → 6.66
    assert apply_discount(Decimal("9.99"), "THIRD", clock=clock_at(0)) == Decimal("6.66")
```

Seven tests. Every branch taken, boundary pinned, clock deterministic. No test calls `time.time()`. No test asserts a value obtained by running `apply_discount` first.

## Edge cases

- **Function returns `None` / `void`:** The thing to assert is the *side effect*. Mock the collaborator and assert it was called with the right arguments — or assert the state change directly if it's observable.
- **Private function:** Don't reach in. Test through the public caller. If there *is* no public caller, the function is dead or the design is inside-out — flag it, don't contort the test.
- **Hard-to-construct inputs** (deeply nested objects, many required fields): Build one test-data factory with sensible defaults and `**overrides`, reuse it. Don't inline 30 lines of setup per test — nobody can tell what's significant.
- **Code that catches everything** (`except Exception: log; pass`): You can only test that it *doesn't* raise, which is nearly worthless. Note in output: *"Error path is swallowed at line N — consider narrowing the catch or re-raising so failures are observable."*
- **Randomness or concurrency inside the unit:** Seed the RNG or inject it. If concurrency is load-bearing to the behavior, this isn't a unit test — hand off to → `metamorphic-test-generator` or an integration harness.

## Do not

- **Don't assert on `repr()` / `str()` of objects** unless the string representation *is* the contract. `assert str(result) == "<Foo id=3 ...>"` breaks when someone adds a field.
- **Don't assert mock call counts when you mean to assert arguments.** `mock.assert_called_once()` passes when called with the wrong args. Use `assert_called_once_with(expected)`.
- **Don't share mutable state between tests.** Module-level list that each test appends to → order-dependent failures that only reproduce on CI. Use fixtures with function scope.
- **Don't test the framework.** `assert isinstance(response, Response)` — the web framework guarantees that. Test *your* logic.
- **Don't generate a test you know will be deleted.** If the function is trivially delegating (`def foo(x): return bar(x)`), say so: *"No test generated — `foo` is a passthrough to `bar`. Test `bar` directly."*
