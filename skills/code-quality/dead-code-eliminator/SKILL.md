---
name: dead-code-eliminator
description: Finds and safely removes code that is never executed — unreachable branches, uncalled functions, unused classes, dead feature flags. Use when cleaning up after a feature removal, when the user suspects the codebase has accumulated cruft, or when reducing build/bundle size.
license: Apache-2.0
metadata:
  category: "code-quality"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "code-refactoring-assistant, behavior-preservation-checker"
---

# Dead Code Eliminator

Dead code is code that **cannot execute** under any input. It's not "rarely used" — it's *never* reachable. The distinction matters: deleting rarely-used code is a product decision; deleting dead code is housekeeping.

## Deadness classes — from certain to uncertain

| Class                            | How to prove dead                                               | Confidence |
| -------------------------------- | --------------------------------------------------------------- | ---------- |
| Code after `return`/`throw`/`exit` | Syntactic — compiler/linter catches this                       | Certain    |
| Private function, zero callers   | Grep the module. Private = no external callers possible.        | Certain    |
| Branch with constant-false condition | `if False:`, `if (1 == 2)`. Constant folding.              | Certain    |
| Public function, zero callers in repo | Grep. But: reflection, plugins, DI, external callers?       | **Uncertain** — see below |
| Feature flag hard-wired off      | `if config.NEW_CHECKOUT:` and the flag is `False` everywhere, permanently | High (check all envs first) |
| `else` branch of exhaustive match | All enum values covered above; `else` can't fire              | High (until someone adds an enum value) |
| Zero coverage in tests           | Covered-by-nobody. But tests don't cover everything users hit.  | **Low** — absence of test ≠ absence of use |

## The public-symbol problem

Static analysis says `doThing()` has zero callers. Before deleting, check:

- **Reflection / dynamic dispatch:** `getattr(obj, name)()`, `Class.forName(...)`, Spring `@Component` scanning. The call is in a string.
- **Entry points:** CLI scripts, web handlers, cron jobs — registered in config, not called from code.
- **External callers:** It's a library. Someone downstream imports this. Check `__all__`, check the public API docs.
- **Serialization hooks:** `__getstate__`, `toJSON()`, ORM hooks — called by the framework, not by you.
- **Tests only:** The only caller is a test. The test exists to test the function — circular. Delete both.

If you can't rule all of these out, it's *probably* dead but you can't *prove* dead. Deprecate before delete.

## Safe deletion workflow

```
1. Identify → 2. Verify → 3. Deprecate (if public/uncertain) → 4. Delete → 5. Test
```

| Step | Action                                                               |
| ---- | -------------------------------------------------------------------- |
| 1    | Find candidates (static analysis + coverage + grep)                  |
| 2    | For each: check the uncertainty table above                          |
| 3    | Uncertain → add a deprecation warning/log, deploy, wait one cycle. Silence = dead. |
| 4    | Delete the code AND its tests AND its imports                        |
| 5    | Full test suite. Then → `behavior-preservation-checker` on the containing module. |

## Worked example

**Candidate:** `utils/legacy_format.py` — 3 functions, 0% test coverage, no callers found by grep.

**Verify:**
- Private? No — top-level module functions.
- Reflection? `rg 'legacy_format' --type py` → one hit: `import utils.legacy_format` in `utils/__init__.py`. Nobody uses the import. Re-exported but unused.
- Entry point? Check `setup.py` / `pyproject.toml` `[project.scripts]` — not there.
- External? This is an app, not a library. No downstream.
- Serialization? Function names are `parse_v1`, `parse_v2`, `detect_version` — not hook-shaped.

**Verdict:** Dead. But the `__init__.py` re-export means someone *thought* it was public. Check git log: last non-formatting commit to `legacy_format.py` was 3 years ago, message says "migrate to v3 format." The v3 migration finished; v1/v2 parsers are orphaned.

**Delete:**
1. Remove the `__init__.py` re-export. Test. Green.
2. Delete `utils/legacy_format.py`. Test. Green.
3. `rg 'legacy_format'` one more time — clean.

Commit message: `Remove dead v1/v2 format parsers — v3 migration completed 2023, no remaining callers`.

## Dead feature flags

A flag that's been at one value in every environment for 3+ months is a dead conditional:

```python
if settings.USE_NEW_CHECKOUT:
    new_path()
else:
    old_path()     # ← dead if flag has been True everywhere since Q3
```

Delete the flag check, inline the winning branch, delete the losing branch. Then delete the flag from config. This is the #1 source of dead code in feature-flag-heavy codebases.

## Do not

- **Do not** delete based on zero coverage alone. Tests are not exhaustive. Production logs are closer — if you have them, check for the function's log lines.
- **Do not** delete a public symbol without a deprecation cycle unless you can *prove* no external callers.
- **Do not** delete the function but leave the test. Now you have a test for nothing — and it'll fail, and someone will "fix" it by re-adding the function.
- **Do not** delete an `else` branch just because the `if` is always true today. If the condition reads external config, "always true" is a current fact, not an invariant.
- **Do not** batch-delete 50 functions in one commit. One commit per logical dead-code cluster — if you're wrong about one, the revert is surgical.

## Output format

```
## Dead (certain — delete now)
- <file>:<symbol>  — <proof: private+no-callers | after-return | const-false>
  Also delete: <tests, imports>

## Probably dead (deprecate first)
- <file>:<symbol>  — <why uncertain: might-be-reflected | public-api>
  Deprecation: <log line / warning to add>
  Revisit after: <one release cycle>

## Not dead (looked dead, isn't)
- <file>:<symbol>  — <actual caller you found>
```
