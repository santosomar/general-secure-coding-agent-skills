---
name: test-guided-bug-detector
description: Uses failing test results as signals to guide bug search and narrow down candidate fault locations. Use when one or more tests are failing and the user wants to understand what's broken, when CI reports failures, or when triaging a batch of test failures after a change.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "bug-localization, regression-root-cause-analyzer"
---

# Test-Guided Bug Detector

When tests fail, the failure set itself is a signal. One failure tells you where to look; the *pattern* across many failures tells you *what kind* of thing broke.

## Step 1 — Triage by failure shape

| Failure pattern                              | Most likely cause                                     | First move                                      |
| -------------------------------------------- | ----------------------------------------------------- | ----------------------------------------------- |
| One test fails                               | Localized bug in the code that test covers            | Read the assertion; → `bug-localization`        |
| Many tests fail with the **same** error      | Shared dependency broke (fixture, helper, import)     | Find the shared thing — not the individual tests |
| Many tests fail with **different** errors    | Environment/infra (DB down, fixture not loading)      | Check setup/teardown logs, not test bodies      |
| All tests in one **file** fail               | Module-level import/fixture in that file              | Check the file's top-level, not the tests       |
| Tests fail only in **CI**, not locally       | Env difference: version, path, timezone, locale, parallelism | Diff CI env vs local env, not the code     |
| Tests fail only when run **together**        | Test pollution — one test mutates shared state        | Bisect the test *order*; find the polluter      |
| Same tests **intermittently** fail           | Flake — timing, network, randomness                   | Do NOT chase the code — stabilize the test      |

## Step 2 — Spectrum-based fault localization (when many tests fail)

The classic move: code executed by **failing** tests but not by **passing** tests is suspicious.

1. Run with coverage. Record which lines each test hits.
2. For each line, compute suspiciousness (Ochiai): `fail_hits / sqrt(total_fails × (fail_hits + pass_hits))`
3. Sort descending. The top lines are where the fault probably lives.

This is mechanical but surprisingly effective. You need ≥3 failing and ≥3 passing tests for the signal to separate from noise.

## Step 3 — Cluster failures

Before debugging, **group** failures that share a root cause. Debugging 20 failures that are secretly 1 bug is 19× wasted effort.

Cluster by, in order:
- Identical exception type + message → almost certainly same bug
- Identical top-of-stack frame → very likely same bug
- Same file-under-test → likely
- Same fixture used → possible (if the fixture is the bug)

Pick the largest cluster. Fix it. Re-run. Repeat.

## Worked example

**Input:** 47 tests failing after a merge.

**Triage:**
- 41 fail with `KeyError: 'tenant_id'` → same error → **one cluster**
- 5 fail with various assertion mismatches in `test_billing.py` → file-local → **one cluster**
- 1 fails with `ConnectionRefused` → infra → **ignore for now**

**Cluster 1 (41 tests):** All 41 use `@with_authenticated_user` fixture. Fixture source: creates a `User` dict. Grep the diff: `tenant_id` was added as a required field in `User.__init__` but the fixture wasn't updated.

**Root cause:** One line in `conftest.py`. 41 failures → 1 bug.

**Cluster 2 (5 tests):** After fixing cluster 1, re-run. 3 of the 5 now pass (they were also blocked by the fixture). 2 remain. Both assert on a dollar amount that's off by exactly the tax rate. The merge also changed tax calculation.

47 → 2 root causes.

## Edge cases

- **Every single test fails:** Your test runner is broken, not your code. Import error at `conftest.py`/`setup.py` level, or the test DB didn't come up.
- **The failing assertion is `True == True` or similar tautology:** The test itself is broken — pytest collected an accidentally-named non-test function, or someone committed a `assert True # TODO` placeholder.
- **New tests fail, old tests pass:** The new tests might be wrong. Don't assume the product code is at fault until you've read the new test.

## Do not

- **Do not** debug failures one at a time without clustering first. You will fix the same bug five times.
- **Do not** assume the test is right. The test is a *claim*; verify the claim against spec before chasing the code.
- **Do not** re-run a flaky test until it passes and call it fixed. Mark it, quarantine it, move on — but don't lie to yourself.
- **Do not** skip spectrum analysis because it sounds fancy. It's three lines of script and it cuts search time in half.

## Output format

```
## Clusters
1. <N> failures — <shared root: exception/fixture/file>
   Suspected fault: <file:line>  (<how you narrowed it>)
2. ...

## Recommended order
Fix cluster <N> first (<reason: biggest / blocks others / fastest>)

## Quarantine
- <test name>: flaky, <mechanism> — do not chase
```
