---
name: design-pattern-suggestor
description: Recognizes structural situations that match known design patterns and recommends whether to apply them — or explains why the pattern doesn't fit. Use when the user has a structural problem and is considering a pattern, when reviewing a design that uses a pattern questionably, or when the user asks which pattern fits their situation.
license: Apache-2.0
metadata:
  category: "code-quality"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "code-smell-detector, code-refactoring-assistant, api-design-assistant"
---

# Design Pattern Suggestor

A pattern is a **named solution to a recurring structural problem.** The value is the vocabulary, not the code. The risk is applying a pattern because it's familiar, not because the problem it solves is present.

## Problem → pattern (NOT pattern → problem)

Start from the symptom. The pattern is the answer; the problem is the question.

| Symptom                                                           | Pattern           | Or maybe just…                            |
| ----------------------------------------------------------------- | ----------------- | ----------------------------------------- |
| Switch on type code, same switch in 3+ places                     | Strategy / State  | …one switch if it's only in one place     |
| Constructor has 8 parameters, 5 are optional                      | Builder           | …keyword arguments with defaults          |
| Need exactly one of a thing, globally                             | Singleton         | …a module-level variable (in Python/JS)   |
| Class hierarchy × orthogonal dimension = class explosion          | Decorator / Bridge| …composition with a field                 |
| Creating objects, but which concrete type depends on runtime data | Factory           | …a dict `{key: constructor}` — often enough |
| Many objects need to know when one object changes                 | Observer          | …a callback list — that IS observer       |
| Different interface than what you have, can't change either side  | Adapter           | Usually correct, rarely over-applied      |
| Need undo, or need to queue/log/serialize operations              | Command           | Usually correct                           |
| Traversing a composite structure uniformly                        | Visitor / Iterator| …if the structure rarely changes, just recurse |

## The "or maybe just…" column matters

Most pattern misuse is applying GoF ceremony where the language already gives you the mechanism for free:

- **Strategy in Python:** Functions are first-class. `strategies = {"a": fn_a, "b": fn_b}` is the pattern.
- **Singleton in Python/JS:** A module IS a singleton. `import config` — done.
- **Observer in anything with events:** `addEventListener`, signals, Rx — the pattern is built into the runtime.
- **Iterator:** Every modern language has `yield` / generators. You don't write `hasNext()/next()`.

If the "or maybe just" fits, use it. The pattern is the fallback for languages that lack the mechanism.

## Anti-pattern detection

Sometimes the finding is "you're using a pattern wrong":

| Anti-pattern                            | Symptom                                                         |
| --------------------------------------- | --------------------------------------------------------------- |
| Singleton-as-global                     | Singleton exists to avoid passing a dependency — hidden coupling, untestable |
| Factory-for-one                         | `UserFactory.create()` returns `new User()` — the factory adds nothing |
| Observer soup                           | 10 observers, firing order matters, some observers trigger other observers — you've built an implicit state machine |
| Premature Strategy                      | One concrete strategy. The interface exists "in case we need another." → speculative generality |
| Visitor on a stable hierarchy           | The hierarchy never changes; visitor's indirection buys nothing |

## Worked example

**Symptom:** "Every time we add a payment method we have to edit `process_payment()` and `validate_payment()` and `refund_payment()` — three switches on `method_type`."

**Match:** Same switch, 3 places, on a type code → **Strategy**.

**Check the "or maybe just":** What language? Java. No first-class functions pre-lambdas (or the team doesn't use them). Strategy is the idiomatic fit.

**Recommendation:**

> Three switches on `method_type`, growing with every new payment method → Strategy pattern.
>
> ```
> interface PaymentMethod {
>     void process(Payment p);
>     ValidationResult validate(Payment p);
>     void refund(Payment p);
> }
> class CreditCard implements PaymentMethod { ... }
> class PayPal implements PaymentMethod { ... }
> ```
>
> New payment method → one new class, zero edits to existing switches.
>
> **Counter-check:** Do `process`/`validate`/`refund` always change together when a method is added? If `validate` is actually the same for 4 of 5 methods, Strategy will force you to duplicate it. Consider: Strategy for `process`/`refund`, a single `validate` with a small type check.

## Do not

- **Do not** suggest a pattern without stating the *problem it solves here*. "Use Factory" is cargo cult. "Use Factory because which concrete class depends on a config value you don't know at compile time" is a reason.
- **Do not** apply Java-idiom patterns to languages that don't need them. Strategy in a language with first-class functions is just "pass a function."
- **Do not** add a pattern for a problem you *might* have. One concrete class? No Strategy. One way to build it? No Builder. YAGNI.
- **Do not** suggest Visitor unless the type hierarchy is closed and the operations are open. If you're adding types more often than operations, Visitor fights you.

## Output format

```
## Symptom
<the structural problem — what hurts>

## Pattern fit
<pattern> — <why it matches this specific symptom>

## Or maybe just…
<simpler alternative if one exists — and when to prefer it>

## Sketch
<minimal code showing the pattern applied to THIS case, not a textbook example>

## When NOT to
<conditions under which this recommendation would be wrong>
```
