---
name: code-pattern-extractor
description: Identifies recurring structural patterns in a codebase — idioms, copy-paste clones, homegrown abstractions — and characterizes each as a reusable template. Use when learning a codebase's conventions, when hunting for copy-paste that should be a function, or when documenting how this team does things.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "code-search-assistant, code-smell-detector, code-refactoring-assistant"
---

# Code Pattern Extractor

Every codebase has patterns — some intentional (house idioms), some accidental (copy-paste). Finding them tells you *how code gets written here*, and which duplication should be consolidated.

## Three pattern types

| Type              | What it is                                       | What to do with it                        |
| ----------------- | ------------------------------------------------ | ----------------------------------------- |
| **Idiom**         | The team's standard way of doing X               | Document it. New code should follow it.   |
| **Clone**         | Copy-pasted code with minor tweaks               | Extract to a function. → `code-refactoring-assistant` |
| **Anti-pattern**  | A recurring mistake                              | Flag it. → `code-smell-detector`          |

The same structural pattern can be any of the three — it depends on whether the repetition is good, accidental, or bad.

## Finding clones — structural similarity

Exact-text clones are easy (`rg` + sort + uniq -c). **Near-clones** — same structure, different variable names — need normalization:

1. **Tokenize** each function.
2. **Normalize:** replace identifiers with placeholders (`$1`, `$2`, ...), normalize literals (`42` → `NUM`, `"foo"` → `STR`).
3. **Hash** the normalized token stream.
4. **Group** by hash. Collisions are clone candidates.

```
Original A:   user = db.get(user_id);  if user is None: raise NotFound("user")
Original B:   order = db.get(order_id); if order is None: raise NotFound("order")

Normalized:   $1 = db.get($2);         if $1 is None: raise NotFound(STR)
              $1 = db.get($2);         if $1 is None: raise NotFound(STR)

→ Same hash. Clone pair found.
```

**Extractable as:** `def get_or_404(model, id, name): ...`

## Finding idioms — frequency of small patterns

Idioms are short (2–5 lines) and appear *everywhere*. Mine them by n-gram frequency on normalized AST nodes:

| Normalized n-gram                             | Count | Interpretation                           |
| --------------------------------------------- | ----- | ---------------------------------------- |
| `if $1 is None: return None`                  | 89    | Null-propagation idiom — this codebase uses it heavily |
| `with self._lock: $BODY`                      | 34    | Locking idiom — `_lock` is the house convention |
| `logger.info(f"{$1.__class__.__name__}: ...")`| 27    | Logging idiom — always includes class name |
| `try: $X except Exception: pass`              | 11    | **Anti-pattern** — swallowing all exceptions |

The count tells you: high-count idioms are conventions to follow; high-count anti-patterns are systemic problems.

## Worked example — extracting a house idiom

**Observed in 23 places:**

```python
def get_foo(self, foo_id):
    resp = self._client.get(f"/foos/{foo_id}")
    resp.raise_for_status()
    data = resp.json()
    return Foo.from_dict(data)

def get_bar(self, bar_id):
    resp = self._client.get(f"/bars/{bar_id}")
    resp.raise_for_status()
    data = resp.json()
    return Bar.from_dict(data)
# ... ×21 more
```

**Pattern template:**

```
def get_$RESOURCE(self, ${RESOURCE}_id):
    resp = self._client.get(f"/${RESOURCE}s/{${RESOURCE}_id}")
    resp.raise_for_status()
    return $MODEL.from_dict(resp.json())
```

**Parameters:** `$RESOURCE` (string, e.g. `foo`), `$MODEL` (class, e.g. `Foo`).

**Verdict:** This is a **clone, not an idiom.** 23 copies of the same 4 lines with 2 parameters → extract:

```python
def _get_resource(self, path: str, model: type[T]) -> T:
    resp = self._client.get(path)
    resp.raise_for_status()
    return model.from_dict(resp.json())

def get_foo(self, foo_id): return self._get_resource(f"/foos/{foo_id}", Foo)
```

23 × 4 lines → 23 × 1 line + 4 lines shared. And now error handling changes in one place.

## Distinguishing idiom from clone

| Signal                                  | Points to idiom      | Points to clone           |
| --------------------------------------- | -------------------- | ------------------------- |
| Pattern length                          | Short (2–3 lines)    | Long (5+ lines)           |
| Parameter count                         | 0–1                  | 2+                        |
| Repetition within one file              | Rare                 | Common                    |
| Language/framework requires this shape  | Yes — it's idiom     | No — it's duplication     |
| Would extraction make callsites clearer?| No — idiom reads fine | Yes — callsite becomes a name |

`with self._lock:` (1 line, 0 params, language-required shape) → idiom.
The `get_$RESOURCE` block above (4 lines, 2 params, nothing requires this shape) → clone.

## Do not

- **Do not** extract every 2-line pattern. `if x is None: return None` appearing 89 times isn't duplication — it's an idiom, and extracting it to `propagate_none(x)` makes code *less* readable.
- **Do not** report clone pairs without the extraction proposal. "These two functions are similar" is not actionable. "Extract this helper" is.
- **Do not** ignore the parameter count. A pattern with 6 parameters that differ each time isn't extractable — the "common" part is tiny.
- **Do not** miss semantic clones that differ textually. `if not user` vs `if user is None` — different text, same pattern. Normalize aggressively.

## Output format

```
## Idioms (follow these)
| Pattern | Count | Example location |
| ------- | ----- | ---------------- |

## Clones (extract these)
### <pattern name>
Occurrences: <N>
Template:
<normalized pattern with $PARAMS>
Parameters: <list — what varies>
Proposed extraction:
<function signature + body>
Affected files: <list>

## Anti-patterns (fix these)
| Pattern | Count | Why bad | Locations |
| ------- | ----- | ------- | --------- |
```
