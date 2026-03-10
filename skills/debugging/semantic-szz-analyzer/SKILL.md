---
name: semantic-szz-analyzer
description: Extends classic SZZ with semantic code understanding to reduce false positives and improve accuracy of bug-introducing commit identification. Use after classic SZZ has produced candidates, when SZZ precision is too low for the task, or when the user needs high-confidence bug-introduction data.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "szz-bug-identifier, regression-root-cause-analyzer"
---

# Semantic SZZ Analyzer

**This skill is a delta over → `szz-bug-identifier`.** Run classic SZZ first; this skill filters and re-ranks its candidates using semantic understanding instead of line-level blame.

Classic SZZ's precision problem: `git blame` is **textual**. It tells you who last touched a line, not who last changed its *meaning*. A variable rename, an indent, a refactor that moves a line unchanged — all of these become false-positive bug-introducers.

## Semantic filters — applied on top of classic SZZ output

| Filter                         | What it checks                                                                | Effect                               |
| ------------------------------ | ----------------------------------------------------------------------------- | ------------------------------------ |
| AST-diff meaningfulness        | Did the blamed commit change the line's AST, or only its text?                | Drops rename/reformat FPs            |
| Def-use chain relevance        | Does the blamed line define/use a variable that the *fix* reads/writes?       | Drops incidental adjacent-line hits  |
| Semantic-preserving refactor   | Is the blamed commit a known-safe refactor (extract method, inline, rename)?  | Reroutes blame to the commit *before* the refactor |
| Bug-pattern match              | Does the blamed commit's diff *look like* it introduces the kind of bug the fix addresses? (null check added → look for the commit that removed a guard or added the deref) | Boosts confidence when matched       |

## Re-ranking

When multiple candidates survive filtering, rank by:

1. **Semantic distance:** how much did the candidate change the *behavior* of the fixed line? A commit that added the line beats one that renamed a variable in it.
2. **Temporal proximity to bug report:** a commit 2 days before the report beats one from 2 years before.
3. **Author signal:** if the fix author is also the candidate author, slight boost — they probably know what they broke.

## Worked example

**Fix:** Adds a null check on `user.profile` before dereferencing.

**Classic SZZ candidates:**
- `a1b2c3d` — "Rename `p` → `profile`" (6 months ago) — blame hits this because the dereference line was touched
- `e4f5g6h` — "Remove unused profile validation" (3 weeks ago) — blame misses this; it touched a *different* line
- `i7j8k9l` — "Add profile feature" (1 year ago) — the original dereference

**Semantic pass:**
- `a1b2c3d`: AST-diff → rename only, no behavior change → **dropped**. Re-blame *through* it.
- Re-blame lands on `i7j8k9l`. But: bug was reported 3 weeks ago. `i7j8k9l` is 1 year old — survived a year without a report?
- Bug-pattern match: the fix adds a guard. Did any recent commit *remove* a guard? → `e4f5g6h` removed `validateProfile()` which checked non-null.

**Verdict:** `e4f5g6h` is the true introducer. Classic SZZ missed it entirely because the fix and the removal touch different lines.

## Edge cases

- **The fix and the introduction touch disjoint code:** Classic SZZ is blind here; semantic SZZ can catch it *only* if the def-use chain connects them. If not, neither algorithm finds it.
- **Introduced by omission:** "The bug is that a check was never added." No commit to blame — the closest you get is the commit that added the code *around* where the check should be.
- **Squash-merge repos:** All bugs blame to the squash commit. You need the pre-squash branch history, or you're stuck.

## Do not

- **Do not** use semantic SZZ as your first pass. It's an order of magnitude slower. Classic SZZ → filter → semantic re-rank on survivors.
- **Do not** override a classic-SZZ hit with a semantic-SZZ miss. Semantic SZZ *adds* candidates the textual blame missed; it shouldn't silently drop direct hits.

## Output format

Same as `szz-bug-identifier`, with an additional `semantic_evidence` field per candidate:

```
fix: <sha>
  candidates (semantic-filtered):
    <sha>  confidence=<high|med|low>
      semantic_evidence: <AST change summary / def-use link / pattern match>
      classic_szz_hit: <yes|no — found via semantic reroute>
```
