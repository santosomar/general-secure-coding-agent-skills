---
name: model-guided-code-repair
description: Uses a model checker's counterexample trace to localize the fault in the model, propose a fix, and propagate that fix back to the source code. Use when a model checker (TLC, NuSMV, Spin) finds a violation and you need to turn the trace into a code change, not just understand it.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "tlaplus-guided-code-repair, counterexample-debugger, counterexample-to-test-generator"
---

# Model-Guided Code Repair

A counterexample trace says "here's a sequence of steps that breaks your property." The repair question is: **which step shouldn't have been allowed?** Strengthen that step's guard, and the trace is blocked.

For TLA+-specific repair, → `tlaplus-guided-code-repair`. This skill is the general workflow across checkers.

## The repair loop

```
┌─────────────────┐     violation      ┌──────────────┐
│ Model checker   │────────────────────▶│ Trace        │
│ (TLC/NuSMV/Spin)│                     │ analysis     │
└─────────────────┘                     └──────┬───────┘
        ▲                                      │
        │                                ┌─────▼────────┐
        │                                │ Pick culprit │
        │                                │ step         │
        │                                └─────┬────────┘
        │        check new model               │
        └────────────────────────────────┌─────▼────────┐
                                         │ Strengthen   │
                                         │ guard        │
                                         └─────┬────────┘
                                               │
                                         ┌─────▼────────┐
                                         │ Propagate to │
                                         │ source code  │
                                         └──────────────┘
```

## Step 1 — Find the culprit step

The trace is a sequence of states `s₀ → s₁ → ... → sₙ` where `sₙ` violates the property. Walk **backwards** from `sₙ`:

1. At `sₙ`, what's wrong? (Which clause of the invariant is false?)
2. At `sₙ₋₁ → sₙ`, what action fired? Did that action *make* the bad thing true?
3. If yes, `sₙ₋₁ → sₙ` is a candidate culprit.
4. If no (the bad thing was already latent at `sₙ₋₁`), keep walking back.

The culprit is the **latest step that could have prevented the violation** — the one where adding a guard would have blocked the trace.

## Step 2 — Propose a guard

Three kinds of fix, from least to most invasive:

| Fix kind             | What it does                                        | When                                            |
| -------------------- | --------------------------------------------------- | ----------------------------------------------- |
| Strengthen guard     | Action can't fire in the bad state                  | The action was legitimate, just fired too early |
| Add missing action   | New action to restore the invariant                 | Something should have happened but didn't       |
| Change the property  | The property was wrong, not the system              | The "violation" is actually intended behavior   |

**For a strengthened guard:** The new conjunct should be the negation of whatever was true at the culprit step that shouldn't have been. Look at `sₙ₋₁` — what predicate would have excluded it?

## Step 3 — Check the fix (in the model)

Re-run the checker. Three outcomes:

- **Property now holds** → good, but check you haven't broken liveness (made the guard so strong nothing happens).
- **Different counterexample** → you blocked one trace, another exists. Iterate.
- **Same counterexample** → your guard didn't actually block it. The culprit step was wrong.

Also check: did the fix introduce deadlock? Run with deadlock checking on.

## Step 4 — Propagate to code

The model action maps to some code region (you built that mapping during → `program-to-tlaplus-spec-generator` or equivalent). The new guard conjunct translates:

| Model guard                    | Code change                                            |
| ------------------------------ | ------------------------------------------------------ |
| `/\ lock_holder = self`        | Add `assert(lock_held_by_me)` or wrap in lock          |
| `/\ Len(queue) < Max`          | Add `if (queue.size() >= Max) return/wait`             |
| `/\ state[p] = "ready"`        | Add state check: `if (p.state != READY) return`        |
| `/\ msg.epoch = current_epoch` | Add epoch comparison; drop stale messages              |

The code change should be **at least as strong** as the model guard. If the model says `x > 0` and you add `x > 5` in code — fine, stronger. If you add `x >= 0` — weaker, the bug may still exist.

## Worked example

**Model (NuSMV):** Reader-writer lock. Property: `AG !(reading & writing)` — never read and write simultaneously.

**Trace:**
```
State 1: readers=0, writers=0, reading=FALSE, writing=FALSE
State 2: readers=1, writers=0, reading=TRUE,  writing=FALSE   [StartRead]
State 3: readers=1, writers=1, reading=TRUE,  writing=TRUE    [StartWrite]  ← VIOLATION
```

**Walk back:** State 3 violates. Action `StartWrite` at 2→3 set `writing=TRUE` while `reading=TRUE`. That's the culprit.

**Current `StartWrite` guard:** `writers = 0`. It only checks no other *writers*. It doesn't check readers.

**Fix:** `StartWrite` guard becomes `writers = 0 & readers = 0`.

**Re-check:** Property holds. Deadlock check: fine (readers eventually finish → guard becomes true).

**Code:** Find the `acquire_write_lock` function. It currently checks `if (writer_count == 0)`. Add `&& reader_count == 0`.

## Do not

- **Do not** strengthen the *property* to make it pass. The property encodes the requirement. Weakening it until it holds is lying.
- **Do not** fix only the model without propagating to code. The model is a means to an end — the code is what runs.
- **Do not** add a guard that blocks *every* trace. `FALSE` makes any safety property hold and every liveness property fail. Check the action still fires on good paths.
- **Do not** trust a fix without re-running the *full* check. Blocking one counterexample doesn't mean there isn't a second path to the same violation.

## Output format

```
## Counterexample (key steps)
<state i> → [Action] → <state i+1>     <-- culprit

## Violation
At state <n>: <which clause of the property is false>

## Culprit
Action: <name>
Fired in state: <key variables>
Why it shouldn't have: <predicate that should have blocked it>

## Model fix
<before/after of the action guard>

## Re-check result
<property status, deadlock status, new counterexamples if any>

## Code fix
File: <path>
Change: <before/after>
Mapping: model action <X> ↔ code <function/region>
```
