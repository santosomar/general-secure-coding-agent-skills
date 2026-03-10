---
name: dependency-resolver
description: Diagnoses and resolves package dependency conflicts — version mismatches, diamond dependencies, cycles — across npm, pip, Maven, Cargo, and similar ecosystems. Use when install fails with a resolution error, when two packages require incompatible versions of a third, or when upgrading one dependency breaks another.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "build-ci-migration-assistant"
---

# Dependency Resolver

Dependency conflicts are a constraint satisfaction problem. The manifest is a set of version constraints; the resolver tries to pick one version per package that satisfies all of them. When it can't, you have to change a constraint.

## Conflict types

| Conflict                | Symptom                                              | Root cause                                  |
| ----------------------- | ---------------------------------------------------- | ------------------------------------------- |
| Diamond                 | A needs X@2, B needs X@1, you need A and B           | Transitive deps disagree on a shared dep    |
| Upper-bound collision   | A needs X<3, you need X>=3                           | A pinned too tightly, hasn't updated        |
| Peer dep mismatch       | npm: "X@2 requires peer Y@^1 but Y@2 is installed"   | Peer deps — caller must provide, and did wrong |
| Cycle                   | A depends on B depends on A                          | Bad package design (rare in published pkgs, common in monorepos) |
| Yanked / missing        | Version in lockfile no longer on registry            | Upstream pulled it (security or mistake)    |

## Step 1 — See the full tree

Before fixing, understand. Every ecosystem has a tree command:

| Ecosystem | Tree command                                      |
| --------- | ------------------------------------------------- |
| npm       | `npm ls <pkg>` or `npm ls --all`                  |
| pip       | `pipdeptree` or `pip show <pkg>` for one level    |
| Maven     | `mvn dependency:tree -Dincludes=<group>:<artifact>` |
| Gradle    | `gradle dependencies --configuration runtimeClasspath` |
| Cargo     | `cargo tree -i <pkg>` (inverted — who depends on this) |
| Go        | `go mod graph | grep <module>`                    |

Find **every path** to the conflicting package. The conflict is at the merge point.

## Step 2 — Diagnose the diamond

```
  your-app
  ├── framework@5.0  ── requests@^2.28
  └── legacy-sdk@1.2 ── requests@>=2.20,<2.26
```

`requests@^2.28` means `>=2.28, <3.0`. Intersection with `>=2.20,<2.26` is **empty**. No version satisfies both.

**Who's wrong?** Usually the upper-bounded one (`legacy-sdk`). Upper bounds on deps are almost always too pessimistic — `legacy-sdk` probably works fine with `requests@2.28` but the author pinned defensively.

## Step 3 — Resolution strategies, cheapest first

| Strategy                      | When                                               | Risk                                   |
| ----------------------------- | -------------------------------------------------- | -------------------------------------- |
| Update the stale dep          | `legacy-sdk@1.3` might have relaxed the bound      | None if it exists — check changelog    |
| Override / force resolution   | Tell the resolver "use X@2.28 anyway"              | `legacy-sdk` might actually break      |
| Fork and patch                | Fork `legacy-sdk`, bump its requests bound, publish privately | Maintenance burden         |
| Vendor / inline               | Copy `legacy-sdk` into your repo, drop its requests dep | Loses upstream updates          |
| Remove one side               | Do you actually need `legacy-sdk`?                 | Feature loss                           |

**Ecosystem-specific overrides:**

| Ecosystem | Override mechanism                                              |
| --------- | --------------------------------------------------------------- |
| npm       | `overrides` in package.json (npm 8+), or `resolutions` (yarn)   |
| pip       | No first-class override — use constraints file, or fork         |
| Maven     | `<dependencyManagement>` forces a version for all transitives   |
| Gradle    | `resolutionStrategy.force 'group:artifact:version'`             |
| Cargo     | `[patch.crates-io]` section — replace a dep with a fork         |
| Go        | `replace` directive in go.mod                                   |

## Worked example — npm diamond

**Error:**
```
npm ERR! ERESOLVE unable to resolve dependency tree
npm ERR! Found: react@18.2.0
npm ERR! Could not resolve dependency:
npm ERR! peer react@"^17.0.0" from old-chart-lib@2.1.0
```

**Tree:**
```
$ npm ls react
├── react@18.2.0                    ← your direct dep
└── old-chart-lib@2.1.0
    └── react@^17.0.0  (peer)       ← wants 17.x
```

**Diagnosis:** `old-chart-lib` declares `react@^17` as a peer dep — it was written for React 17, never updated.

**Resolution ladder:**

1. **Check for updates:** `npm view old-chart-lib versions` → `2.1.0` is latest. No fix upstream.
2. **Check compat:** Does `old-chart-lib` actually break on React 18? Read its React usage — if it uses no removed APIs, the peer bound is overly tight.
3. **Override:**
   ```json
   "overrides": { "old-chart-lib": { "react": "$react" } }
   ```
   `$react` means "use whatever version I have at the top level." Forces `old-chart-lib` to accept React 18.
4. **Test.** Run the app. Chart renders → the peer bound was indeed too tight. Done.
5. **If it breaks:** Fork `old-chart-lib`, fix the actual incompatibility (likely a removed lifecycle method), use `npm:@yourorg/old-chart-lib`.

## Cycles

In monorepos: `pkg-a` depends on `pkg-b` depends on `pkg-a`. Most resolvers handle this (npm, Cargo). Some don't (pip, without `-e`). Either way it's a smell.

Break the cycle by extracting the shared piece:
```
  pkg-a ─────► pkg-b        pkg-a ──► pkg-shared ◄── pkg-b
     ▲           │     →
     └───────────┘
```

Whatever `pkg-a` needs from `pkg-b` and vice versa — that's `pkg-shared`.

## Do not

- **Do not** use `--force` / `--legacy-peer-deps` as a permanent fix. It silences the resolver; it doesn't resolve anything. The conflict is still there at runtime.
- **Do not** delete the lockfile to "fix" resolution. You'll get *a* resolution, likely a different one each time. Lockfiles exist for reproducibility.
- **Do not** override without testing. The resolver's constraint was there for a reason — maybe a wrong reason, but you don't know until you run the code.
- **Do not** upper-bound your own deps unless you've confirmed incompatibility. `requests<3` in your package is how you become someone else's diamond problem.

## Output format

```
## Conflict
<resolver error, verbatim>

## Tree
<paths to the conflicting package — npm ls / cargo tree / etc output>

## Diagnosis
Conflicting constraints: <X requires Y@A> vs <Z requires Y@B>
Intersection: <empty | range>
Suspected over-constrainer: <which one's bound is probably wrong>

## Resolution
Strategy: <update | override | fork | vendor | remove>
Change:
<diff to manifest/lockfile>
Risk: <what might break — and how you verified it doesn't>

## Verification
<install succeeds, tests pass, feature using the overridden dep works>
```
