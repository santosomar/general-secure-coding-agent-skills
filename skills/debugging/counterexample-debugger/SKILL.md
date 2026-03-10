---
name: counterexample-debugger
description: Interprets and explains counterexamples produced by model checkers or property-based testing tools to make them actionable. Use when TLC, NuSMV, CBMC, or a property-based test emits a counterexample the user doesn't understand, when a trace is too long to read, or when mapping a model-level trace back to source code.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "counterexample-to-test-generator, tlaplus-guided-code-repair, model-guided-code-repair"
---

# Counterexample Debugger

A counterexample is a **proof** that a property fails — a concrete sequence of states that leads to the violation. The trace is correct by construction; the job is to make it *understandable*.

## Triage by source

| Tool                   | Counterexample format                    | Key decoding move                                      |
| ---------------------- | ---------------------------------------- | ------------------------------------------------------ |
| TLC (TLA+)             | Numbered states with full variable dumps | Diff consecutive states — only the *changes* matter    |
| NuSMV / nuXmv          | State trace + `-> State: N.M <-` headers | Same: diff states, ignore unchanged vars               |
| CBMC / ESBMC           | C-level trace with assignments           | Map each assignment to a source line                   |
| Alloy                  | Instance graph (relational)              | Project to the atoms involved in the violated assertion|
| Hypothesis / QuickCheck| Shrunk failing input                     | The input IS the counterexample — no trace to decode   |

## The compression algorithm

Raw traces are unreadable because they show **every variable at every step**. Compress:

1. **Diff.** For each step `i → i+1`, show only variables that changed.
2. **Project.** Drop variables not mentioned in the violated property and not data-flowing into ones that are.
3. **Collapse stutter.** If five consecutive steps change nothing relevant (the model is busy-waiting on something you projected away), collapse to `(... 5 irrelevant steps ...)`.
4. **Name the steps.** If the model has actions/transitions, label each step with the action that fired. `Step 3 [ClientSend]` beats `Step 3`.

A 40-state TLC trace usually compresses to 5–8 relevant transitions.

## Narrative translation

After compression, narrate the trace as a story:

- **Setup:** What's the initial state? (Usually trivial — all zeros/empty)
- **Trigger:** Which step first moves toward the bad state? This is often NOT step 1.
- **Point of no return:** Which step makes the violation inevitable? Every step before this is reversible; every step after is doomed.
- **Violation:** Which step actually breaks the property, and what's the state at that moment?

## Worked example (TLC)

**Property:** `Inv == (lock_holder /= NULL) => (waiters = {})`
(If someone holds the lock, no one should be waiting — i.e., the lock is supposed to be fair.)

**Raw trace:** 12 states, 7 variables each. 84 lines.

**Compressed:**

```
Step 1 [Init]       lock_holder=NULL  waiters={}  pc=<<idle,idle>>
Step 4 [Acquire(1)] lock_holder=1                                    ← trigger
Step 7 [Wait(2)]                      waiters={2}                    ← point of no return
Step 8 [CheckInv]   VIOLATION: lock_holder=1 ∧ waiters={2}
```

**Narrative:** Process 1 acquires the lock. Three steps later — before process 1 releases — process 2 enters the wait set. The invariant assumed acquire-release would be atomic from the waiters' perspective. It isn't.

**Map to code:** Step 7's `Wait(2)` action corresponds to `Lock::wait()` at `lock.c:44`. The check-then-wait in that function isn't guarded.

## Lasso-shaped traces (liveness)

Liveness counterexamples (`<>[]P` failures) are **lassos**: a finite prefix followed by a cycle that repeats forever. TLC marks the loop-back point with `Back to state N`.

- The prefix tells you how the system got into the trap.
- The cycle tells you what it does forever instead of making progress.
- The fix is almost always in the cycle, not the prefix.

## Edge cases

- **Single-state counterexample:** The *initial state* violates the invariant. You don't have a bug in a transition — your initial state is wrong (or your invariant is).
- **Trace longer than the model's diameter:** TLC found the violation via BFS, so this shouldn't happen — unless you're running DFS/simulation mode. Switch to BFS; the minimal trace is the readable one.
- **Counterexample involves a fairness assumption you forgot to state:** The "bug" is that you didn't say `WF_vars(SomeAction)`. The trace will show an enabled action that never fires.
- **The counterexample is correct behavior:** Sometimes the property is wrong, not the system. If the narrative sounds reasonable, re-read the property.

## Do not

- **Do not** dump the raw trace on the user. Always compress first.
- **Do not** fix the first step that looks suspicious. Find the point of no return — that's where the fix goes.
- **Do not** assume model-level state maps 1:1 to code-level state. The model is an abstraction; confirm the mapping.
- **Do not** turn the trace into a test without understanding it first. → `counterexample-to-test-generator` consumes the *narrative*, not the raw trace.

## Output format

```
## Property violated
<property, restated in English>

## Compressed trace
Step <N> [<action>]  <changed vars only>   ← <annotation: trigger / PNR / violation>
...

## Narrative
<setup → trigger → point of no return → violation, in prose>

## Source mapping
Step <N> ↔ <file>:<line>  (<function>)

## Suggested fix location
<file>:<line> — <what about this code allows step N>
```
