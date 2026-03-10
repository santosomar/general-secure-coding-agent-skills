---
name: python-test-updater
description: Updates broken pytest tests after intentional code changes — triaging assertion failures from mock-coupling failures from genuine regressions, using Python's introspection to automate where safe. Use when a refactor or API change leaves a pile of failing tests and you need to decide update vs. fix vs. delete.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "java-test-updater, behavior-preservation-checker"
---

# Python Test Updater

Same triage discipline as → `java-test-updater`. Python differences: no compile-time breaks (everything fails at runtime), more `mocker.patch` coupling, and snapshot libraries make some updates one command.

## Python failure taxonomy

No compile step means everything surfaces at test runtime:

| Failure                                       | Python-specific signal                          | Action                   |
| --------------------------------------------- | ----------------------------------------------- | ------------------------ |
| `AttributeError: 'X' has no attribute 'foo'`  | Renamed/removed method                          | Update call site         |
| `TypeError: f() missing 1 positional argument` | Signature changed                              | Add arg or use default   |
| `TypeError: f() got an unexpected keyword`    | Kwarg renamed/removed                           | Update kwarg             |
| `ImportError` / `ModuleNotFoundError`         | Module moved                                    | Update import            |
| `AssertionError` with value diff              | Behavior changed (intentional?) or regression   | **Triage**               |
| `AssertionError: Expected 'mock' to be called` | Over-mocked internal                           | Loosen/delete mock       |
| `AssertionError` in snapshot compare          | Snapshot stale                                  | Review diff, `--snapshot-update` if intentional |

## Automating the mechanical fixes

Python's runtime errors point straight at the problem. For signature changes across many tests:

```python
# conftest.py — one-time shim during migration
# OLD: Order(items, region)   NEW: Order(items, region, currency="USD")
# Tests still call Order(items, region).  Temporary compat:

@pytest.fixture(autouse=True)
def _order_compat(mocker):
    orig_init = Order.__init__
    def compat_init(self, items, region, currency="USD"):
        orig_init(self, items, region, currency)
    mocker.patch.object(Order, "__init__", compat_init)
```

This is a **bridge**, not a fix. All tests pass → remove the shim → fix tests one by one as you touch them. Don't leave compat shims in permanently.

## Mock coupling — the `mocker.patch` problem

```python
def test_process(mocker):
    mock_validate = mocker.patch("orders.service._validate_order")
    mock_save = mocker.patch("orders.service._save_order")
    process(order)
    mock_validate.assert_called_once_with(order)
    mock_save.assert_called_once()
```

`_validate_order` was inlined into `process`. Test fails: `Expected '_validate_order' to have been called once. Called 0 times.`

The behavior didn't change — the order is still validated — but the test was asserting *structure*. **Delete the mock assertion.** The test should assert the *outcome* of validation:

```python
def test_process_rejects_invalid():
    bad = Order(items=[])
    with pytest.raises(InvalidOrder, match="empty"):
        process(bad)
```

This survives refactors because it tests *what* validation does, not *that a function named `_validate_order` was called.*

## Snapshot updates — review first

If using `syrupy` / `pytest-snapshot`:

```bash
pytest --snapshot-update
```

This updates **all** failing snapshots to current output. Dangerous if any failure is a regression. Workflow:

1. Run without `--snapshot-update`. Read every diff.
2. For each: intentional change? Or regression?
3. Only after confirming all are intentional: `--snapshot-update`.
4. Commit the snapshot changes with a message explaining *why* they changed.

## Assertion triage — same as Java

```python
assert invoice.total == Decimal("27.80")   # fails: actual 27.55
```

`git log -p -- src/pricing.py` → "Fix: half-even rounding." Intentional. Update with reason:

```python
# abc123: half-even rounding fix. Was 27.80 (half-up bug).
assert invoice.total == Decimal("27.55")
```

Versus:

```python
assert len(results) == 5   # fails: actual 4
```

The change was to a completely different module. Why are there fewer results? **Regression.** Don't update. Investigate.

## `pytest.approx` drift

```python
assert score == 0.8472819  # now fails: 0.8472820
```

Last digit changed — floating-point ops reordered. If precision to 7 decimals isn't spec'd, this was over-tight:

```python
assert score == pytest.approx(0.8473, rel=1e-4)
```

This isn't "loosening to make it pass." It's fixing an over-specific assertion that should never have been that tight.

## Do not

- **Do not** blindly run `--snapshot-update`. Review diffs first. A regression in a snapshot looks identical to an intentional change.
- **Do not** fix `mocker.patch("module._private")` failures by updating the patch path. You're chasing implementation. Replace with behavioral assertions.
- **Do not** widen `pytest.approx` tolerance until the test passes. If `rel=0.5` is what it takes, the test isn't testing anything.
- **Do not** skip investigating assertion failures in "unrelated" tests. Unrelated is where regressions hide.
- **Do not** leave compat shims in `conftest.py` after the migration. They mask further API drift.

## Output format

```
## Failing tests
Total: <N>  Import/Attribute: <N>  Signature: <N>  Assertion: <N>  Mock: <N>  Snapshot: <N>

## Mechanical fixes
| Test | Error | Fix |
| ---- | ----- | --- |

## Mock decoupling
| Test | Over-coupled patch | Replacement assertion |
| ---- | ------------------ | --------------------- |

## Assertion triage
| Test | Old | New | Cause commit | Classification | Action |
| ---- | --- | --- | ------------ | -------------- | ------ |

## Snapshot review
| Snapshot | Diff summary | Intentional? |
| -------- | ------------ | ------------ |

## Regressions
<tests correctly failing — file bugs, don't update>

## After
Passing: <N>  Updated: <N>  Decoupled: <N>  Deleted: <N>  Bugs filed: <N>
```
