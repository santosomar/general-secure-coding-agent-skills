---
name: bug-reproduction-test-generator
description: Creates minimal, reproducible test cases from bug reports to confirm the defect before and after a fix. Use when a bug is reported without a failing test, when the user needs a regression test for a fix, or when the user asks to reproduce a bug as a test.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "bug-localization, test-case-reducer, bug-to-patch-generator"
---

# Bug Reproduction Test Generator

A bug you can't reliably reproduce is a bug you can't reliably fix. This skill turns prose ("it crashes when I upload a big file") into an executable test that is **red** on the buggy code and will be **green** once it's fixed.

## Step 1 — Classify the input you have

| Input available                           | First move                                              |
| ----------------------------------------- | ------------------------------------------------------- |
| Stack trace                               | Top project-code frame → target function; exception → assertion |
| "Steps to reproduce" prose                | Translate each step to a setup line; last step → action |
| Log excerpt                               | Find the first anomalous line; grep for the producer    |
| Failing production request (curl, HAR)    | Replay against a test harness; wrap in an assertion     |
| Screenshot / "it looks wrong"             | Not machine-checkable — ask for the *data*, not the UI  |
| "It's intermittent"                       | Do not write a test yet — first stabilize (Step 4)      |

## Step 2 — Identify the triple

Every reproduction test is `(setup, action, assertion)`. Extract each:

- **Setup:** the minimal state needed. A user? One user, not a seeded database. A file? The smallest file that triggers it.
- **Action:** the single call/request/event that triggers the bug. If the bug needs two actions in sequence, both are the action.
- **Assertion:** what *should* happen, not what currently happens. The test must describe correct behavior, because once fixed this test is your regression guard.

## Step 3 — Write, then minimize

Write a test that reproduces. It will be too big. Minimize **mechanically**:

| Dimension    | Minimization                                                               |
| ------------ | -------------------------------------------------------------------------- |
| Setup state  | Delete one setup line → still red? Keep deleting. Add back when it goes green. |
| Input size   | Binary-bisect the input (half the file, half the list) until it goes green |
| Dependencies | Inline mocks; if removing a mock turns it green, that mock was load-bearing |

The target is: a test so small that when it fails, the fault is obvious from reading the test alone.

## Step 4 — Stabilize intermittents

If the bug is non-deterministic, the test can't simply assert correct behavior — it'll pass by luck half the time.

| Source of flakiness   | Stabilization                                                 |
| --------------------- | ------------------------------------------------------------- |
| Timing / sleep-based  | Replace wall-clock with an injected clock you advance manually |
| Thread interleaving   | Force the bad interleaving with a latch/barrier; assert on the state you now deterministically reach |
| Network               | Mock the transport; inject the specific response/error that triggers the bug |
| Test order            | Run the test in isolation; if it passes alone, the bug is test pollution, not product code |
| Randomness            | Seed the RNG with the seed that reproduces (if logged); if not, loop 1000× and assert all pass |

If you cannot stabilize, do **not** ship a test that retries. A retry loop in a reproduction test is an admission you don't understand the bug.

## Worked example

**Report:** "Exporting a CSV with a quote in a column value produces a file Excel can't open."

**Triple extraction:**
- Setup: one row with one field containing a double-quote
- Action: call `export_csv(rows)`
- Assertion: output field is RFC-4180 escaped (`"` → `""` and field is quoted)

**Minimal test:**

```python
def test_csv_export_escapes_quotes():
    rows = [{"name": 'say "hi"'}]
    out = export_csv(rows)
    assert out == 'name\r\n"say ""hi"""\r\n'
```

Three lines of meaning. When this fails, the reader immediately sees: quotes aren't escaped.

## Edge cases

- **The bug is a hang, not a crash:** Assertion is `pytest.fail()` *after* the call, with a timeout decorator. The test passes if it reaches `fail()` within the timeout.
- **The bug is a leak:** Can't assert on memory from a unit test. Instead, assert that `close()`/`__exit__` was called on the resource mock.
- **The correct behavior is undefined:** Ask. A test must assert *something*; if nobody knows what "correct" is, you found a requirements bug — → `requirement-enhancer`.

## Do not

- **Do not** assert on the current (buggy) behavior. The test should describe what *should* happen and must currently *fail*.
- **Do not** reproduce via the UI when you can reproduce via the API. UI tests are 10× slower and 10× flakier.
- **Do not** add this test to the "flaky" quarantine. If it's flaky, you haven't finished reproducing.
- **Do not** skip minimization because "the test works." A 50-line reproduction nobody can read is a 50-line liability.

## Handoff

Once red: → `bug-localization` (test is the signal) → `bug-to-patch-generator` (test is the oracle).
