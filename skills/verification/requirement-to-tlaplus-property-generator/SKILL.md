---
name: requirement-to-tlaplus-property-generator
description: Translates natural-language requirements into TLA+ properties — invariants for safety, temporal formulas for liveness — checkable with TLC. Use when writing the PROPERTY and INVARIANT sections of a TLA+ spec, when formalizing acceptance criteria, or when the user has a requirement and a model but no property.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "specification-to-temporal-logic-generator, program-to-tlaplus-spec-generator, nl-to-constraints"
---

# Requirement → TLA+ Property Generator

A requirement is a claim about behavior. A TLA+ property is the same claim, stated precisely enough to check. The translation is about **finding the formal shape** of an informal statement.

## Step 1 — Classify: safety or liveness?

| The requirement says…                    | It is…     | TLA+ form                        |
| ---------------------------------------- | ---------- | -------------------------------- |
| "never", "always", "must not", "at most" | Safety     | Invariant: `Inv == ...`          |
| "eventually", "will", "responds", "terminates" | Liveness | Temporal: `Prop == <>...` or `[]<>...` |
| "if X then eventually Y"                 | Liveness (response) | `[](X => <>Y)` or `X ~> Y`  |
| "X until Y"                              | Safety-ish | Encoded with history variables or LTL-style |

Safety = something bad never happens. Checkable at each state.
Liveness = something good eventually happens. Needs fairness; checkable only over infinite traces.

## Step 2 — Find the state predicate

Every property bottoms out in a predicate over state variables. Extract the nouns from the requirement; map them to `VARIABLES`.

| Requirement noun              | Likely state variable              |
| ----------------------------- | ---------------------------------- |
| "the lock"                    | `lock_holder`                      |
| "the queue"                   | `queue` (a sequence)               |
| "process P"                   | `pc[P]` (its program counter)      |
| "a request"                   | An element of `msgs` or `pending`  |
| "the leader"                  | `leader` (or `\E p : role[p] = "leader"`) |

## Pattern catalog — NL → TLA+

| English                                         | TLA+                                                      |
| ----------------------------------------------- | --------------------------------------------------------- |
| "At most one process holds the lock"            | `Cardinality({p \in Proc : holding[p]}) <= 1`             |
| "The queue never exceeds N"                     | `Len(queue) <= N`                                         |
| "If a process is in the critical section, it holds the lock" | `\A p : pc[p] = "cs" => lock_holder = p`       |
| "Every request is eventually acknowledged"      | `\A r \in Req : (r \in pending) ~> (r \in acked)`         |
| "The system never deadlocks"                    | `[]<><<Next>>_vars` (always eventually a step) — or check in `.cfg` with `CHECK_DEADLOCK` |
| "X and Y are never true simultaneously"         | `~(X /\ Y)`                                               |
| "Once elected, a leader stays leader"           | `[](leader = p => [](leader = p))` — or with action: `[][leader' = leader \/ leader = NULL]_leader` |
| "P always precedes Q"                           | Tricky — use a history variable: `seen_P => Q` won't fire before `P` |

## Worked example

**Requirement:** "A write request must be replicated to a majority of nodes before it is acknowledged to the client."

**Decompose:**
- "A write request" → some `r \in writes`
- "replicated to N" → `replicated[r]` is a set of nodes, `Cardinality(replicated[r]) >= ...`
- "majority" → `> Cardinality(Nodes) \div 2`
- "before acknowledged" → if `r \in acked` then the replication condition holds. This is an ordering constraint on *when* `acked` can be set.

**Property (safety — "never ack prematurely"):**

```tla
Majority == Cardinality(Nodes) \div 2 + 1

SafeAck ==
  \A r \in acked :
    Cardinality(replicated[r]) >= Majority
```

Checkable as an invariant. If TLC finds a state where `r \in acked` but `replicated[r]` is a minority, the replication protocol is wrong.

**Companion liveness — "every write does eventually get acked":**

```tla
AckLiveness ==
  \A r \in writes : (r \in pending) ~> (r \in acked)
```

Needs fairness on the replication and ack actions to be checkable.

## The precedence trap

"X before Y" is NOT `X => <>Y`. That says "if X, eventually Y" — nothing about order.

"X before Y" means: at the moment Y happens, X has already happened. In state terms: `Y => (X has been true at some past point)`. TLA+ is forward-looking — past is awkward. Standard trick:

```tla
VARIABLE seen_X    \* history variable — not in the real system
Init == ... /\ seen_X = FALSE
Next == ... /\ seen_X' = (seen_X \/ X)   \* latches TRUE once X holds

Precedence == Y => seen_X
```

## Do not

- **Do not** translate "eventually" without fairness. `<>P` is only checkable if `Next` has fairness — otherwise TLC says "yes, if I stutter forever P never happens, property violated." Add `WF_vars(Action)` for the action that achieves P.
- **Do not** write `[]<>P` when you mean `<>P`. `[]<>P` = "infinitely often P" (keeps happening). `<>P` = "at least once." Different requirements.
- **Do not** state liveness properties you can't check. TLC checks liveness but it's expensive. If your model is already at the state-space limit for safety, liveness won't finish.
- **Do not** let a requirement stay ambiguous. "The system is consistent" — consistent *how*? Push back for a precise statement before formalizing.

## Output format

```
## Requirement (as given)
<verbatim>

## Classification
<safety | liveness | response | precedence>

## State mapping
<english noun> ↦ <TLA+ variable/expression>

## Property
<TLA+>

## Fairness needed (liveness only)
<WF/SF conditions — and why>

## Sanity check
<does the property trivially hold? is it vacuously true? — one sentence>
```
