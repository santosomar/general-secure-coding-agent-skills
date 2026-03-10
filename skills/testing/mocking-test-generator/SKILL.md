---
name: mocking-test-generator
description: Generates tests that mock external dependencies — HTTP, databases, filesystems, clocks — isolating the unit under test while still exercising realistic interactions. Use when the code has side effects you can't run in a test, when external services are slow or unavailable, or when testing error paths that are hard to trigger for real.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "unit-test-generator, test-oracle-generator"
---

# Mocking Test Generator

Mocks replace real dependencies with controllable fakes. Done right, they isolate the unit and let you test error paths. Done wrong, they test the mock instead of the code.

## When to mock

| Dependency                   | Mock?                                           | Why                                         |
| ---------------------------- | ----------------------------------------------- | ------------------------------------------- |
| External HTTP API            | Yes                                             | Slow, flaky, costs money, rate limits       |
| Database                     | Usually yes (unit), no (integration)            | Real DB in unit tests = slow + shared state |
| Filesystem                   | Yes, or use a temp dir                          | Test pollution, permission issues           |
| Clock / time                 | Yes                                             | `sleep(3600)` in a test is not a test       |
| Random number generator      | Yes (seed it)                                   | Reproducibility                             |
| Your own pure functions      | **No**                                          | That's what you're testing                  |
| Your own classes (same module) | Rarely                                        | Over-mocking tests wiring, not behavior     |

**Rule of thumb:** mock at the **process boundary** (network, disk, clock). Don't mock your own code.

## Mock types — pick the right one

| Type          | What it does                                  | Use for                                    |
| ------------- | --------------------------------------------- | ------------------------------------------ |
| **Stub**      | Returns canned values                         | "When I call X, give me Y" — most tests    |
| **Mock**      | Stub + records calls for assertion            | "Verify X was called with args A, B"       |
| **Fake**      | Working implementation, simplified (in-memory DB) | Integration-ish tests, stateful interactions |
| **Spy**       | Real object + call recording                  | "Let it really happen, but verify it did"  |

**Default to stubs.** Mocks (with call-count assertions) couple the test to *how* the code does something, not *what* it does. Brittle.

## Worked example — mocking HTTP + clock

**Code under test:**

```python
def fetch_with_retry(url, max_attempts=3):
    for attempt in range(max_attempts):
        resp = requests.get(url, timeout=5)
        if resp.status_code == 200:
            return resp.json()
        time.sleep(2 ** attempt)
    raise FetchError(f"{max_attempts} attempts failed")
```

**What to mock:** `requests.get` (network) and `time.sleep` (clock).

**Test — happy path:**

```python
def test_fetch_succeeds_first_try(mocker):
    mock_get = mocker.patch("requests.get")
    mock_get.return_value.status_code = 200
    mock_get.return_value.json.return_value = {"data": 42}

    result = fetch_with_retry("http://example.com")

    assert result == {"data": 42}
    assert mock_get.call_count == 1   # didn't retry
```

**Test — retry then succeed:**

```python
def test_fetch_retries_on_500(mocker):
    mocker.patch("time.sleep")   # don't actually sleep
    mock_get = mocker.patch("requests.get")
    # Two 500s, then a 200:
    r500 = mocker.Mock(status_code=500)
    r200 = mocker.Mock(status_code=200)
    r200.json.return_value = {"data": 42}
    mock_get.side_effect = [r500, r500, r200]

    result = fetch_with_retry("http://example.com")

    assert result == {"data": 42}
    assert mock_get.call_count == 3
```

**Test — exhausted retries:**

```python
def test_fetch_raises_after_max_attempts(mocker):
    mocker.patch("time.sleep")
    mock_get = mocker.patch("requests.get")
    mock_get.return_value.status_code = 500

    with pytest.raises(FetchError, match="3 attempts"):
        fetch_with_retry("http://example.com", max_attempts=3)

    assert mock_get.call_count == 3
```

The error path is the **hardest to trigger for real** and the **easiest to test with mocks**. That's where mocking earns its keep.

## The over-mocking smell

```python
def test_process_order(mocker):
    mocker.patch.object(Order, "validate")
    mocker.patch.object(Order, "calculate_total")
    mocker.patch.object(Order, "save")
    mocker.patch.object(Inventory, "reserve")
    mocker.patch.object(Notifier, "send")

    process_order(order)

    Order.validate.assert_called_once()
    Order.calculate_total.assert_called_once()
    # ...
```

This tests that `process_order` *calls five things in order*. It doesn't test that the order is processed correctly. Change the implementation (merge two methods, reorder calls) → test breaks, but nothing's wrong.

**Fix:** mock only `Inventory.reserve` and `Notifier.send` (external side effects). Let `Order`'s methods actually run — they're your code.

## Mocking the clock

Time is the sneakiest dependency. Two approaches:

| Approach                            | How                                                      |
| ----------------------------------- | -------------------------------------------------------- |
| Inject clock                        | `def __init__(self, clock=time.time)` → pass `lambda: fake_now` in tests |
| Patch globally                      | `freezegun.freeze_time("2025-01-01")` — monkey-patches `datetime`/`time` everywhere |

Injection is cleaner (explicit dependency). Freezegun is easier for existing code that calls `datetime.now()` inline.

## Do not

- **Do not** mock what you're testing. If you're testing `parse()`, don't mock `parse()`'s helpers — mock its I/O.
- **Do not** assert on call counts unless the count is the *behavior*. "Retries 3 times" is a behavior — assert it. "Internally calls `_helper` twice" is implementation — don't.
- **Do not** let mocks drift from reality. The real API returns `{"status": "ok"}`; your mock returns `{"ok": true}`. Tests pass, prod breaks. Verify mocks against a real response periodically (contract tests).
- **Do not** mock the database in integration tests. The point of integration is to integrate.
- **Do not** forget to mock `time.sleep` in retry tests. A test that sleeps 7 seconds is a test nobody runs.

## Output format

```
## Code under test
<function/class>

## Dependencies
| Dependency | Type | Mock? | Why |
| ---------- | ---- | ----- | --- |

## Tests
### <test name>
Mocks: <what's patched>
Scenario: <what path — happy, retry, error>
<code>

## Over-mocking check
<any mocks that patch your own code — justify or remove>

## Contract risk
<mocks that mirror external APIs — how you verify they match reality>
```
