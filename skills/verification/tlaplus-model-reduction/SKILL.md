---
name: tlaplus-model-reduction
description: Reduces a TLA+ model so TLC can actually check it — shrinks constants, adds state constraints, abstracts data, or applies symmetry — when the state space is too large to enumerate. Use when TLC runs out of memory, when checking takes hours, or when a spec works at N=2 and you need confidence at larger scale.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "program-to-tlaplus-spec-generator, requirement-to-tlaplus-property-generator"
---

# TLA+ Model Reduction

TLC is explicit-state: it literally visits every reachable state. State count is roughly **product of each variable's domain size, raised to how many variables you have, times fanout.** When that product hits billions, you're done — unless you shrink something.

## Diagnose first — where's the blowup?

Run TLC with `-coverage 1`. Look at:
- **Distinct states** vs **states generated** — high ratio of generated/distinct means lots of redundant exploration (symmetry helps).
- **States per action** — one action dominates? That's your bottleneck.
- **Queue size** — growing unboundedly? You have an infinite state space.

| Symptom                                  | Likely cause                      | Fix                                |
| ---------------------------------------- | --------------------------------- | ---------------------------------- |
| Runs forever, queue keeps growing        | Unbounded domain (Nat, Seq)       | State constraint                   |
| Huge distinct-states count, slow         | Too many processes/nodes          | Shrink CONSTANTS                   |
| generated/distinct ratio very high       | Symmetric processes               | SYMMETRY set                       |
| One variable has huge domain             | Over-precise data modeling        | Abstract the data                  |

## Reduction techniques — cheapest first

### 1. Shrink CONSTANTS

The `.cfg` constants control scale. `Proc = {p1, p2, p3, p4, p5}` → try `{p1, p2, p3}`. Most bugs show at N=2 or N=3.

**Small model hypothesis:** if a protocol bug exists, it usually exists at small scale. Paxos bugs manifest with 3 nodes. Consensus needs 3 for a meaningful majority; 5 rarely finds new bugs.

### 2. State constraint — bound the unbounded

Any variable ranging over `Nat` or unbounded sequences makes the state space infinite. Add a `CONSTRAINT` in `.cfg`:

```
CONSTRAINT StateConstraint
```

```tla
StateConstraint ==
  /\ counter <= 10
  /\ Len(queue) <= 5
  /\ clock <= 20
```

TLC stops exploring successors of any state violating the constraint. **This is not a proof of safety** — it's a search within a bounded region. But if the property holds up to the bound and the bound is well above any threshold in the spec, that's strong evidence.

### 3. SYMMETRY — collapse equivalent states

If processes are interchangeable (any permutation of `{p1, p2, p3}` gives the same behavior), declare it:

```
SYMMETRY Perms
```

```tla
Perms == Permutations(Proc)
```

TLC now treats `(p1 leader, p2 follower, p3 follower)` and `(p2 leader, p1 follower, p3 follower)` as the **same state**. For N symmetric processes, this divides state count by roughly N!.

**Warning:** Symmetry is unsound with liveness checking. TLC will warn you. Safety only.

### 4. Abstract the data

If a message payload doesn't affect the protocol, don't model it. Replace `value \in Int` with `value \in {v1, v2}` — two distinct symbolic values are enough to distinguish "same" from "different."

| Over-modeled                              | Abstracted                                              |
| ----------------------------------------- | ------------------------------------------------------- |
| `msg.body \in STRING`                     | Drop `.body` — protocol doesn't inspect it              |
| `timestamp \in Nat`                       | `timestamp \in 0..3` with a constraint, or just ordering (`t1 < t2`) |
| `balance \in Int`                         | `balance \in {Neg, Zero, Pos}` if only sign matters     |

### 5. View reduction — coarser state equality

A `VIEW` tells TLC which part of the state to hash for "already seen" checks:

```
VIEW StateView
```

```tla
StateView == <<pc, lock_holder>>    \* ignore tmp, counter for dedup purposes
```

States that differ only in un-viewed variables are treated as equal. **Dangerous:** you can miss bugs in the ignored variables. Use only when you're *sure* those variables don't affect the property.

## Worked example — before/after

**Spec:** Leader election among N nodes. Original `.cfg`:

```
CONSTANTS Nodes = {n1, n2, n3, n4, n5}
          MaxRound = 100
```

TLC: 47M distinct states after 2 hours, still growing. Queue at 3M.

**Diagnosis:** `round \in Nat` is unbounded → infinite. Nodes are symmetric. 5 nodes is overkill.

**Reduced `.cfg`:**

```
CONSTANTS Nodes = {n1, n2, n3}
          MaxRound = 4

CONSTRAINT StateConstraint
SYMMETRY NodePerms
```

```tla
StateConstraint == round <= MaxRound
NodePerms == Permutations(Nodes)
```

TLC: 18K distinct states, finishes in 12 seconds. Invariant holds.

**Is this enough?** 3 nodes is the minimum for a meaningful majority. `MaxRound = 4` — the protocol's timeout threshold is 2, so 4 rounds is two full timeout cycles. Any round-based bug would fire by then. Strong evidence, not proof.

## When reduction isn't enough

If you've shrunk to N=2 with tight constraints and it's *still* too big, the spec has too much internal state. Options:

- **Decompose.** Check components independently with assume/guarantee: assume the environment behaves, prove this component does.
- **Switch to inductive invariant proving.** Instead of enumerating states, prove `Init ⟹ Inv` and `Inv ∧ Next ⟹ Inv'` with TLAPS. Harder to write, scales infinitely.
- **Simulate, don't model-check.** Run TLC in simulation mode (`-simulate`) — random walks through the state space. Doesn't prove anything but finds bugs.

## Do not

- **Do not** use SYMMETRY when checking liveness. It's unsound — TLC will silently miss violations.
- **Do not** set the state constraint bound *below* a spec threshold. If the spec says `if counter >= 10 then crash` and you constrain `counter <= 5`, you've hidden the bug.
- **Do not** forget to mention the reduction in your results. "Invariant holds" with N=3 and a constraint is very different from "invariant holds" unconditionally. State the bounds.
- **Do not** use VIEW to hide variables that appear in the invariant. You'll get false "already seen" hits on states that actually violate it.

## Output format

```
## Diagnosis
Blowup source: <unbounded var | constant size | symmetry | data precision>
Evidence: <states/sec, queue growth, coverage numbers>

## Reductions applied
1. <technique> — <before → after>
...

## New .cfg
<config block>

## Soundness caveats
<what this reduction can miss — and why you believe it won't, for this property>

## Result
<states explored, time, invariant status>
```
