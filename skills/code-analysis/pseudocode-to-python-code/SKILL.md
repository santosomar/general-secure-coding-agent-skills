---
name: pseudocode-to-python-code
description: Translates pseudocode into idiomatic Python, choosing the right standard-library structures and leveraging Python idioms that pseudocode doesn't express. Use when implementing an algorithm from a paper or spec, when the user hands you pseudocode and wants Python, or when realizing a verified-pseudocode artifact.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "pseudocode-to-java-code, verified-pseudocode-extractor"
---

# Pseudocode → Python

Python is closer to pseudocode than most languages — that's the danger. It's tempting to transliterate line-by-line and miss the stdlib function that does it in one call.

See → `pseudocode-to-java-code` for the general decision framework. This skill covers Python-specific choices.

## Pseudocode primitives → stdlib

| Pseudocode                      | Python — NOT the loop, the stdlib call                    |
| ------------------------------- | --------------------------------------------------------- |
| `count occurrences of each x`   | `collections.Counter(xs)`                                 |
| `group xs by key(x)`            | `itertools.groupby` (needs sorted!) or `defaultdict(list)` |
| `priority queue`                | `heapq` (min-heap; negate for max)                        |
| `bidirectional map`             | Two dicts, or a class — no stdlib bidict                  |
| `sorted by multiple keys`       | `sorted(xs, key=lambda x: (x.a, -x.b))` — tuple key       |
| `default value on missing key`  | `dict.get(k, default)` or `defaultdict`                   |
| `find first matching`           | `next((x for x in xs if pred(x)), None)`                  |
| `all/any of ...`                | `all(...)` / `any(...)` — short-circuits                  |
| `memoize f`                     | `@functools.cache`                                        |
| `deep copy`                     | `copy.deepcopy` — be deliberate, it's expensive           |

## Python-specific decisions

| Decision                  | Default                                                         |
| ------------------------- | --------------------------------------------------------------- |
| Infinity                  | `float('inf')` — comparable with ints, no overflow              |
| Missing/null              | `None`, with `Optional[T]` type hint                            |
| Error on bad input        | `raise ValueError(...)` for bad values, `TypeError` for bad types |
| Class vs function         | Function unless there's state between calls. Prefer `@dataclass` for state bundles. |
| Mutability of args        | Don't mutate inputs. If the algorithm is in-place in pseudocode, either copy first or document the mutation loudly. |
| Type hints                | Always. Pseudocode is untyped; your Python shouldn't be.        |

## Worked example

**Pseudocode (topological sort, Kahn's):**

```
function topo_sort(G):
    in_degree ← map: each vertex → count of incoming edges
    Q ← queue of all vertices with in_degree 0
    result ← []

    while Q not empty:
        u ← dequeue(Q)
        append u to result
        for each v in neighbors(u):
            in_degree[v] ← in_degree[v] - 1
            if in_degree[v] = 0:
                enqueue(Q, v)

    if |result| ≠ |vertices(G)|:
        error "cycle"
    return result
```

**Python:**

```python
from collections import deque, defaultdict

def topo_sort(adj: dict[str, list[str]]) -> list[str]:
    """
    Topologically sort a DAG given as an adjacency list.

    :param adj: {u: [v1, v2, ...]} — edges u → v_i. Every vertex must appear
                as a key (even if its list is empty).
    :return: vertices in topological order.
    :raises ValueError: if the graph has a cycle.
    """
    in_degree: dict[str, int] = defaultdict(int)
    for u in adj:
        in_degree[u]  # ensure every vertex is in the map, even at 0
    for neighbors in adj.values():
        for v in neighbors:
            in_degree[v] += 1

    queue = deque(u for u, d in in_degree.items() if d == 0)
    result: list[str] = []

    while queue:
        u = queue.popleft()
        result.append(u)
        for v in adj[u]:
            in_degree[v] -= 1
            if in_degree[v] == 0:
                queue.append(v)

    if len(result) != len(adj):
        raise ValueError(f"graph has a cycle ({len(adj) - len(result)} vertices unreachable)")
    return result
```

**Python-specific choices:**

- `deque` not `list` for the queue — `list.pop(0)` is O(n).
- `defaultdict(int)` for in-degree — no `if k not in d: d[k] = 0` boilerplate. But the `in_degree[u]` loop is needed to seed vertices with zero in-edges (sink-only vertices).
- `ValueError` with diagnostic — how many vertices are stuck tells you roughly how big the cycle is.
- Generator expression in `deque(...)` — no intermediate list.

## The index-base trap

Academic pseudocode is usually 1-indexed. Python is 0-indexed. Three strategies:

1. **Shift on translation.** `for i ← 1 to n` → `for i in range(n)`; every `A[i]` stays `A[i]`. Natural Python. **Preferred.**
2. **Pad the array.** `A = [None] + actual_data`, keep `for i in range(1, n+1)`, keep `A[i]`. Ugly but mechanical — fewer mistakes on complex index arithmetic.
3. **Keep 1-indexed loop, adjust accesses.** `for i in range(1, n+1): ... A[i-1]`. Worst — easy to miss one `-1`.

Pick (1) unless the pseudocode does arithmetic on indices that's hard to re-derive.

## Do not

- **Do not** write the loop when stdlib has it. `max(xs, key=f)` not a manual tracking loop. `Counter` not a `defaultdict(int)` + loop.
- **Do not** use `list` as a queue. `pop(0)` is O(n). `deque.popleft()` is O(1).
- **Do not** mutate default arguments. `def f(x, cache={}):` — that `{}` is shared across calls. Use `None` and `cache = cache or {}`.
- **Do not** skip type hints because "Python is dynamic." Pseudocode has implicit types — make them explicit.
- **Do not** translate `error "..."` as `print` + `return None`. Raise an exception.

## Output format

```
## Stdlib mapping
<pseudocode primitive> → <Python stdlib>

## Code
<python — type-hinted>

## Index base
<1-indexed → 0-indexed, strategy used>

## Deviations
<where the Python is structurally different from the pseudocode, and why it's still equivalent>
```
