---
name: specification-to-temporal-logic-generator
description: Translates specifications into temporal logic formulas (LTL, CTL, or TLA) by matching the specification's shape to the right logic and operators. Use when formalizing requirements for any model checker, when choosing between LTL and CTL for a property, or when the user has a temporal claim and doesn't know which operators express it.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "requirement-to-tlaplus-property-generator, smv-model-extractor"
---

# Specification → Temporal Logic Generator

Three logics, one job: saying precisely what "always" and "eventually" mean. The choice of logic is mostly dictated by your checker; the choice of *formula* is the craft.

## Which logic?

| Logic | Time model              | Checker           | When it's the right fit                          |
| ----- | ----------------------- | ----------------- | ------------------------------------------------ |
| LTL   | Linear — one future     | Spin, NuSMV, most | "On every execution, X eventually holds"         |
| CTL   | Branching — many futures| NuSMV, nuXmv      | "There exists an execution where..." / "on all branches..." |
| TLA   | Linear + actions        | TLC, TLAPS        | Distributed systems; → `requirement-to-tlaplus-property-generator` |

**LTL vs CTL:** LTL quantifies over *paths* implicitly (every path). CTL quantifies explicitly (`A` = all paths, `E` = some path). The formula `AG EF reset` ("from any state, it's possible to reach reset") is CTL-only — LTL can't say "possible."

## Operator cheat sheet

| English                | LTL        | CTL           | Intuition                                       |
| ---------------------- | ---------- | ------------- | ----------------------------------------------- |
| Always P               | `G P`      | `AG P`        | P holds in every state (of every path)          |
| Eventually P           | `F P`      | `AF P` / `EF P` | P holds in some future state                  |
| Next P                 | `X P`      | `AX P` / `EX P` | P holds in the very next state                |
| P until Q              | `P U Q`    | `A[P U Q]`    | P keeps holding until Q does (and Q does happen)|
| P weak-until Q         | `P W Q`    | —             | P until Q, OR P forever (Q optional)            |
| Infinitely often P     | `G F P`    | `AG AF P`     | P keeps recurring                               |
| Eventually always P    | `F G P`    | `AF AG P`     | P stabilizes — becomes permanently true         |
| It's possible to reach P | —        | `EF P`        | **CTL-only.** Some path reaches P.              |
| P is always reachable  | —          | `AG EF P`     | **CTL-only.** From anywhere, P is still possible.|

## Pattern → formula

| Spec pattern                                  | Formula                      | Logic |
| --------------------------------------------- | ---------------------------- | ----- |
| "P is an invariant"                           | `G P`                        | LTL   |
| "Every request gets a response"               | `G (req → F resp)`           | LTL   |
| "P never happens before Q"                    | `¬P W Q`                     | LTL   |
| "Between Q and R, P always holds"             | `G (Q → (P W R))`            | LTL   |
| "The system can always be reset"              | `AG EF reset`                | CTL   |
| "There's no deadlock"                         | `AG EX true`                 | CTL   |
| "Once P, always P" (stable)                   | `G (P → G P)`                | LTL   |
| "P and Q alternate"                           | `G (P → X(¬P U Q)) ∧ G (Q → X(¬Q U P))` | LTL |

## Worked example

**Spec:** "After a successful login, the session token remains valid until logout or timeout, and it must become invalid immediately after either."

**Decompose:**
- "After login" → scope starts at `login_ok`
- "remains valid until" → `U` (strong until — one of the terminators must happen)
- "logout or timeout" → `logout ∨ timeout`
- "immediately after" → `X` (next state)

**LTL:**

```
G ( login_ok →
    ( valid U (logout ∨ timeout) )         -- valid until one fires
    ∧
    ( (logout ∨ timeout) → X ¬valid )      -- and then immediately invalid
  )
```

**Caveat:** The second conjunct is subtle — it says "whenever logout/timeout is true, next state is invalid." But that's global, not scoped to this session. If logout can happen without a preceding login, the formula is too strong. Tighter:

```
G ( login_ok →
    ( valid ∧ ¬logout ∧ ¬timeout ) U ( (logout ∨ timeout) ∧ X ¬valid )
  )
```

Now the "and then invalid" is part of the Until's right-hand side — it only applies to *this* session's termination.

## The vacuity check

After writing a formula, ask: **is it trivially true?**

- `G (false → P)` is always true — `false` never holds.
- `G (req → F resp)` — if `req` never occurs in the model, this passes. Meaningless.
- `F P` — if `P` is true in the initial state, this says nothing about the future.

Run the checker with a "witness" property too: `F req` should be satisfiable, confirming `req` actually happens.

## Do not

- **Do not** use LTL for "it's possible that." LTL can only say "on all paths" — possibility needs CTL's `E`.
- **Do not** confuse `G F P` with `F G P`. "Infinitely often" ≠ "eventually always." A blinking light satisfies `G F on` but not `F G on`.
- **Do not** use `U` when you mean `W`. `P U Q` *requires* Q to eventually hold. If Q might never happen and that's okay, use `W` (weak until).
- **Do not** nest more than 3 temporal operators without a sanity check. Deep nesting is where formulas go wrong — unfold on a small trace by hand.

## Output format

```
## Spec (as given)
<verbatim>

## Logic
<LTL | CTL | TLA> — <why this logic>

## Formula
<temporal logic>

## Decomposition
<english phrase> ↦ <sub-formula>

## Vacuity check
<witness property that should also hold — confirms the antecedent fires>
```
