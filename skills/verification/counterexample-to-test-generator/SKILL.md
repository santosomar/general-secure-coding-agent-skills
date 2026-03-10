---
name: counterexample-to-test-generator
description: Converts a model checker counterexample trace into an executable test case in the source language, so the bug found in the model is reproducible (and regression-guarded) in the real code. Use when TLC/NuSMV/Spin finds a violation and you want a failing test before writing the fix.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "counterexample-debugger, bug-reproduction-test-generator, model-guided-code-repair"
---

# Counterexample → Test Generator

A model checker trace is a **proof that the bug exists** — in the model. A test is a **proof it exists in the code.** The translation is: map each model action back to a code operation, sequence them, and assert the invariant.

This is the model-checking analogue of → `bug-reproduction-test-generator`.

## The mapping problem

| Model trace element            | Test element                                                |
| ------------------------------ | ----------------------------------------------------------- |
| Initial state                  | Test setup / fixture                                        |
| `Action(p)` firing             | Call the code function that `Action` models, as process `p` |
| Interleaving of actions        | Thread scheduling — controlled or forced                    |
| Final (violating) state        | Assertion on the corresponding code state                   |

The hard part is **interleaving.** The model checker found a specific schedule that breaks things. Your test has to reproduce that schedule.

## Controlling the schedule

| Determinism level         | Technique                                                    | Reliability            |
| ------------------------- | ------------------------------------------------------------ | ---------------------- |
| Full control              | Single-threaded simulation: call each step explicitly in order | 100% — preferred     |
| Partial control           | Barriers/latches between steps to force ordering             | High if used carefully |
| No control                | Run threads concurrently many times, hope for the interleaving | Flaky — last resort  |

**Always try single-threaded first.** If the model actions are `Read(p1), Read(p2), Write(p1), Write(p2)`, you don't need real threads — just call `p1.read(); p2.read(); p1.write(); p2.write();` in sequence. The interleaving that matters is the *order of operations on shared state*, not actual OS threads.

## Step-by-step

1. **Extract the action sequence** from the trace. Ignore state dumps; you want the action names and parameters between states.
2. **Map each action to a code call.** Use the mapping from → `program-to-tlaplus-spec-generator` (or equivalent extraction).
3. **Identify shared state.** What model variables correspond to real shared objects? Those need to be set up in the fixture.
4. **Sequence the calls.** One after another, in trace order. If an action was `\E p : Foo(p)`, use the `p` TLC chose.
5. **Translate the invariant.** `Inv == counter = N` becomes `assert counter == N`.
6. **Run it.** If the test passes, your model doesn't match your code (see below). If it fails — good, you have a repro.

## Worked example

**TLC trace** (from the lost-update counter):

```
State 1: counter=0, tmp=[p1|->0, p2|->0], pc=[p1|->"start", p2|->"start"]
State 2: <Read(p1)>    counter=0, tmp=[p1|->0, p2|->0], pc=[p1|->"write", p2|->"start"]
State 3: <Read(p2)>    counter=0, tmp=[p1|->0, p2|->0], pc=[p1|->"write", p2|->"write"]
State 4: <Write(p1)>   counter=1, ...
State 5: <Write(p2)>   counter=1    ← Inv violated: 2 increments, counter should be 2
```

**Action sequence:** `Read(p1), Read(p2), Write(p1), Write(p2)`.

**Code mapping:**
- `Read(p)` ↔ first half of `increment()`: `tmp = counter`
- `Write(p)` ↔ second half of `increment()`: `lock(); counter = tmp + 1; unlock()`

Problem: `increment()` is one function in code, two actions in the model. To reproduce the interleaving, split it:

**Test (Go):**

```go
func TestLostUpdate_FromTLC(t *testing.T) {
    counter := 0
    var mu sync.Mutex

    // Simulate the TLC trace: Read(p1), Read(p2), Write(p1), Write(p2)
    // Split increment() into its two atomic halves.
    tmp1 := counter           // Read(p1) -- reads 0
    tmp2 := counter           // Read(p2) -- reads 0
    mu.Lock(); counter = tmp1 + 1; mu.Unlock()   // Write(p1) -- counter=1
    mu.Lock(); counter = tmp2 + 1; mu.Unlock()   // Write(p2) -- counter=1

    // Invariant: two increments → counter == 2
    if counter != 2 {
        t.Fatalf("lost update: 2 increments, counter=%d (TLC trace reproduced)", counter)
    }
}
```

Test fails with `counter=1`. Bug reproduced, deterministically, no real threads.

## When the test doesn't reproduce

The trace violated the model invariant but the test passes. Three reasons:

| Cause                                  | How to tell                                             | Fix                                         |
| -------------------------------------- | ------------------------------------------------------- | ------------------------------------------- |
| Model is more abstract than code       | Code has a check the model doesn't                      | Fix the model — it was over-approximating   |
| Wrong code mapping                     | You called the wrong function for an action             | Re-check the extraction mapping             |
| Atomicity mismatch                     | Model treats X as one step; code has finer interleaving | Refine the model's actions                  |

A test that doesn't reproduce is **good news** — it means the model found a bug in itself, not the code. Fix the model.

## Do not

- **Do not** use real threads when a sequential simulation reproduces the interleaving. Real threads make the test flaky and slow.
- **Do not** assert the *entire* final state. Assert the invariant. Other state may legitimately differ between model and code.
- **Do not** skip the test because the model "already proved it." The model is an abstraction — the test confirms the abstraction was faithful.
- **Do not** leave the test in the suite after the fix without inverting the assertion. Post-fix, the test should *pass* (counter == 2) — it becomes a regression guard.

## Output format

```
## Action sequence (from trace)
1. <Action(params)>
2. ...

## Code mapping
<Action> ↔ <function/code region>

## Schedule strategy
<single-threaded simulation | barriers | stress loop> — <why>

## Test
<code block — executable>

## Expected
FAIL before fix: <what assertion trips>
PASS after fix:  <same assertion, now holds>
```
