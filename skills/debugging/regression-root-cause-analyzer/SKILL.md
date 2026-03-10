---
name: regression-root-cause-analyzer
description: Traces regressions to the specific commit, change, or code path that introduced the behavioral breakage. Use when a previously passing test or feature now fails, when the user asks what change caused a regression, or when bisecting a regression across commits.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "bug-localization, szz-bug-identifier, bug-to-patch-generator"
---

# Regression Root Cause Analyzer

A regression has one advantage over a greenfield bug: somewhere in history there is a commit where it **worked**. The entire skill is exploiting that advantage. Do not read code first — read history first.

## Step 1 — Establish the good/bad bounds

| What you know                             | Good bound                 | Bad bound               |
| ----------------------------------------- | -------------------------- | ----------------------- |
| "Worked in v2.3, broken in v2.4"          | `v2.3` tag                 | `v2.4` tag              |
| "Broke sometime this week"                | Monday's first commit      | `HEAD`                  |
| "Test X just started failing in CI"       | Last green CI SHA          | First red CI SHA        |
| "Broke after I pulled"                    | `ORIG_HEAD` or `reflog`    | `HEAD`                  |
| "No idea when"                            | Oldest tag — or bail (§ edge cases) | `HEAD`       |

**Verify both bounds** before bisecting. Check out good → run → confirm green. Check out bad → run → confirm red. If either bound is wrong, bisect will converge on garbage.

## Step 2 — Bisect

`git bisect` is log₂(N), so even 1000 commits is 10 runs. Automate if the oracle is scriptable:

```
git bisect start <bad> <good>
git bisect run ./oracle.sh
```

Where `oracle.sh` exits 0 when the regression is **absent** and non-zero when **present**. If the oracle is manual (visual check), you're doing it by hand — still only ~10 iterations.

| Oracle situation                          | Approach                                                        |
| ----------------------------------------- | --------------------------------------------------------------- |
| Single failing test                       | `oracle.sh = pytest path/to/test.py::test_name`                 |
| Build breaks mid-range                    | Exit 125 in oracle to skip — bisect knows how to route around   |
| Oracle takes 10 minutes                   | Bisect by first-parent on merge commits first, then drill into the guilty merge |
| Behavior, not a test                      | Write a one-liner test *now*; it's your bisect oracle and your regression guard |

## Step 3 — Read the guilty commit

Bisect gives you a SHA. Now engage: `git show <sha>`. You're looking for **which hunk** caused it, not just which commit.

| Commit shape                | Triage                                                                  |
| --------------------------- | ----------------------------------------------------------------------- |
| Single small hunk           | Done — this is the root cause                                           |
| Multi-file refactor         | Revert one file at a time against `bad`; re-run oracle                  |
| Merge commit                | Bisect the merged branch separately: `git bisect start <merge> <merge>^1` |
| Dependency bump             | Diff the dep's changelog between old/new pins                           |
| Seemingly unrelated commit  | Re-verify your bounds — 90% of "bisect lied" is actually bad bounds     |

## Worked example

**Input:** `test_admin_can_delete_user` passes on `v3.1.0`, fails on `main` (400 commits later).

```
git bisect start main v3.1.0
git bisect run pytest tests/test_admin.py::test_admin_can_delete_user -x
```

9 iterations. Guilty: `a3f91c2 — Refactor permission middleware`.

`git show a3f91c2` — 200-line diff across 4 files. Revert files one at a time:

```
git checkout a3f91c2 -- src/middleware/perms.py && pytest ... -x   # still red
git checkout a3f91c2^ -- src/middleware/perms.py && pytest ... -x  # green
```

Fault is in `perms.py`. Inspect its hunk: `ADMIN in user.roles` became `user.role == ADMIN` during the refactor — but users have a *list* of roles. One-character design shift, 200-line diff.

→ `bug-to-patch-generator` with fault location `src/middleware/perms.py:L42`.

## Edge cases

- **Bisect blames a merge commit and both parents are green:** The bug is in the *merge itself* — an automatic or manual merge resolution that combined two correct changes into an incorrect result. Inspect `git show -m <merge>`.
- **The commit reverts cleanly but the test still fails:** Your bounds were wrong, or there are two independent regressions. Bisect again from good to the commit *before* the one you found.
- **Good bound doesn't build anymore (toolchain drift):** Use `git bisect skip` generously, or bisect the *generated artifact* (if CI retains old builds) rather than source.
- **No known good bound:** This isn't a regression — it's a bug that was always there. → `bug-localization`.

## Do not

- **Do not** start by reading diffs manually. Bisect is faster than you are.
- **Do not** trust a bisect whose good bound you haven't verified. Garbage in, garbage out.
- **Do not** stop at "this commit." The output is "this line in this commit" — keep drilling.
- **Do not** blame the dependency bump without reading the dep's changelog. The bump *surfaced* the bug; it might not have *caused* it.

## Output format

```
## Introducing commit
<sha> — <subject>
<author>, <date>

## Fault hunk
<file:line-range>
<the relevant diff lines, not the whole commit>

## Mechanism
<One paragraph: what the change did and why it breaks the observed behavior>

## Handoff
→ bug-to-patch-generator with <file>:<line>
```
