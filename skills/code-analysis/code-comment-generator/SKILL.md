---
name: code-comment-generator
description: Generates code comments that explain non-obvious intent, constraints, and tradeoffs — not what the code already says. Use when code is correct but opaque, when documenting for future maintainers, or when a function's why is harder to see than its what.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "code-summarizer, legacy-code-summarizer"
---

# Code Comment Generator

Good comments explain **why**, not **what**. The code already says what. If you're narrating the code, delete the comment and make the code clearer instead.

## What deserves a comment

| Worth commenting                                  | Why                                                   |
| ------------------------------------------------- | ----------------------------------------------------- |
| A non-obvious constraint the code depends on      | "x is sorted here because caller guarantees it"       |
| A tradeoff that was made deliberately             | "O(n²) is fine — n ≤ 20 by construction"              |
| Why the obvious approach is wrong                 | "Can't use .sort() — caller holds references into this list" |
| A workaround for an external bug                  | "libfoo 2.3 returns NULL here; remove when we upgrade" |
| Magic numbers with a source                       | "15s — AWS Lambda cold start p99 + margin"            |
| Coupling to something far away                    | "Must match the regex in nginx.conf"                  |
| A subtle correctness argument                     | "Safe to read without lock: only written at startup"  |

## What does NOT deserve a comment

| Skip                                              | Why                                                   |
| ------------------------------------------------- | ----------------------------------------------------- |
| `i++  // increment i`                             | The code says that. Noise.                            |
| `// loop over items` above `for item in items:`   | Same.                                                 |
| Restating the function name in a docstring        | `def save_user(): """Saves user."""` — waste.         |
| Commented-out code                                | That's what git is for. Delete it.                    |
| `// TODO: fix this later`                         | Unactionable. TODO: *what* and *why not now*?         |
| Comments that will rot                            | "Called from foo.py:42" — until foo.py changes.       |

## The test: does this survive a rename?

If you rename all the variables and the comment still makes sense, it's a good comment (explains intent). If the comment is now wrong, it was restating the code.

```python
# BEFORE rename
x = compute_total(items)
# Store the total     ← restating. Bad.

# AFTER rename test
q = compute_total(items)
# Store the total     ← now the comment is confusing. It was bad.
```

vs.

```python
# Compute total before filtering — downstream needs the pre-filter sum for the audit log.
x = compute_total(items)
items = [i for i in items if i.active]
```

Rename `x` → comment still explains *why the ordering matters*. Good.

## Worked example

**Uncommented code:**

```python
def reconnect(self):
    delay = 0.1
    for attempt in range(8):
        try:
            self._connect()
            return
        except ConnectionError:
            time.sleep(delay + random.uniform(0, delay))
            delay = min(delay * 2, 5.0)
    raise ConnectionError("exhausted retries")
```

**What the code already says:** retries 8 times, exponential backoff, cap at 5s, jitter. Don't comment any of that.

**What's not obvious:**

```python
def reconnect(self):
    # Full jitter would be random.uniform(0, delay), but that occasionally
    # produces near-zero sleeps that hammer the server. Half-jitter (delay + uniform(0, delay))
    # guarantees at least `delay` between attempts — gentler on the remote.
    delay = 0.1
    for attempt in range(8):   # 8 attempts × max 10s each ≈ 80s worst case — under the 90s LB timeout
        try:
            self._connect()
            return
        except ConnectionError:
            time.sleep(delay + random.uniform(0, delay))
            delay = min(delay * 2, 5.0)
    raise ConnectionError("exhausted retries")
```

Two comments. The first explains a non-obvious design choice (why half-jitter, not full-jitter). The second explains a magic number in terms of an external constraint (LB timeout).

## Docstrings — the API contract

Docstrings are a different animal: they're for *callers*, not maintainers. They should state:

- What the function does (one line)
- Parameter constraints (types, ranges, nullability)
- Return value meaning (including sentinel values)
- Exceptions raised and when
- Side effects (mutation, I/O)

They should NOT describe the implementation. If the implementation changes but the contract doesn't, the docstring shouldn't need updating.

## Do not

- **Do not** write `// increment counter` above `counter++`. Ever.
- **Do not** explain *what* a standard idiom does. `# list comprehension` above a list comprehension — the reader knows Python or they don't; the comment won't help either way.
- **Do not** leave `TODO` without a what and a why-not-now. `TODO(#1234): remove this shim once libfoo 3.0 ships (expected Q2)` is actionable. `TODO: fix` is not.
- **Do not** write comments that reference line numbers or file paths elsewhere. They rot immediately. Reference concepts, not locations.
- **Do not** add a comment when the fix is to rename a variable. `x = get_total()  # x is the total` → just `total = get_total()`.

## Output format

For each comment proposed:

```
## <file:line>

### Current code
<snippet>

### Proposed comment
<the comment>

### Why this comment
<what non-obvious thing it explains — constraint, tradeoff, coupling, workaround>

### Not commenting
<things in this snippet that looked comment-worthy but aren't — the code says it>
```
