---
name: tlaplus-guided-code-repair
description: TLA+-specific instance of model-guided repair — reads a TLC error trace, identifies the enabling condition that should have been false, strengthens the corresponding action, and maps the fix to source code. Use when TLC reports an invariant violation or deadlock and you have the code-to-TLA+ mapping from extraction.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "model-guided-code-repair, program-to-tlaplus-spec-generator, counterexample-debugger"
---

# TLA+-Guided Code Repair

Specialization of → `model-guided-code-repair` for TLC. The general loop is the same; this skill covers the TLC-specific mechanics: reading error traces, understanding action enablement, and the `.cfg` levers.

## Reading a TLC error trace

TLC prints the trace as numbered states. Between states, it names which `Next` disjunct fired:

```
Error: Invariant Inv is violated.
State 1: <Initial predicate>
  pc = [p1 |-> "start", p2 |-> "start"]
  lock = "free"
  ...
State 2: <Read(p1) line 42, col 3 ...>
  pc = [p1 |-> "locked", p2 |-> "start"]
  ...
State 3: <Read(p2) line 42, col 3 ...>
  ...
State 4: <Write(p1) line 50, col 3 ...>         ← invariant fails here
```

Key TLC-specific info:
- **Action name + source location** between states — this is your `Next` disjunct.
- **Only changed variables** shown by default (toggle with `-difftrace`).
- **The parameter** if the action is `\E p : Foo(p)` — TLC tells you which `p`.

## Culprit identification — TLA+ specifics

In TLA+, every action is `guard /\ effect`. The guard is the conjunction of all unprimed conjuncts. The culprit's guard was **true when it shouldn't have been.**

At the culprit step, ask: *what unprimed predicate, if added to this action's guard, would have been FALSE in the pre-state?* That predicate is your fix candidate.

**Extracting the candidate from the pre-state:**
1. Look at the pre-state of the culprit action.
2. Look at the invariant. Which sub-clause of `Inv` is about to be violated?
3. The guard should have checked that sub-clause (or something that implies it).

## Common repair patterns by violation type

| TLC error                           | Likely culprit                                | Typical fix                                      |
| ----------------------------------- | --------------------------------------------- | ------------------------------------------------ |
| Invariant violated                  | Action that wrote the invariant-breaking value | Add guard preventing that write                  |
| Deadlock                            | *All* actions' guards are false — none can fire | *Weaken* a guard, or add a recovery action      |
| Temporal property violated (liveness) | Missing fairness, or an action that blocks progress | Add `WF_vars(Action)` or fix the blocker     |
| Action property `[][A]_v` violated  | Some `Next` step changed `v` in a way `A` forbids | That disjunct needs to respect `A`           |

**Deadlock is the opposite of invariant violation.** For invariants, guards are too weak. For deadlock, guards are too strong. Don't tighten guards after a deadlock.

## Worked example — TLC trace to code fix

**TLA+ spec:** Two-phase commit coordinator. Invariant: `Inv == committed => \A p \in Participants : vote[p] = "yes"`.

**TLC trace:**

```
State 1: <Init>
  vote = [p1 |-> "none", p2 |-> "none"]
  committed = FALSE

State 2: <Vote(p1) ...>
  vote = [p1 |-> "yes", p2 |-> "none"]

State 3: <Commit ...>                  ← Inv violated: committed=TRUE but vote[p2]="none"
  committed = TRUE
```

**Culprit:** `Commit` at step 2→3. Its pre-state: `vote = [p1 |-> "yes", p2 |-> "none"]`. `Commit` fired with p2 un-voted.

**Current action:**
```tla
Commit ==
  /\ committed = FALSE
  /\ committed' = TRUE
  /\ UNCHANGED vote
```

No check on votes at all. Missing guard: all participants voted yes.

**Fix:**
```tla
Commit ==
  /\ committed = FALSE
  /\ \A p \in Participants : vote[p] = "yes"      \* NEW
  /\ committed' = TRUE
  /\ UNCHANGED vote
```

**Re-run TLC:** Inv holds. Deadlock check: `Commit` is now disabled when p2 votes "no" — is there an `Abort` action? If not, add one (this is the "add missing action" repair).

**Code mapping:** `Commit` ↔ `Coordinator.finalize()`. The code:
```python
def finalize(self):
    if not self.committed:
        self.committed = True
        self.broadcast("COMMIT")
```

Missing the vote check. Fix:
```python
def finalize(self):
    if not self.committed and all(v == "yes" for v in self.votes.values()):
        self.committed = True
        self.broadcast("COMMIT")
```

## Fixing the model vs fixing the mapping

Sometimes the model's action was wrong, not the code. Two cases:

- **Model is too permissive:** The code *does* check votes, but you forgot the conjunct when extracting the TLA+. Fix: correct the TLA+ to match the code. Re-check — if Inv now holds, the code was fine; your extraction was buggy.
- **Model is too restrictive:** The code does something the model doesn't allow. Usually shows as deadlock in the model while the real system proceeds. Fix: add the missing action to the model.

Before touching code, verify: **does the TLA+ action actually match the code?** Go look.

## Do not

- **Do not** add the *invariant itself* as a guard. `Commit == /\ Inv /\ ...` makes Inv vacuously preserved by Commit. That's cheating — and usually causes deadlock.
- **Do not** fix only one action when several can violate the same invariant. Search `Next` for every disjunct that writes the variables in the violated clause.
- **Do not** forget `UNCHANGED` when adding new variables during repair. Every action must account for every variable.
- **Do not** assume the code has the same atomicity as the TLA+ action. If `Commit` is one TLA+ step but three code statements, the code fix might need a lock.

## Output format

```
## TLC trace (relevant steps)
State <i>: <vars>
  → <Action(params)>  [<file:line>]
State <i+1>: <vars>

## Culprit
Action: <name>
Pre-state: <the vars that mattered>
Missing guard: <predicate>

## TLA+ fix
```tla
<before>
--- becomes ---
<after>
```

## Re-check
Invariant: <pass/fail>
Deadlock: <pass/fail>
Other properties: <status>

## Code fix
<file:function> — the TLA+ action's code counterpart
Before/after: <diff>
Atomicity note: <does the code need additional synchronization to match the TLA+ atomicity?>
```
