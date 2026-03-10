---
name: program-to-tlaplus-spec-generator
description: Extracts a TLA+ specification from concurrent or distributed code, modeling the state machine, actions, and fairness conditions for model checking with TLC. Use when verifying concurrency properties of production code, when designing a protocol and wanting to check it before implementation, or when the user has a race condition and needs to prove the fix.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "requirement-to-tlaplus-property-generator, tlaplus-model-reduction, smv-model-extractor"
---

# Program → TLA+ Spec Generator

TLA+ models **state machines, not code.** The translation isn't line-by-line — it's "what are the states, and what are the atomic transitions between them?" This is an *abstraction* exercise.

## Step 1 — Identify the state

What persists between steps? That's your `VARIABLES`.

| Code construct                | TLA+ variable                                                |
| ----------------------------- | ------------------------------------------------------------ |
| Shared mutable state (locks, queues, counters) | Direct — `queue`, `lock_holder`             |
| Per-thread/process local state | A function: `pc \in [Proc -> {"idle", "waiting", "cs"}]`   |
| Network messages in flight    | A set or bag: `msgs \subseteq Message`                      |
| The program counter           | A `pc` variable per process — where in the code each one is |

**Do not model:** pure function arguments, loop counters that don't cross await points, anything that's deterministic from the rest.

## Step 2 — Identify atomic actions

An action is one indivisible step. In code, atomicity boundaries are:

- Lock acquire → release: everything inside is one action (if no awaits inside)
- Single CAS / atomic instruction: one action
- Between `await` points / blocking calls: one action
- A message send: one action; message receive: another action

Each action becomes a disjunct in `Next`:

```tla
Next == \/ Action1
        \/ Action2
        \/ \E p \in Proc : ProcessStep(p)
```

## Step 3 — Write each action

```tla
ActionName ==
  /\ <guard — when is this enabled?>
  /\ <effect — primed variables>
  /\ UNCHANGED <<everything else>>
```

The `UNCHANGED` is not optional. TLA+ makes you say what *doesn't* change.

## Worked example

**Code (Go-ish, two goroutines, shared counter, broken):**

```go
var counter int
var mu sync.Mutex

func increment() {         // called by 2 goroutines
    tmp := counter         // ← read outside lock!
    mu.Lock()
    counter = tmp + 1
    mu.Unlock()
}
```

**State:** `counter`, `mu` (who holds it or NULL), `pc[p]` per process, `tmp[p]` per process (the local).

**Atomicity:** The read `tmp := counter` is its own action (no lock). The lock-write-unlock is one action (atomic under the lock).

**TLA+:**

```tla
---- MODULE Counter ----
EXTENDS Integers
CONSTANTS Proc
VARIABLES counter, mu, pc, tmp

Init ==
  /\ counter = 0
  /\ mu = "free"
  /\ pc = [p \in Proc |-> "start"]
  /\ tmp = [p \in Proc |-> 0]

Read(p) ==
  /\ pc[p] = "start"
  /\ tmp' = [tmp EXCEPT ![p] = counter]
  /\ pc' = [pc EXCEPT ![p] = "locked_write"]
  /\ UNCHANGED <<counter, mu>>

LockedWrite(p) ==
  /\ pc[p] = "locked_write"
  /\ mu = "free"                          \* guard: can acquire
  /\ mu' = "free"                         \* lock+write+unlock atomic → mu unchanged net
  /\ counter' = tmp[p] + 1
  /\ pc' = [pc EXCEPT ![p] = "done"]
  /\ UNCHANGED tmp

Next == \E p \in Proc : Read(p) \/ LockedWrite(p)

Inv == (\A p \in Proc : pc[p] = "done") => counter = Cardinality(Proc)
====
```

**TLC finds:** `Inv` violated. Trace: P1 reads (tmp=0), P2 reads (tmp=0), P1 writes (counter=1), P2 writes (counter=1). Two increments, counter=1. The read-outside-lock is the bug; TLA+ found it because `Read` and `LockedWrite` are separate actions.

## What to abstract away

| Keep                                  | Abstract away                                             |
| ------------------------------------- | --------------------------------------------------------- |
| All shared state                      | Local variables that don't survive an atomic boundary     |
| Concurrency structure (who runs when) | Sequential business logic inside an atomic block          |
| Message contents that affect protocol | Message payloads that are just data                       |
| Failure modes (crash, partition)      | Performance, timing (unless you're proving real-time)     |

TLC explores every interleaving. Every variable you keep multiplies the state space. → `tlaplus-model-reduction` if TLC runs out of memory.

## Do not

- **Do not** translate line-by-line. A 10-line critical section under one lock is ONE action, not ten.
- **Do not** forget `UNCHANGED`. Omitting it means "anything can happen to this variable" — you'll get spurious counterexamples.
- **Do not** model infinite domains without bounding. `counter \in Nat` → TLC never finishes. Use `counter \in 0..N` in the `.cfg`.
- **Do not** add fairness unless you're checking liveness. For safety (invariants), fairness is irrelevant and slows TLC.
- **Do not** trust an `Inv` that passes without also checking you modeled the actions right. Sanity-check: does TLC find a state where `pc[p] = "done"` for all `p`? If not, your model never terminates and the invariant is vacuously true.

## Output format

```
## State
VARIABLES <list> — <each maps to what code construct>

## Actions
<Action> — atomic because <lock region | CAS | between awaits>
...

## Spec
<TLA+ module>

## .cfg
<constants, invariants, state constraints for TLC>

## Abstractions made
<what you dropped and why it's sound to drop>
```
