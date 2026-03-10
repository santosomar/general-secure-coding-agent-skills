---
name: smart-mutation-operator-generator
description: Generates domain-specific mutation operators beyond the standard arithmetic/relational set — mutations tailored to your codebase's idioms, APIs, and bug history that standard tools don't try. Use when generic mutation testing plateaus, when your domain has specific failure modes, or when mining bug history reveals patterns standard operators miss.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "mutation-test-suite-optimizer"
---

# Smart Mutation Operator Generator

Standard mutation (`+`→`-`, `<`→`<=`) tests generic arithmetic. Your bugs aren't generic. If your last three bugs were "forgot to check `.is_expired()`," no standard operator catches that. Write one that does.

## Why standard operators plateau

| Standard operator | Catches                      | Misses                                        |
| ----------------- | ---------------------------- | --------------------------------------------- |
| `+` ↔ `-`         | Arithmetic errors            | API misuse, resource leaks, protocol errors   |
| `<` ↔ `<=`        | Off-by-one                   | Wrong method called on similar API            |
| Remove statement  | Unchecked side effects       | Wrong order of operations on stateful API     |
| Negate condition  | Untested branches            | Missing check entirely (not wrong check)      |

Past 80% mutation score on standard operators, the survivors are mostly equivalent mutants. The real bugs live elsewhere.

## Mining mutation operators from bug history

Your git log is a mutation operator database:

```bash
git log --all --grep="fix\|bug" -p --since="2 years ago" > fix-commits.txt
```

For each fix diff, ask: **what's the inverse?** The inverse of the fix is the mutation.

| Bug fix                                      | Inverse = mutation operator                       |
| -------------------------------------------- | ------------------------------------------------- |
| Added `if session.is_expired(): raise ...`   | **Delete `is_expired()` checks**                  |
| Changed `json.loads(s)` → `json.loads(s, parse_float=Decimal)` | **Drop `parse_float` kwarg** |
| Changed `.get(k)` → `.get(k, default)`       | **Drop the default from `dict.get`**              |
| Added `conn.commit()` before return          | **Delete commit calls before return**             |
| Swapped `list.pop()` → `list.pop(0)`         | **Swap pop index (LIFO ↔ FIFO)**                  |
| Added `@transaction.atomic`                  | **Remove `@transaction.atomic` decorators**       |

Each of these is a mutation your standard tool doesn't try. Each of them represents a **real past bug** — if it slips in again, will tests catch it?

## Domain-specific operator catalog

| Domain           | Operator                                                    | Bug it simulates                        |
| ---------------- | ----------------------------------------------------------- | --------------------------------------- |
| Web/HTTP         | Swap status codes: 200↔201, 400↔404, 401↔403                | Wrong status → client mishandles        |
| Web/HTTP         | Drop a response header                                      | Missing CORS/cache header               |
| Async            | Remove `await` (Python) / `.await` (Rust)                   | Fire-and-forget bug                     |
| Async            | Swap `asyncio.gather` ↔ sequential loop                     | Lost concurrency, or race introduced    |
| DB/ORM           | Remove `.select_for_update()`                               | Lost locking → race condition           |
| DB/ORM           | Swap `.filter()` ↔ `.exclude()`                             | Inverted query                          |
| DB/ORM           | Drop `.distinct()`                                          | Duplicate rows                          |
| Locking          | Remove `with lock:` wrapper                                 | Unprotected critical section            |
| Resource         | Remove `.close()` / drop `with` block                       | Resource leak                           |
| Serialization    | Swap field order in tuple unpacking                         | Misaligned deserialization              |
| Retry logic      | Set `max_attempts=1`                                        | Retry logic never exercised             |
| Auth             | Replace `check_permission(user)` with `True`                | Auth bypass — does any test notice?     |

## Implementing a custom operator

Most mutation tools accept custom operators. `mutmut` example:

```python
# mutmut_config.py

def pre_mutation(context):
    line = context.current_source_line

    # Custom operator: remove `parse_float=Decimal` from json.loads calls
    # (Simulates bug #4521 — float precision loss in financial fields)
    if "json.loads" in line and "parse_float=Decimal" in line:
        context.mutations.append(
            line.replace(", parse_float=Decimal", "")
        )

    # Custom operator: remove `@transaction.atomic` decorator
    # (Simulates bug #3390 — partial writes on exception)
    if line.strip() == "@transaction.atomic":
        context.mutations.append("")   # delete the decorator line

    # Custom operator: swap `.get(k, default)` → `.get(k)`
    # (Simulates bug #2817 — KeyError on missing config)
    import re
    m = re.search(r"\.get\(([^,]+),\s*[^)]+\)", line)
    if m:
        context.mutations.append(
            line[:m.start()] + f".get({m.group(1)})" + line[m.end():]
        )
```

Each operator cites the bug it came from. When a mutant from operator X survives, you know: "the test suite wouldn't catch bug #4521 if it recurred."

## Validating a new operator

Before adding an operator permanently:

1. **Does it produce compileable/runnable mutants?** An operator that always makes syntax errors is useless — those get killed trivially by "the test file doesn't import."
2. **Does it produce *non-equivalent* mutants?** Try it on a few locations. Can you imagine a test that distinguishes?
3. **Is it *killable* in principle?** Removing a log line might be a valid operator, but if your policy is "don't test logs," every mutant survives — noise.
4. **Is it targeted?** `Replace any string with ""` produces thousands of mutants, mostly noise. `Replace auth token with ""` produces a few, all meaningful.

## Do not

- **Do not** add operators for bugs your tests *shouldn't* catch. If the policy is "logging is best-effort," a "remove log call" operator generates noise.
- **Do not** make operators so specific they apply once. `Delete line 47 of auth.py` is a test, not an operator. Operators are patterns.
- **Do not** add operators without citing the bug class. Six months later you won't remember why `swap pop index` is in the list.
- **Do not** let custom operators explode the mutant count. Mutation testing is already slow. Target: each operator applies to 10–100 sites, not 10,000.

## Output format

```
## Bug history mined
| Bug # | Fix summary | Inverse mutation |
| ----- | ----------- | ---------------- |

## Proposed operators
### OP-<N>: <name>
Pattern: <what it matches>
Mutation: <what it changes>
Bug class: <cites bug # or CWE>
Expected sites: ~<N> in current codebase
Equivalence risk: <low | medium | high — and why>

## Operator implementation
<tool-specific config — mutmut_config.py, pitest plugin, stryker plugin>

## Dry run
| Operator | Mutants generated | Killed | Survived | Equivalent (est.) |
| -------- | ----------------- | ------ | -------- | ----------------- |

## Recommended for adoption
<subset with good kill/survive/equivalent ratio>
```
