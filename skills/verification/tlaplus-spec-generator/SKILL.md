---
name: tlaplus-spec-generator
description: Translates natural-language or pseudocode descriptions of concurrent and distributed systems into TLA+ specifications ready for the TLC model checker. Identifies state variables, actions, type invariants, safety properties, and liveness properties from the description. Use when formalizing a protocol, when the user describes a distributed algorithm to verify, when designing a consensus or locking scheme, or when starting formal verification of a concurrent system.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "requirement-to-tlaplus-property-generator"
---

# TLA+ Spec Generator

Turn a prose description of a concurrent or distributed system into a TLA+ module that TLC can actually check. The hard part is not TLA+ syntax — it is deciding what belongs in the state and what doesn't.

## Step 1 — Decide if TLA+ is the right tool

TLA+ is for: nondeterminism, concurrency, message-passing, protocols, state machines — anything where *interleaving* matters.

TLA+ is **not** for: sequential algorithms, data-structure invariants within one thread, arithmetic correctness, type-level properties. For those → `cpp-to-dafny-translator` or → `c-cpp-to-lean4-translator`.

Rule of thumb: if the bug you're afraid of requires two things happening "at the same time," use TLA+. If it's "this function returns the wrong value," don't.

## Step 2 — Extract state variables

Read the description and mark every **noun that persists across steps and can be observed**. These become `VARIABLES`.

From *"Clients send requests to a server, which processes them one at a time from a queue"* you pull:
- `queue` — sequence of pending requests
- `serverState` — what the server is currently doing (`"idle"` or `"busy"`)
- *not* "clients" — they are a constant set, not a variable
- *not* "request" — it's an element *in* the queue, not its own variable

Every variable must appear in `Init` (initial value) and in the `UNCHANGED` tuple of every action that doesn't touch it. Omitting a variable from `UNCHANGED` lets it change arbitrarily — the #1 beginner bug.

**Keep the state minimal.** Every variable multiplies the state space. If you can derive X from Y and Z, don't make X a variable — make it an operator: `X == Y + Z`.

## Step 3 — Extract actions

Read the description and mark every **verb that changes state**. Each becomes an action — a predicate relating unprimed (before) and primed (after) variables.

For each action, nail down three things:

| Part           | Question                              | Example for a `Dequeue` action              |
| -------------- | ------------------------------------- | ------------------------------------------- |
| Guard          | When is this action allowed to fire?  | `queue # <<>>` (queue non-empty)            |
| Effect         | Which variables change, and to what?  | `queue' = Tail(queue)`, `current' = Head(queue)` |
| Frame          | Which variables do **not** change?    | `UNCHANGED <<serverState>>`                 |

If the description says "the server picks any request from the queue," that's nondeterminism — model it with `\E i \in 1..Len(queue): ...`, not by always picking index 1. Under-modeling nondeterminism hides the bugs you're looking for.

## Step 4 — Extract properties

Two kinds. Get them from different words in the description:

**Safety** (nothing bad happens) — look for "never," "always," "at most," "cannot":
- "Two clients never hold the lock at once" → `Mutex == \A c1, c2 \in Clients: c1 # c2 => ~(holds[c1] /\ holds[c2])`
- "The balance is always non-negative" → `Solvent == balance >= 0`

**Liveness** (something good eventually happens) — look for "eventually," "will," "guaranteed to," "makes progress":
- "Every request is eventually served" → `Progress == \A r \in Requests: r \in submitted ~> r \in served`

