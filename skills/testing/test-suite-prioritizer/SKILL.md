---
name: test-suite-prioritizer
description: Orders tests so failures surface earliest — runs tests covering changed code first, historically flaky/failing tests early, and slow low-value tests last. Use when the suite is too slow to run in full on every change, when CI feedback takes too long, or when deciding what to run in a smoke-test tier.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "coverage-enhancer, test-deduplicator"
---

# Test Suite Prioritizer

If the suite takes 40 minutes and fails at minute 38, you wasted 38 minutes. Run the test that's going to fail **first.** Prioritization is predicting failures and front-loading them.

## Signals for priority

| Signal                         | Why it predicts failure                                  | How to get it                                     |
| ------------------------------ | -------------------------------------------------------- | ------------------------------------------------- |
| Covers changed code            | This change might have broken it                         | Coverage map + `git diff --name-only`             |
| Failed recently                | What failed yesterday fails today                        | CI history — last N runs                          |
| Flaky                          | Runs early → flakes detected early, can be retried       | CI history — pass/fail variance                   |
| Fast                           | More tests per minute of budget                          | Duration from last run                            |
| High code coverage (this test) | Covers more → more chance of catching something          | Per-test coverage                                 |
| Co-change with modified files  | Historically changes with these files                    | `git log` correlation                             |

Combine into a score. Sort. Run in order.

## Scoring formula (starting point — tune it)

```
priority(test) =
    10.0 * covers_changed_lines(test, diff)       # binary: 1 if any overlap
  +  3.0 * recent_failure_rate(test, last_20_runs)
  +  1.0 * flake_rate(test)
  +  0.1 * (1.0 / (duration_seconds(test) + 1))   # tie-break: faster first
  -  5.0 * is_quarantined(test)                   # known-broken go last
```

The `covers_changed_lines` weight dominates: if you changed `foo.py`, tests that cover `foo.py` run first. Everything else is tie-breaking.

## Building the change → test map

You need: *which tests cover which lines.* Per-test coverage:

| Ecosystem | How                                                                  |
| --------- | -------------------------------------------------------------------- |
| Python    | `pytest-testmon` maintains the map incrementally; or `coverage run --parallel` per test + `coverage combine` |
| Java      | JaCoCo per-test — tricky, needs agent per test; or use `git diff` → changed classes → tests importing them (approximation) |
| JS        | `jest --changedSince=<ref>` does this natively                       |
| Go        | `go test -run` with package granularity; or `gotestsum --junitfile` + parse |

The map is expensive to build from scratch (run every test in isolation, collect coverage). Build it once, update incrementally.

## Tiered execution

Don't just reorder — **cut off:**

| Tier       | What                                            | Runs on          | Time budget |
| ---------- | ----------------------------------------------- | ---------------- | ----------- |
| Smoke      | Top-20 by priority                              | Every commit     | < 2 min     |
| Affected   | Everything covering changed code                | Every PR         | < 10 min    |
| Full       | Everything, priority-ordered                    | Merge to main    | Whatever it takes |
| Nightly    | Full + slow integration + flake retry loop      | Cron             | Hours OK    |

Fail fast at each tier. Smoke fail → don't run affected. Affected fail → don't run full.

## Worked example

**Change:** PR touches `auth/session.py` (lines 45–60) and `auth/tokens.py` (new file).

**Coverage map says:**

| Test                               | Covers session.py 45–60 | Covers tokens.py | Last 20 runs | Duration |
| ---------------------------------- | ----------------------- | ---------------- | ------------ | -------- |
| `test_session_refresh`             | Yes                     | No               | 20/20 pass   | 0.3s     |
| `test_token_issuance`              | No                      | Yes              | (new)        | 0.1s     |
| `test_session_expiry`              | Yes (line 58)           | No               | 18/20 pass   | 0.2s     |
| `test_login_e2e`                   | Yes (indirectly)        | Yes              | 20/20 pass   | 12s      |
| `test_unrelated_billing`           | No                      | No               | 20/20 pass   | 0.4s     |
| ...300 more unrelated tests...     |                         |                  |              |          |

**Priority order:**
1. `test_session_expiry` — covers change, recent failures (2/20), fast → score 13.2
2. `test_session_refresh` — covers change, clean, fast → 10.1
3. `test_token_issuance` — covers new file, clean, fastest → 10.1
4. `test_login_e2e` — covers both, but slow → 10.008
5–305. Everything else (scores < 1)

**Affected tier runs 1–4** (12.6s total). If they pass, high confidence the change is safe. Full suite runs on merge.

## Flake handling

Flaky tests are a prioritization headache: run them early (catch flakes fast, retry within budget) or run them late (don't block good changes on noise)?

**Answer: run them early, with automatic retry.** A flake that fails-then-passes-on-retry cost you 5 seconds. A flake that fails at minute 38 with no retry budget cost you 38 minutes.

## Do not

- **Do not** trust static import-based "affected test" approximations for dynamic languages. `getattr(module, name)()` isn't an import. Use real coverage.
- **Do not** let the priority order stagnate. Rebuild it when coverage changes significantly (nightly at minimum).
- **Do not** skip the full suite entirely. Prioritization is for fast feedback, not for replacing comprehensive checks. Something always slips past "affected."
- **Do not** deprioritize a test into oblivion. A test that's always last never runs if the budget is tight — at that point delete it or fix it.

## Output format

```
## Change
<diff summary — files, line ranges>

## Coverage map freshness
Last rebuilt: <when>
Staleness: <N tests may have stale coverage>

## Priority order (top 20)
| Rank | Test | Score | Covers change | Recent fails | Duration |
| ---- | ---- | ----- | ------------- | ------------ | -------- |

## Tiers
| Tier | Tests | Est. duration | Run condition |
| ---- | ----- | ------------- | ------------- |

## Run plan
<pytest/jest/mvn command(s) — ordered>

## Flakes
<tests with flake_rate > 0 — retry policy>
```
