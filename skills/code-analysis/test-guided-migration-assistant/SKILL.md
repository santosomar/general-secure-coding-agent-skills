---
name: test-guided-migration-assistant
description: Uses an existing test suite as the behavioral oracle during a migration, tracking which tests pass at each step and localizing regressions to specific migration changes. Use when porting or refactoring code that has tests, when the user wants to migrate incrementally with a safety net, or when a migration broke something and you need to find which step did it.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "module-level-code-translator, behavior-preservation-checker, regression-root-cause-analyzer"
---

# Test-Guided Migration Assistant

The test suite is a **specification in executable form.** During a migration, it's the oracle: if the tests pass, the migration preserved behavior (as far as the tests know).

## The invariant

At every step of the migration: **tests pass.** If a step breaks a test, either:
- the step was wrong → revert and redo, or
- the test was wrong (tested implementation, not behavior) → fix the test *first, in a separate commit*, then redo the step.

Never proceed with a red test. Red accumulates.

## Step 0 — Assess the suite

Before migrating, know what you're working with:

| Metric                     | Command                                              | Threshold                        |
| -------------------------- | ---------------------------------------------------- | -------------------------------- |
| Line coverage              | `pytest --cov` / `jacoco` / etc                      | > 70% for comfort                |
| Branch coverage            | Same tools, `--cov-branch`                           | > 60% for comfort                |
| Test speed                 | How long does the full suite take?                   | < 5 min → run after every step. > 30 min → you'll need prioritization (→ `test-suite-prioritizer`) |
| Flakes                     | Run the suite 3× — do the same tests pass each time? | Zero flakes, or quarantine them  |

**If coverage is low:** the tests are a weak oracle. Migration steps that touch uncovered code are flying blind. Either write characterization tests for the gaps first, or flag those steps as high-risk.

## Step 1 — Baseline

Before touching anything:
1. Run the full suite. Record which tests pass.
2. **Any failures now are not your problem.** Note them, skip them (`@pytest.mark.skip(reason="pre-existing — issue #123")`). Start green.
3. Record coverage. Uncovered regions are migration blind spots.

## Step 2 — Migrate in test-sized increments

Pick migration steps small enough that **at most a handful of tests could plausibly be affected.** One function, one class, one config block.

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Step N   │────►│ Run tests│────►│  Green?  │
│ (small)  │     │ (fast    │     │    │     │
└──────────┘     │  subset) │     │  Y │ N   │
                 └──────────┘     └──┬─┴──┬──┘
                                     │    │
                            ┌────────▼┐  ┌▼────────────────┐
                            │ Commit  │  │ Bisect: which   │
                            │ Step N+1│  │ part of step N? │
                            └─────────┘  └─────────────────┘
```

**Fast subset:** don't run the whole suite after every step. Run the tests that cover the code you touched (`pytest --testmon`, or manually: `pytest tests/test_foo.py` if you changed `foo.py`). Full suite at milestones.

## Step 3 — When a test fails

Don't guess. Localize:

1. **Which step broke it?** If you've been committing green, `git bisect` finds the bad step mechanically.
2. **Within that step, which change?** If the step touched 5 things, revert 4 and re-run. Binary search.
3. **Is the test right?** Read the test. Is it asserting *behavior* (output for input) or *implementation* (which methods get called, internal state)?
   - Behavior test failing → your migration broke behavior. Fix the migration.
   - Implementation test failing → the test was coupled to the old implementation. Fix the test — **in a separate commit, before retrying the step.** Document why.

## Worked example — Python 2 → 3 migration

**Suite:** 340 tests, 78% line coverage, 4 min full run.

**Baseline:** 337 pass, 3 fail (pre-existing — skipped with issue refs). Green.

**Step 1: `__future__` imports everywhere.** → 337 pass. Green, commit.

**Step 2: `dict.iteritems()` → `.items()`.** → 337 pass. Green, commit.

**Step 3: `/` → `//` for integer division (12 sites).** → **331 pass, 6 fail.**

Bisect within step 3: revert 6 sites, keep 6 → still fails. Revert 3 more → passes. Narrowed to 3 sites. One at a time:

- `pagination.py:42` → `total / per_page` was float in Py2 for `total=10, per_page=3` → `3.33`, and the code did `ceil()` on it. Changing to `//` gives `3`, not `3.33`, and `ceil(3)` is `3` not `4`. **Migration error.** Should be `total / per_page` (true division — the Py2 code was *already* using float division via `from __future__ import division`, which I missed). Revert this site.

**Lesson:** the test caught a real behavior regression. The mechanical `/` → `//` rule was wrong for this site because of a `__future__` import I hadn't noticed.

## Implementation tests — fix before step, not during

**Test that will break legitimately:**

```python
def test_cache_uses_lru(self):
    cache = Cache()
    cache.get("a"); cache.get("b"); cache.get("a")
    assert cache._order == ["b", "a"]   # ← asserting internal list order
```

Migrating `Cache` to use `OrderedDict` changes `_order` from a list to dict keys. Test fails. But the *behavior* (LRU eviction order) is preserved.

**Fix the test first, in its own commit:**

```python
def test_cache_evicts_lru(self):
    cache = Cache(maxsize=2)
    cache.get("a"); cache.get("b"); cache.get("a")
    cache.get("c")  # should evict "b", not "a" — "a" was used more recently
    assert "a" in cache and "c" in cache and "b" not in cache
```

Now it tests the *behavior*. Commit that. Then do the migration step — it passes.

## Do not

- **Do not** change the test and the code in the same commit. You can't tell if the test change was legitimate or if you just made it pass.
- **Do not** skip failing tests to keep moving. Every skip is deferred debt, and 20 skips later you don't know if the migration works.
- **Do not** run only the fast subset forever. Full suite at every milestone — subsets miss integration breaks.
- **Do not** migrate uncovered code without acknowledging the risk. "No tests cover this — manually verified by <how>" in the commit message.
- **Do not** trust a green suite on low-coverage code. It's green because nothing checked.

## Output format

```
## Baseline
Tests: <N> passing, <M> skipped (pre-existing: <issues>)
Coverage: <line%> / <branch%>
Uncovered risk areas: <modules with low coverage>

## Migration log
| Step | Change | Tests run | Result | Commit |
| ---- | ------ | --------- | ------ | ------ |
| 1    | <what> | <subset>  | ✓      | <sha>  |
| 2    | <what> | <subset>  | ✗ → <test> | —  |
| 2a   | <localized fix> | ... | ✓  | <sha>  |

## Tests modified (before-migration commits)
| Test | Why it was implementation-coupled | New assertion |
| ---- | --------------------------------- | ------------- |

## Final
Tests: <N> passing, <M> skipped
Behavior regressions found and fixed: <count>
```
