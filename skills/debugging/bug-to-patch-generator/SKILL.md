---
name: bug-to-patch-generator
description: Automatically synthesizes code patches to fix identified bugs, leveraging the bug location and surrounding context. Use when a bug has been localized and the user wants an automated fix, when generating candidate patches for review, or when the user asks to fix a specific bug.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "bug-localization, bug-reproduction-test-generator, behavior-preservation-checker"
---

# Bug-to-Patch Generator

Produce the **minimal** edit that makes the failing test pass without breaking anything else. The discipline is subtractive: most first-draft patches are too big, and every unnecessary line is a line a reviewer has to verify.

## Preconditions — do not start patching until you have these

| Precondition                | Why you need it                                         | If missing                              |
| --------------------------- | ------------------------------------------------------- | --------------------------------------- |
| Fault location (file:line)  | You need to know *where* to edit                        | → `bug-localization`                    |
| Failing test                | You need a red→green signal                             | → `bug-reproduction-test-generator`     |
| Green baseline              | You need to know what "doesn't break anything" means    | Run the full suite once, record result  |

If you don't have all three, you are not patching a bug — you are guessing.

## Step 1 — Classify the fault

Match the fault to a fix family. Different families have different minimal-patch shapes.

| Fault class                  | Typical symptom                          | Minimal patch shape                           |
| ---------------------------- | ---------------------------------------- | --------------------------------------------- |
| Off-by-one                   | Loop over/under-runs by one              | Change `<` ↔ `<=`, `n` ↔ `n-1`, `++i` ↔ `i++` |
| Missing null/empty guard     | NPE / KeyError / index-out-of-bounds     | One early-return or `if (x == null)` branch   |
| Wrong operator               | Inverted condition, swapped args         | Single-token edit: `&&` ↔ `\|\|`, swap two args |
| Wrong constant               | Magic number is slightly wrong           | One literal changes                           |
| Missing state update         | Stale cache, unrotated log, unset flag   | One statement added: the missing mutation     |
| Wrong API call               | Deprecated/misused library call          | Replace call; may cascade to import/signature |
| Missing case                 | Unhandled enum branch / edge input       | One branch added to a switch/match/if-chain   |
| Resource leak                | FD/connection/lock not released          | Move into `try`/`with`; add `finally`/`defer` |

Start with the **smallest** family that could explain the symptom.

## Step 2 — Draft the patch

Write the minimal candidate for the classified family. Constraints:

- Touch **one location** if at all possible. Two-location patches are suspicious; three-location patches are almost always wrong or hiding a deeper design flaw.
- No refactoring. The temptation to "clean up while you're here" couples the fix to unrelated risk.
- No new abstractions. A bug fix that introduces a class or interface is a design change wearing a bug-fix costume.
- Match surrounding style. If the file uses `==` not `is`, use `==`.

## Step 3 — Validate

Run validation in order of cost. Stop at the first failure.

| Gate                    | What it checks                                      | On failure                                 |
| ----------------------- | --------------------------------------------------- | ------------------------------------------ |
| Compile / typecheck     | Patch is syntactically & type-valid                 | Revert; your classification is wrong       |
| Failing test now passes | Patch actually fixes the bug                        | Wrong family — go back to Step 1           |
| Green baseline still green | No collateral damage                             | Shrink the patch; you changed too much     |
| → `behavior-preservation-checker` | Semantic equivalence on unchanged paths   | If time allows — catches subtle breakage   |

## Worked example

**Input:**
- Fault: `src/pagination.py:47` in `page_slice(items, page, per_page)`
- Failing test: `test_last_page_partial` — last page returns `[]` when it should return 2 items
- Code at fault line:

```python
def page_slice(items, page, per_page):
    start = page * per_page
    end = start + per_page
    if end > len(items):
        return []                    # line 47
    return items[start:end]
```

**Classification:** Missing case — but look closer. The guard `end > len(items)` isn't missing a case, it's *wrong*. The fault class is actually **wrong operator**: it should only bail when `start` is past the end, not `end`.

**Minimal patch:**

```diff
-    if end > len(items):
-        return []
+    if start >= len(items):
+        return []
```

One line. `items[start:end]` already handles `end > len(items)` correctly via Python's slice semantics.

**Validation:** `test_last_page_partial` passes; full suite green. Done.

## Edge cases

- **The fix is "delete the guard":** Sometimes the bug *is* the defensive code. If removing a condition makes the test pass and nothing else fails, removal is the fix. Don't invent a replacement guard.
- **Two fixes both work:** Pick the one closer to the fault's introduction point. If commit A added a bad condition and commit B worked around it downstream, fix A and delete B's workaround.
- **The test was wrong:** Rare but real. If the code matches spec and the test doesn't, patch the test — but flag it explicitly; never silently flip an assertion.
- **Intermittent failure:** Stop. You cannot validate a patch against a flaky oracle. → `bug-reproduction-test-generator` to stabilize first.

## Do not

- **Do not** add `try/except` to swallow the symptom. That's a bug in a trench coat.
- **Do not** broaden the fix beyond the failing scenario "just in case." You don't have a test for "just in case."
- **Do not** change the test assertion to match current behavior unless you've established the test is wrong.
- **Do not** ship a patch that touches more than ~5 lines without a second review. The minimal fix for a real bug is almost never that big.

## Output format

```
## Patch
<unified diff — minimal>

## Classification
<fault family from table>

## Validation
- [ ] compiles
- [ ] failing test → pass
- [ ] baseline suite → green
- [ ] behavior-preservation check (optional)

## Assumptions
<anything the patch relies on that isn't proven by the test>
```
