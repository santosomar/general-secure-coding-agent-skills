---
name: smv-model-extractor
description: Extracts an SMV (NuSMV/nuXmv) finite-state model from code or state-machine descriptions, for CTL/LTL model checking of reactive systems. Use when verifying hardware-adjacent or embedded logic, when the state space is naturally finite and small, or when CTL branching-time properties are needed.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "program-to-tlaplus-spec-generator, specification-to-temporal-logic-generator"
---

# SMV Model Extractor

SMV is for **small, finite, synchronous** systems — think hardware controllers, protocol handshakes, embedded state machines. Compared to → `program-to-tlaplus-spec-generator`: SMV is synchronous (all transitions fire together) and finite by construction; TLA+ is asynchronous and handles unbounded state.

## When SMV over TLA+

| Characteristic              | SMV fits                                       | TLA+ fits                          |
| --------------------------- | ---------------------------------------------- | ---------------------------------- |
| State space                 | Naturally finite (< 10⁸ states ideally)        | Can be unbounded (abstracted)      |
| Execution model             | Synchronous — all modules step together        | Asynchronous interleaving          |
| Property language           | CTL (branching) + LTL                          | TLA (linear, actions)              |
| Source domain               | Hardware, embedded, protocol handshakes        | Distributed systems, concurrency   |

## The SMV shape

```
MODULE main
VAR
  state : {idle, waiting, active, error};
  counter : 0..15;
  input : boolean;

ASSIGN
  init(state) := idle;
  next(state) := case
    state = idle & input      : waiting;
    state = waiting & counter = 0 : active;
    state = active & !input   : idle;
    TRUE                      : state;      -- stutter
  esac;

  init(counter) := 0;
  next(counter) := case
    state = waiting & counter > 0 : counter - 1;
    TRUE : counter;
  esac;

SPEC AG (state = error -> AF state = idle)   -- CTL: always recoverable
```

## Extraction from code

| Code pattern                      | SMV construct                                               |
| --------------------------------- | ----------------------------------------------------------- |
| `enum State { A, B, C }`          | `state : {A, B, C}`                                         |
| `switch(state) { case A: ... }`   | `next(state) := case state = A : ...; esac`                 |
| Bounded int (`uint4_t`, `i % 16`) | `x : 0..15`                                                 |
| Bool flag                         | `flag : boolean`                                            |
| `if (cond) state = X`             | One branch in the `case`                                    |
| Parallel components               | Separate `MODULE`s, instantiated in `main`                  |
| Shared variable between components| Passed as parameter: `MODULE foo(shared_var)`               |

## Worked example

**Code (traffic light controller):**

```c
enum { RED, GREEN, YELLOW } light = RED;
int timer = 0;

void tick(bool car_waiting) {
    switch (light) {
    case RED:
        if (timer >= 30 && car_waiting) { light = GREEN; timer = 0; }
        else timer++;
        break;
    case GREEN:
        if (timer >= 25) { light = YELLOW; timer = 0; }
        else timer++;
        break;
    case YELLOW:
        if (timer >= 5) { light = RED; timer = 0; }
        else timer++;
        break;
    }
}
```

**SMV:**

```
MODULE main
VAR
  light : {RED, GREEN, YELLOW};
  timer : 0..31;
  car_waiting : boolean;         -- environment input, unconstrained

ASSIGN
  init(light) := RED;
  init(timer) := 0;

  next(light) := case
    light = RED    & timer >= 30 & car_waiting : GREEN;
    light = GREEN  & timer >= 25               : YELLOW;
    light = YELLOW & timer >= 5                : RED;
    TRUE : light;
  esac;

  next(timer) := case
    next(light) != light : 0;            -- reset on transition
    timer < 31           : timer + 1;
    TRUE                 : timer;        -- saturate
  esac;

-- Safety: never GREEN and YELLOW simultaneously (trivially true here — single var)
-- Liveness: if a car waits, eventually GREEN
LTLSPEC G (car_waiting -> F light = GREEN)
```

`timer` capped at 31 — SMV needs finite ranges. The saturation at 31 is a modeling choice; check it doesn't affect the property (here it doesn't — thresholds are all ≤ 30).

**Check:** `LTLSPEC` fails if `car_waiting` can oscillate — a car that waits then leaves then waits again resets no clock. The fix is in the property (`G car_waiting -> F GREEN` — *persistent* waiting) or a fairness assumption.

## Bounding the infinite

SMV requires finite domains. When the code has unbounded state:

| Unbounded construct        | Bounding strategy                                                |
| -------------------------- | ---------------------------------------------------------------- |
| Counter that grows forever | Cap at `max(threshold) + 1` — values above the highest comparison are equivalent |
| Message queue              | Bound length. If property violated → was it at the bound? Raise the bound. |
| Arbitrary data payload     | Abstract to a few symbolic values. `data : {d1, d2}` — if it doesn't affect control, one value suffices. |

If you can't sensibly bound it, SMV is the wrong tool — use TLA+ with a state constraint, or go parametric.

## Do not

- **Do not** model a naturally asynchronous system as synchronous. If processes step independently, SMV's lockstep will hide interleavings. Use TLA+.
- **Do not** forget the `TRUE : state` stutter branch. Without it, `case` is partial → "incomplete case" or arbitrary next value.
- **Do not** leave environment inputs unconstrained without thinking. `car_waiting : boolean` with no constraint means it can toggle every tick — that's maximum chaos, often too adversarial.
- **Do not** use a range larger than you need. `0..1000000` → NuSMV allocates log₂(10⁶) ≈ 20 bits per state. Blowup is exponential in bits.

## Output format

```
## Finite-state abstraction
<var> : <range> — <what code construct, how bounded>

## SMV model
<code block>

## Properties
<CTL/LTL specs — each traced to a requirement>

## Bounding assumptions
<every cap you imposed, and why it's sound for this property>
```