Safety is the default target. **Only add liveness if the description explicitly promises it** — liveness needs fairness (`WF_vars` / `SF_vars`) on specific actions, and picking the wrong fairness gives you false positives (TLC says the property holds when it doesn't in reality).

Always include `TypeOK`. Even if the user didn't ask. Without it, TLC explores states where your queue is the number 7.

## Step 5 — Assemble the module

```tla
---------------------------- MODULE <Name> ----------------------------
EXTENDS Naturals, Sequences, FiniteSets, TLC

CONSTANTS <Parameters>       \* e.g., Clients, MaxQueueLen

VARIABLES <state vars>
vars == <<var1, var2, ...>>

TypeOK ==
    /\ var1 \in <Domain1>
    /\ var2 \in <Domain2>

Init ==
    /\ var1 = <initial1>
    /\ var2 = <initial2>

Action1 ==
    /\ <guard>
    /\ var1' = <new value>
    /\ UNCHANGED <<var2, ...>>

Action2 == ...

Next == Action1 \/ Action2 \/ ...

Spec == Init /\ [][Next]_vars /\ WF_vars(Next)   \* fairness only if liveness needed

<SafetyProp> == ...
<LivenessProp> == ...

THEOREM Spec => []TypeOK /\ []<SafetyProp>
=======================================================================
```

And a matching `.cfg`:

```
SPECIFICATION Spec
CONSTANTS
    Clients = {c1, c2, c3}
    MaxQueueLen = 3
INVARIANTS
    TypeOK
    <SafetyProp>
PROPERTIES
    <LivenessProp>
```

**Keep constants tiny.** 2–3 elements per set, small bounds on naturals. The interesting bugs appear with 2 or 3 processes. If your bug needs 50 processes to manifest, TLA+ model checking won't find it anyway — write a TLAPS proof by hand.

## Worked example

**Input:** *"A lock server grants a lock to one client at a time. Clients request the lock; the server grants it to a waiting client if the lock is free. A client holding the lock eventually releases it. We need mutual exclusion."*

**Extraction:**
- Constants: `Clients` (the set of clients)
- Variables: `holder` (which client holds it, or `None`), `waiting` (set of clients who've requested)
- Actions: `Request(c)`, `Grant(c)`, `Release(c)`
- Safety: at most one holder
- Liveness: "eventually releases" is an *assumption* (clients are fair), not a property to check. "Every waiter eventually gets the lock" — the description doesn't actually promise that. Skip liveness unless asked.

```tla
---------------------------- MODULE LockServer ----------------------------
EXTENDS Naturals, FiniteSets

CONSTANTS Clients
None == CHOOSE x : x \notin Clients

VARIABLES holder, waiting
vars == <<holder, waiting>>

TypeOK ==
    /\ holder \in Clients \cup {None}
    /\ waiting \subseteq Clients

Init ==
    /\ holder = None
    /\ waiting = {}

Request(c) ==
    /\ c # holder
    /\ c \notin waiting
    /\ waiting' = waiting \cup {c}
    /\ UNCHANGED holder

Grant(c) ==
    /\ holder = None
    /\ c \in waiting
    /\ holder' = c
    /\ waiting' = waiting \ {c}

Release(c) ==
    /\ holder = c
    /\ holder' = None
    /\ UNCHANGED waiting

Next == \E c \in Clients : Request(c) \/ Grant(c) \/ Release(c)

Spec == Init /\ [][Next]_vars

Mutex == \A c1, c2 \in Clients : (holder = c1 /\ holder = c2) => c1 = c2
\* Trivially true since holder is a single value — but that's the point:
\* the model *structure* enforces mutex. TypeOK catches violations.
===========================================================================
```

```
SPECIFICATION Spec
CONSTANTS Clients = {c1, c2, c3}
INVARIANTS TypeOK Mutex
```

Note `Grant` uses `\E c \in waiting` — the server picks *any* waiter, not a specific one. That's the nondeterminism in "grants it to a waiting client."

## Edge cases

- **Unbounded state** (counters, ever-growing logs, ever-incrementing timestamps): TLC will never finish. Either cap with a constant (`counter' = (counter + 1) % Max`), or abstract the unbounded part away, or accept that you're checking a bounded prefix.
- **Real time / timeouts:** Don't model wall-clock time. Model *the timeout firing* as a nondeterministic action that can happen whenever its guard holds. "After 30 seconds" becomes "eventually, nondeterministically."
- **Message loss / network partitions:** Model the network as a variable — a set of in-flight messages. `Send` adds to the set, `Receive` removes, `Drop` removes without delivery. The network is part of your state.
- **Crashes / restarts:** Add a `Crash(n)` action that resets node `n`'s local variables to `Init` values. Make sure your invariant still holds *through* a crash.
- **User wants to model-check code, not a protocol:** Wrong tool. TLA+ is for the *design* that the code implements. If the design is correct and the code is wrong, that's a bug for → `bug-localization`, not a model-checking failure.

## Do not

- **Don't over-model.** Thread IDs, specific message bytes, real timestamps — abstract them. Every detail you keep multiplies the state space.
- **Don't under-model nondeterminism.** If the system *can* pick any of N things, use `\E`. Modeling it as "always pick the first" hides every bug that requires picking the second.
- **Don't forget `UNCHANGED`.** An action that omits a variable from its `UNCHANGED` clause lets that variable become *anything*. TLC will then find nonsense counterexamples.
- **Don't add fairness casually.** `WF_vars(Action)` says "if Action is enabled forever, it eventually fires." That's a strong assumption — be sure the real system guarantees it before you rely on it to prove liveness.
- **Don't hand over a spec you haven't run through TLC once.** Parse errors and `TypeOK` violations on step 1 are embarrassing and preventable.
