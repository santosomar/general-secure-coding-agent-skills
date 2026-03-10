---
name: static-bug-detector
description: Identifies bugs through static code analysis (null dereferences, type mismatches, control flow issues) without executing the program. Use when scanning code for defects before running tests, when the user asks for static analysis, or when integrating with CI for defect detection.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "semantic-bug-detector, bug-localization"
---

# Static Bug Detector

Find bugs that are **syntactically provable** without running the code. This is the fast, shallow pass: it won't catch design bugs, but everything it catches is real (or should be — see FP suppression below).

## Signal catalog

| Defect class            | What to look for                                              | False-positive trap                              |
| ----------------------- | ------------------------------------------------------------- | ------------------------------------------------ |
| Null/undefined deref    | Path where `x` could be null and is dereferenced              | Framework guarantees non-null (Spring `@Autowired`, DI) |
| Uninitialized read      | Variable read before any assignment on some path              | Language zero-initializes (Go, Java fields)      |
| Dead store              | Assignment never read before reassign/return/scope-end        | Intentional — value used by debugger/reflection  |
| Unreachable code        | Statements after unconditional return/throw/break             | Intentional dead-switch-default for exhaustiveness |
| Resource leak           | `open`/`acquire` with no `close`/`release` on all exit paths  | Ownership transferred to caller (factory pattern) |
| Unchecked return        | `err`/`Result`/error-returning API call ignored               | Intentionally best-effort (`_ = file.Close()`)   |
| Always-true/false cond  | Condition provably constant via value-range or type           | Defensive belt-and-suspenders after earlier guard |
| Identical branches      | `if`/`else` bodies are syntactically identical                | Copy-paste placeholder during dev — rarely intentional |
| Self-assignment / no-op | `x = x`, `list.remove(x); list.add(x)` with no side effect    | Rarely intentional; near-always a bug            |
| Format string mismatch  | `printf`/`format` arg count or type mismatch                  | Almost never a false positive                    |
| Integer over/underflow  | Arithmetic that provably exceeds type range                   | Intentional wraparound (hash, checksum)          |

## Ranking heuristic

Sort findings by `severity × confidence × reachability`:

- **Severity:** crash (3) > data corruption (3) > leak (2) > dead code (1)
- **Confidence:** intraprocedural proof (1.0) > interprocedural with summaries (0.7) > heuristic pattern (0.4)
- **Reachability:** in a hot path / entry point (1.0) > library code (0.6) > test code (0.2)

## FP suppression

Before reporting, check each finding against the FP-trap column above. Then:

1. **Grep for suppression comments** near the finding (`// NOSONAR`, `# noqa`, `// SAFETY:`). If someone already justified it, lower confidence to 0.
2. **Check the type system**. If the language guarantees the property (Rust borrow checker, TypeScript `strictNullChecks`), you're fighting the compiler.
3. **Look for a test that exercises this path** and passes. If covered and green, the "bug" might be a spec, not a defect.

## Worked example

**Code:**

```java
public String getDisplayName(User u) {
    if (u.getNickname() != null) {
        return u.getNickname();
    }
    return u.getFullName().toUpperCase();   // line 12
}
```

**Finding:**

```
src/User.java:12  NULL_DEREFERENCE  severity=high  confidence=0.7
  `u.getFullName()` may return null (no @NonNull annotation, and UserBuilder
  permits `fullName` to be unset), and `.toUpperCase()` would NPE.

  Callers checked: 3/3 pass non-null `u`, so the outer `u` is safe.
  But no caller guarantees `u.fullName` is set.

  Suggested fix:
    return Optional.ofNullable(u.getFullName()).map(String::toUpperCase).orElse("");
```

## Edge cases

- **Generated code:** Suppress. Nobody will fix `*.pb.go` or `*_generated.ts`. Report the generator config instead if the pattern is dangerous.
- **Dynamic languages with no type info:** Fall back to name heuristics (`xOrNull`, `maybe_x`, `Optional[X]` in docstrings) and flow-sensitive narrowing. Accept the lower confidence.
- **Macros / metaprogramming:** Analyze the *expansion*, not the source. If you can't expand, report as `confidence=low` and note why.

## Do not

- **Do not** report dead code in `if DEBUG:` or `#ifdef` blocks as bugs. It's conditional, not dead.
- **Do not** flag a null-check *before* a deref as "redundant" just because you also see a deref *after*. The check might be the only thing saving the deref.
- **Do not** report findings in vendored/third-party directories. Nobody will fix `node_modules/`.
- **Do not** emit more than ~50 findings per run without also emitting a summary histogram. Walls of findings get ignored.

## Output format

Emit SARIF if the consumer is CI. Otherwise:

```
<file>:<line>  <RULE_ID>  severity=<low|med|high>  confidence=<0.0-1.0>
  <one-line explanation of why this is a bug>
  <one-line suggested fix, or "manual review required">
```
