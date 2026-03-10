---
name: module-level-code-translator
description: Translates an entire module or package between languages, handling imports, file layout, visibility, and cross-function dependencies that single-function translation misses. Use when porting a library, when a migration spans multiple files, or when the user hands you a directory and a target language.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "code-translation, test-guided-migration-assistant, dependency-resolver"
---

# Module-Level Code Translator

Single-function translation (→ `code-translation`) is clean. Module translation is messy: imports, circular deps, visibility, file boundaries, and the build system all get involved. This is about the **glue**.

## Module concerns that function-level misses

| Concern                  | Why it matters                                                              |
| ------------------------ | --------------------------------------------------------------------------- |
| Import/export structure  | Python `__init__.py` ≠ Java package ≠ Go package ≠ Rust `mod`. What's public? |
| File ↔ unit mapping      | Java: one public class per file. Python: anything goes. Go: package = dir.  |
| Circular imports         | Python tolerates them at runtime. Go/Rust don't compile. Java barely does.  |
| Visibility               | Python `_private` convention. Java `private`/`package`/`protected`/`public`. Go capitalization. Rust `pub`/`pub(crate)`. |
| Module-level state       | Python module-level vars are singletons. Java needs a class. Go `var`. Rust `static`/`lazy_static`. |
| Build integration        | `setup.py`/`pyproject.toml` → `pom.xml`/`build.gradle` → `go.mod` → `Cargo.toml` |

## Step 1 — Build the module dependency graph

Before translating anything, map out what depends on what **within the module**:

```
  util.py ──────────┐
     │              ▼
     └──────► parser.py ──► api.py
                   ▲            │
  config.py ───────┘            │
                                ▼
                           __init__.py (re-exports)
```

This tells you:
- **Translation order**: leaves first (`util`, `config`), then dependents.
- **Circular deps**: if `parser` imports `api` and vice versa — break the cycle first, it won't compile in Go/Rust.
- **Public surface**: what `__init__.py` exports is your API contract. Everything else can be restructured freely.

## Step 2 — Decide the target file layout

Don't mirror the source layout blindly — match the target language's conventions:

| Source (Python)           | Java                                    | Go                                  | Rust                              |
| ------------------------- | --------------------------------------- | ----------------------------------- | --------------------------------- |
| `mymod/__init__.py`       | Package `com.x.mymod` (no file)         | Package `mymod` (dir)               | `mymod/mod.rs` or `mymod/lib.rs`  |
| `mymod/util.py` (5 funcs) | 5 files? Or one `Util.java` static class| `util.go` in package `mymod`        | `util.rs` as `mod util`           |
| `mymod/_internal.py`      | Package-private classes                 | Lowercase names (unexported)        | No `pub` — crate-private          |
| Module-level `CACHE = {}` | Static field in a class                 | Package-level `var cache = ...`     | `static CACHE: Lazy<...> = ...`   |

**In Go:** everything in one directory is one package. Your five Python files might become five `.go` files — but they all share one namespace. No import between them.

**In Java:** one public class per file is enforced. Multi-class Python files split.

## Step 3 — Translate leaves first

Walk the dep graph bottom-up. For each file:

1. Translate functions per → `code-translation`.
2. Translate module-level constants, state, initialization.
3. Fix up imports to point at already-translated siblings.
4. **Compile/run it.** Don't move on until this file builds. Errors compound.

## Step 4 — Verify at the seams

Function-level tests pass. Now test **cross-function behavior**:

- A calls B, B returns something A uses — does the type/shape still match?
- Module-level init order — Python runs module bodies at import. Java runs static initializers at class load. Go runs `init()`. Different timing.
- Error propagation across function boundaries — if `B` now returns `error` instead of raising, every `A`-calls-`B` site changed.

→ `test-guided-migration-assistant` if you have tests for the original: they become the oracle.

## Worked example — Python package → Go

**Source structure:**
```
retry/
  __init__.py    # from .policy import *; from .backoff import exponential
  policy.py      # class RetryPolicy; uses backoff
  backoff.py     # def exponential(n), def linear(n)
```

**Dep graph:** `backoff` (leaf) ← `policy` ← `__init__`.

**Target Go layout:**
```
retry/
  backoff.go     # Exponential(n), linear(n)   — linear unexported
  policy.go      # RetryPolicy struct + methods
  retry.go       # package doc comment — no re-exports needed (one package)
```

All three files are `package retry`. No imports between them — they share a namespace. `linear` becomes lowercase `linear` (was only used by `policy`, not in `__init__`'s public exports). `exponential` → `Exponential` (exported).

**Gotcha found during translation:** `RetryPolicy.__init__` takes `backoff=exponential` — a function default arg. Go has no default args. Options: zero-value meaning "use exponential," or functional options pattern. Chose zero-value — simpler, matches `RetryPolicy{}` idiom.

## Do not

- **Do not** translate file-by-file in source order. Translate in dep order — leaves first.
- **Do not** preserve circular imports. They're a design smell in Python and a hard error in most targets. Break them (extract a third module, invert a dependency) before translating.
- **Do not** blindly mirror the source's public surface. `_foo` in Python is convention; in Go, lowercase is enforced. Decide what's actually API.
- **Do not** skip building after each file. Module translations fail at integration, not inside functions. Catch it early.

## Output format

```
## Source → target
<src lang/version> → <target lang/version>

## Dependency graph
<ascii or list — who imports whom>

## File layout mapping
| Source file | Target file(s) | Notes |
| ----------- | -------------- | ----- |

## Public API (preserved)
<what callers see — unchanged>

## Internal restructuring
<what moved/renamed internally, and why>

## Per-file translation
### <file>
<code>
Deltas: <semantic notes>

## Integration checklist
- [ ] Builds
- [ ] Cross-function tests pass
- [ ] Init order verified
```
