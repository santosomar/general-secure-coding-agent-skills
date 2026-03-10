---
name: pseudocode-to-java-code
description: Translates pseudocode into idiomatic Java, inferring types, choosing collection classes, and handling exceptions per Java conventions. Use when implementing an algorithm from a paper or spec, when the user hands you pseudocode and wants Java, or when realizing a verified-pseudocode artifact.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "pseudocode-to-python-code, verified-pseudocode-extractor"
---

# Pseudocode → Java

Pseudocode is under-specified on purpose. The Java you produce has to commit to everything pseudocode leaves open: types, nullability, error handling, mutability, collection implementations.

## Decisions pseudocode doesn't make (but Java requires)

| Pseudocode says       | Java must decide                                                    |
| --------------------- | ------------------------------------------------------------------- |
| `let S be a set`      | `HashSet`? `TreeSet`? `LinkedHashSet`? (Does order matter?)         |
| `x ← lookup(k)`       | Returns `null` on miss? `Optional<V>`? Throws?                      |
| `list of numbers`     | `int[]`? `List<Integer>`? `IntStream`? (Boxed vs primitive matters.)|
| `error: ...`          | Checked `Exception`? `RuntimeException`? Return sentinel?           |
| `for each x in S`     | Enhanced for? Stream? (Mutation during iteration → `ConcurrentModificationException`.) |
| `procedure f(x)`      | Static method? Instance method? What class does it live on?         |

**Default choices** (absent a reason to deviate):

- Sets/maps → `HashSet`/`HashMap` unless order is mentioned; `LinkedHash*` if insertion order matters; `Tree*` if sorted iteration is used.
- Missing lookup → `Optional` for new code, `null` if matching an existing API.
- Number list → `List<Integer>` for flexibility, `int[]` if the pseudocode indexes and size is fixed.
- Errors → `IllegalArgumentException` for bad inputs, `IllegalStateException` for bad state, checked only if the caller can meaningfully recover.
- Home for the method → a final utility class with static methods, unless there's obvious object state.

## Type inference from pseudocode

Pseudocode variables are used before typed. Infer from usage:

| Usage in pseudocode        | Inferred Java type                           |
| -------------------------- | -------------------------------------------- |
| `x ← 0; x ← x + 1`         | `int` (or `long` if values look big)         |
| `x ← []; x.append(y)`      | `List<typeof(y)>` (`ArrayList`)              |
| `x ← {}; x[k] ← v`         | `Map<typeof(k), typeof(v)>`                  |
| `x ← ∅; x ← x ∪ {y}`       | `Set<typeof(y)>`                             |
| `for i ← 1 to n`           | `int i` (1-indexed — watch off-by-one)       |
| `x ← head(list); rest(list)` | Consider `Deque` or `LinkedList`           |

## Worked example

**Pseudocode (Dijkstra, textbook style):**

```
function shortest_paths(G, source):
    dist ← map with dist[source] = 0, all others = ∞
    visited ← ∅
    Q ← priority queue of all vertices, keyed by dist

    while Q is not empty:
        u ← extract-min(Q)
        visited ← visited ∪ {u}
        for each neighbor v of u where v ∉ visited:
            alt ← dist[u] + weight(u, v)
            if alt < dist[v]:
                dist[v] ← alt
                decrease-key(Q, v, alt)

    return dist
```

**Java:**

```java
import java.util.*;

public final class Dijkstra {
    private Dijkstra() {}

    public record Edge(int to, int weight) {}

    /**
     * Computes shortest distances from {@code source} to all reachable vertices.
     * @param adj adjacency list: adj.get(u) is the outgoing edges from u
     * @param source starting vertex
     * @return map from vertex to shortest distance; unreachable vertices absent
     * @throws IllegalArgumentException if source is not in adj
     */
    public static Map<Integer, Integer> shortestPaths(
            Map<Integer, List<Edge>> adj, int source) {
        if (!adj.containsKey(source)) {
            throw new IllegalArgumentException("source vertex not in graph");
        }

        Map<Integer, Integer> dist = new HashMap<>();
        dist.put(source, 0);
        Set<Integer> visited = new HashSet<>();

        // Java's PriorityQueue has no decrease-key. Standard workaround:
        // insert duplicates, skip stale entries on poll.
        PriorityQueue<int[]> pq = new PriorityQueue<>(Comparator.comparingInt(a -> a[1]));
        pq.offer(new int[]{source, 0});

        while (!pq.isEmpty()) {
            int[] top = pq.poll();
            int u = top[0], d = top[1];
            if (visited.contains(u)) continue;          // stale entry — already settled
            visited.add(u);

            for (Edge e : adj.getOrDefault(u, List.of())) {
                if (visited.contains(e.to())) continue;
                int alt = d + e.weight();
                Integer cur = dist.get(e.to());
                if (cur == null || alt < cur) {
                    dist.put(e.to(), alt);
                    pq.offer(new int[]{e.to(), alt});   // "decrease-key" = insert new, ignore old
                }
            }
        }
        return dist;
    }
}
```

**Decisions made (and why):**

- `∞` → absent from map. `dist.get(v) == null` means infinity. Avoids `Integer.MAX_VALUE` overflow when adding weights.
- `decrease-key` → Java's PQ doesn't have it. Lazy deletion: insert duplicates, skip visited on poll. O((V+E) log V) — same asymptotic.
- `int[]{vertex, dist}` in PQ instead of a record — cheap, no boxing. Comment explains.
- `adj.getOrDefault(u, List.of())` — graph might have vertices with no outgoing edges.
- Unreachable vertices absent from result — documented. Caller checks `containsKey`.

## Preserving annotations from verified pseudocode

If the pseudocode came from → `verified-pseudocode-extractor`, it has `// PRE:`, `// POST:`, `// INV:` annotations. **Keep them as Javadoc and comments:**

- `PRE` → `@throws IllegalArgumentException if ...` or `@param ... must be ...`
- `POST` → `@return` clause
- `INV` → comment at the loop head (don't delete — it's why the loop is correct)

## Do not

- **Do not** use `Integer.MAX_VALUE` for infinity without guarding additions. `MAX_VALUE + weight` overflows to negative. Use absent-from-map, or `Long`, or check before adding.
- **Do not** mutate a collection while iterating it with for-each. `ConcurrentModificationException`. Use an explicit iterator with `.remove()`, or iterate over a copy.
- **Do not** translate 1-indexed pseudocode without adjusting. `for i ← 1 to n` → `for (int i = 0; i < n; i++)` and every `a[i]` stays `a[i]` (not `a[i-1]`) — *or* keep 1-indexed loops and adjust all accesses. Pick one, consistently.
- **Do not** make everything a static utility method if the pseudocode has persistent state between calls. That's an object.

## Output format

```
## Type decisions
| Pseudocode var | Java type | Why |
| -------------- | --------- | --- |

## Code
<java>

## Deviations from pseudocode
<decrease-key workaround, ∞ encoding, index base — anything where the Java isn't a direct reading>

## Javadoc
<pre/post conditions translated>
```
