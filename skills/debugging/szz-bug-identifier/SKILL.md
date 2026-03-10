---
name: szz-bug-identifier
description: Applies the SZZ algorithm to VCS history to identify which commits introduced bugs by correlating bug-fix commits with earlier changes. Use when mining a repository for bug-introducing commits, when building a defect-prediction dataset, or when the user asks which commit introduced a given fixed bug.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "semantic-szz-analyzer, regression-root-cause-analyzer"
---

# SZZ Bug Identifier

SZZ (Śliwerski, Zimmermann, Zeller, 2005) answers: **given a bug-fix commit, which earlier commit introduced the bug?** It works by blaming the lines the fix touched.

## The algorithm — classic SZZ

1. **Identify the fix commit.** Usually via a link to an issue tracker (`Fixes #1234`, `BUG-567`) or a message keyword (`fix`, `bug`, `patch`).
2. **Extract the deleted/modified lines.** In the fix's diff, every line removed or changed is a line that was (potentially) buggy.
3. **Blame each of those lines.** `git blame <fix>^ -- <file>` on each modified line to find the commit that last touched it *before* the fix.
4. **Those blamed commits are bug-introducing candidates.**

That's the whole algorithm. The rest is noise filtering.

## Noise filters — classic SZZ's known weaknesses

| Noise source                      | Why it's wrong                                           | Filter                                                 |
| --------------------------------- | -------------------------------------------------------- | ------------------------------------------------------ |
| Whitespace / formatting changes   | Blame hits a `prettier` run, not the real introducer     | `git blame -w`; ignore commits that only touch formatting |
| Comment-only changes              | The fix edited a comment too — that line is not the bug  | Strip comment lines before blaming                     |
| Large refactor commits            | Every line blames to the Great Refactor of 2019          | `git blame --ignore-rev` with a curated ignore-list    |
| The line was *added* by the fix   | No blame target — added lines didn't exist before        | Only blame *deleted/modified* lines, not added         |
| Bug predates the repo             | Blame hits the initial import commit                     | Flag — can't attribute                                 |
| Moved file                        | Blame stops at the `git mv`                              | `git blame -C -M` to follow moves/copies               |
| Blamed commit is newer than bug report | The bug existed before that commit; blame is wrong | Discard candidates with commit-date > bug-report-date  |

## Worked example

**Fix commit:** `c4a9f1b — Fix: null check in getUserEmail (closes #892)`

```diff
  public String getUserEmail(long id) {
      User u = repo.find(id);
-     return u.getEmail();
+     if (u == null) return null;
+     return u.getEmail();
  }
```

**Step 2:** Modified line is `return u.getEmail();` (the old version).

**Step 3:** `git blame c4a9f1b^ -- UserService.java` at that line → `a17d3e0 — Add UserService (Jane, 2021-03-04)`.

**Step 4:** Candidate = `a17d3e0`.

**Filters:**
- Whitespace? No, it's a real code line. ✓
- Comment? No. ✓
- Refactor in ignore-list? No. ✓
- Added by the fix? No — this is the old version of a modified line. ✓
- Newer than bug report? Issue #892 opened 2023-11-01; `a17d3e0` is 2021. ✓

**Verdict:** `a17d3e0` introduced the bug. The null-check was never there.

## Edge cases

- **Multi-line fix where lines blame to different commits:** Each blamed commit is a candidate. Usually one is the real introducer and the rest are incidental; → `semantic-szz-analyzer` to distinguish.
- **Fix is a one-line addition (no deletion):** Classic SZZ has nothing to blame. Heuristic: blame the *surrounding* lines (the ones the added line sits between). Low confidence.
- **The fix reverts an earlier commit entirely:** `git revert` — the reverted commit IS the bug-introducer, definitionally. Shortcut: check if the fix is a revert before running SZZ.
- **Cherry-picked commits:** The blame may point at the cherry-pick, not the original. Check for `(cherry picked from commit …)` trailers and follow the chain.

## Do not

- **Do not** treat SZZ output as ground truth. Published precision is 40–70% without filters. It's a candidate generator for human review.
- **Do not** run SZZ on a repo without a curated `.git-blame-ignore-revs`. Without it, every result blames the last formatting pass.
- **Do not** use commit-message keyword matching (`fix`, `bug`) as your only fix-identification signal. False-positive rate is brutal. Prefer issue-tracker links.
- **Do not** run SZZ at bug-fix time for a single bug — that's what → `regression-root-cause-analyzer` is for (bisect is more precise). SZZ is for *batch mining*.

## Output format

```
fix: <sha> — <subject>
  candidates:
    <sha> — <subject>  (<date>, <author>)
      blamed from: <file>:<line>
      filters passed: whitespace ✓  comment ✓  date ✓
      confidence: <high|medium|low>
    ...
```
