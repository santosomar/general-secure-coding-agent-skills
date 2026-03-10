---
name: python-regression-test-generator
description: Generates pytest regression tests that capture current behavior as snapshot assertions, using Python's dynamism for low-friction recording. Use before refactoring untested Python, when the behavioral spec is "whatever it does now," or when migrating Python 2→3 or between framework versions.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "java-regression-test-generator, test-guided-migration-assistant, unit-test-generator"
---

# Python Regression Test Generator

Same idea as → `java-regression-test-generator` — capture current behavior, lock it in — but Python's tooling makes capture easier: pickle snapshots, `syrupy`/`pytest-snapshot`, and duck typing means fewer fixture gymnastics.

## Capture approaches by effort

| Approach                     | Effort | When                                         |
| ---------------------------- | ------ | -------------------------------------------- |
| Snapshot library (`syrupy`)  | Lowest | Outputs are dicts/lists/strings — most cases |
| Pickle round-trip            | Low    | Complex custom objects with sane `__eq__`    |
| Explicit field assertions    | Medium | You want readable diffs when things break    |
| Golden files                 | Medium | Large outputs (rendered HTML, CSVs)          |

**Default to snapshot libraries** for the bulk, **explicit assertions** for the handful of truly critical paths.

## Snapshot-based capture

```python
# First run with --snapshot-update: records current output
# Subsequent runs: compares against recorded
def test_price_two_items_us(snapshot):
    order = Order(
        items=[LineItem("sku-1", 2, Decimal("10.00")),
               LineItem("sku-2", 1, Decimal("5.50"))],
        region="US",
        discount=None,
    )
    assert engine.price(order) == snapshot
```

Snapshot lives in `__snapshots__/test_pricing.ambr`. When behavior changes, diff is shown. Accept with `--snapshot-update` or fix the regression.

## Explicit assertions — when readability matters

For the few paths you really care about, write it out:

```python
def test_price_enterprise_eu_discount():
    """
    Captured 2024-01-15. The EU enterprise discount is 15% — as observed,
    no spec found. If this breaks, verify with product before updating.
    """
    order = Order(
        items=[LineItem("sku", 1, Decimal("100.00"))],
        region="EU",
        tier="enterprise",
    )
    invoice = engine.price(order)

    assert invoice.subtotal == Decimal("100.00")
    assert invoice.discount == Decimal("15.00")   # 15% — observed, not spec'd
    assert invoice.tax == Decimal("17.00")        # 20% VAT on post-discount 85
    assert invoice.total == Decimal("102.00")
```

Explicit fields → readable failure: `assert Decimal('17.00') == Decimal('20.00')` tells you exactly what changed.

## Input generation via tracing

Python lets you record real inputs cheaply:

```python
# Wrap the real function — deploy briefly to staging, collect inputs
_captured = []

def capturing_price(self, order):
    result = _original_price(self, order)
    _captured.append((copy.deepcopy(order), copy.deepcopy(result)))
    return result

PricingEngine.price = capturing_price

# ... run realistic traffic ...

# Dump
with open("regression_cases.pkl", "wb") as f:
    pickle.dump(_captured, f)
```

Then generate:

```python
with open("regression_cases.pkl", "rb") as f:
    CASES = pickle.load(f)

@pytest.mark.parametrize("order,expected", CASES, ids=lambda c: repr(c)[:60])
def test_price_regression(order, expected):
    assert engine.price(order) == expected
```

Real traffic → realistic edge cases you wouldn't think of.

## Non-determinism — Python specifics

| Issue                          | Fix                                                       |
| ------------------------------ | --------------------------------------------------------- |
| `datetime.now()`               | `freezegun.freeze_time("2024-01-15")` decorator           |
| `time.time()`                  | Same — freezegun patches both                             |
| `uuid.uuid4()`                 | `mocker.patch("uuid.uuid4", side_effect=[UUID("..."), ...])` or assert format |
| `random`                       | `random.seed(0)` at test start, or inject `Random(0)`     |
| `dict` order                   | Guaranteed insertion-ordered since 3.7 — only a problem if code uses `set` or pre-3.7 |
| `set` iteration                | `assert sorted(result) == sorted(expected)`               |
| Float precision                | `pytest.approx(expected, rel=1e-9)`                       |
| `PYTHONHASHSEED`               | Set to fixed value in conftest if hashing user strings    |

## Known-wrong capture

```python
@pytest.mark.xfail(
    reason="Bug #1234: tax computed on post-discount subtotal. "
           "Spec says pre-discount. Expected 9.00, currently 8.10. "
           "When fixed, flip strict=True → test must pass.",
    strict=False,
)
def test_discount_tax_base_KNOWN_BUG():
    ...
    assert invoice.tax == Decimal("9.00")   # the CORRECT value
```

`xfail(strict=False)` — currently fails, that's expected. When the bug is fixed, test passes → xpass is reported. Flip to `strict=True` and it becomes a normal passing test.

Alternative: assert the *wrong* value with a `# TRACKING BUG` comment. Less elegant (xfail is nicer) but works without xfail semantics.

## Do not

- **Do not** pickle objects with open file handles, sockets, or lambdas in them — unpickleable. Use explicit field extraction for those.
- **Do not** snapshot-test functions with hundreds of call sites and one snapshot each. Review burden when things change is enormous. Parametrize and group.
- **Do not** forget to sanitize captured production inputs. Real `Order` objects might have real customer emails in them.
- **Do not** capture outputs that include absolute paths, hostnames, or PIDs. Normalize (`path.relative_to(root)`, `socket.gethostname()` → `"<host>"`) before snapshotting.
- **Do not** leave capture wrappers in production. Monkey-patching `price` for an hour in staging: fine. Forgetting to remove it: not fine.

## Output format

```
## Capture target
<module.function(s)>

## Input set
| # | Input summary | Source | Branches hit |
| - | ------------- | ------ | ------------ |

## Capture approach
<snapshot | explicit | pickle-parametrize | golden>

## Non-determinism
| Source | Fix |
| ------ | --- |

## Generated tests
<pytest code>

## Known-wrong captures
| Test | Bug ref | xfail or tracking-comment |
| ---- | ------- | ------------------------- |

## Coverage delta
Before: <%>  After: <%>  (branch, via `pytest --cov --cov-branch`)
```
