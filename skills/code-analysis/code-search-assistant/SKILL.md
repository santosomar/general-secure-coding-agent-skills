---
name: code-search-assistant
description: Finds code by meaning, structure, or text across large codebases — picks the right search strategy (grep, AST query, call graph walk, semantic search) for the question being asked. Use when the user asks where something is implemented, when navigating unfamiliar code, or when a simple grep isn't enough.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "code-pattern-extractor, legacy-code-summarizer"
---

# Code Search Assistant

There are four kinds of code search. Picking the wrong one wastes time or misses results. The skill is matching the question to the tool.

## Question → tool

| Question shape                                | Tool                       | Why                                          |
| --------------------------------------------- | -------------------------- | -------------------------------------------- |
| "Where is `FooBar` defined?"                  | Text grep (`rg -w`)        | Exact symbol — fast, precise                 |
| "Where is `FooBar` *used*?"                   | Text grep + filter, or LSP "find references" | Same symbol, many hits |
| "What calls this function, transitively?"     | Call graph walk            | Grep finds direct calls; you need the tree   |
| "Where do we validate email addresses?"       | Semantic / fuzzy search    | Concept, not symbol — no single keyword      |
| "Find all places that cast then dereference"  | AST / structural query     | Syntactic pattern, not a string              |
| "What's the code path from HTTP to DB?"       | Dataflow / taint trace     | Cross-function, value-following              |

## Text search — do it right

Grep is fast but dumb. Make it less dumb:

| Trick                                   | Example                                              |
| --------------------------------------- | ---------------------------------------------------- |
| Word boundaries                         | `rg -w foo` — matches `foo` not `foobar`             |
| File type filter                        | `rg -t py foo` — only Python files                   |
| Definition vs use                       | `rg '^(def|class) foo'` vs `rg '\bfoo\('`            |
| Multi-line pattern                      | `rg -U 'if.*\n.*return None'`                        |
| Exclude vendored/generated              | `rg foo -g '!vendor/' -g '!*.pb.go'`                 |
| Case-insensitive for NL concepts        | `rg -i 'email.*valid'`                               |

**False-positive pruning:** comments, strings, tests. `rg foo | rg -v test_ | rg -v '^.*#'` — crude but works. Or use the `-t` type filter to skip test directories if the language has conventions.

## Structural search — when grep lies

Grep finds *text*. AST search finds *structure*. You need AST when:

- The pattern has nesting: "a `return` inside a `for` inside a `try`."
- The pattern is semantic: "function calls where the 2nd arg is a string literal."
- The pattern spans lines in ways regex can't track.

Tools: `semgrep` (pattern syntax looks like code with holes), `ast-grep`, language-specific (Python `ast` module, clang query).

**Example semgrep pattern** — find SQL built by concatenation:

```yaml
pattern: |
  $CURSOR.execute($X + $Y)
```

Grep for `execute` gives you thousands of hits. The pattern gives you the dangerous ones.

## Call graph — the transitive question

"What eventually calls `dangerous_write`?" Grep finds direct callers. For the full tree:

1. Find direct callers of `dangerous_write`.
2. For each, find *their* callers.
3. Repeat until you hit entry points (main, route handlers, tests).

LSP "call hierarchy" does this in IDEs. Manually: breadth-first, dedupe visited functions. Output is a tree, not a list.

## Semantic search — the fuzzy question

"Where do we handle session expiry?" — no single symbol. The code might say `timeout`, `ttl`, `expires_at`, `staleness`, `max_age`. Semantic search embeds code and query, ranks by meaning.

When you don't have a semantic search index, approximate:
1. Brainstorm synonyms: `expir`, `timeout`, `ttl`, `stale`, `max_age`.
2. Grep for each, union results.
3. Rank by proximity to other clue words (`session`, `auth`, `cookie`).

## Worked example — a real question

**Q:** "Where in this Django app do we actually write to the `orders` table?"

**Wrong first move:** grep `orders` — 847 hits, mostly templates and tests.

**Right sequence:**

1. **Find the model.** `rg 'class.*Model.*orders' -t py` or `rg "db_table.*orders"` → `Order` in `models/order.py`.
2. **Find writes.** ORM writes are `.save()`, `.create()`, `.update()`, `.delete()`, `.bulk_create()`. But those are on *any* model — need to narrow.
3. **Structural:** `rg 'Order\.(objects\.)?(create|update|bulk)' -t py` + `rg 'order\.save\(\)'` (instance-level, harder — `order` could be any variable name).
4. **Cross-reference:** find functions that take an `Order` and call `.save()`. `rg 'def.*order.*:' -A 20 | rg save`.
5. **Raw SQL escape hatch:** `rg 'INSERT INTO orders|UPDATE orders' -i` — catches anyone bypassing the ORM.

**Result:** 6 write sites. 4 through the ORM (service layer), 1 in a migration, 1 raw SQL in a management command (flagged — why is this bypassing the ORM?).

## Do not

- **Do not** grep when the question is transitive. "Who calls X" (direct) is grep. "Who eventually calls X" is a graph walk.
- **Do not** trust grep for "all usages" in dynamically-typed languages. `getattr(obj, 'foo')()` won't match `rg foo\(`. Know your language's reflection escape hatches.
- **Do not** semantic-search when you have an exact symbol. It's slower and less precise than grep.
- **Do not** present 847 hits. Filter, rank, group. "Here are 847 matches" is not an answer.

## Output format

```
## Query
<what was asked>

## Search strategy
<text | AST | call-graph | semantic> — <why this one>

## Searches run
1. <command / pattern> → <N> hits
2. <refinement> → <M> hits
...

## Results (ranked)
| Location | Snippet | Relevance |
| -------- | ------- | --------- |

## Notes
<known blind spots — reflection, generated code, dynamic dispatch>
```
