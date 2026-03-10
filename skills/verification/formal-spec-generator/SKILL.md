---
name: formal-spec-generator
description: Dispatch skill — routes a formal specification request to the right concrete generator based on what's being specified and what needs to be proven. Use when the user asks to formally specify something without naming a target formalism, or when unsure which verification tool fits the problem.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "python-to-dafny-translator, program-to-tlaplus-spec-generator, specification-to-temporal-logic-generator"
---

# Formal Spec Generator

**This is a dispatch skill.** It doesn't generate specs itself — it picks the right specialized generator based on what you're trying to verify.

## Dispatch table

| You have                                    | You want to prove                      | → Go to                                          |
| ------------------------------------------- | -------------------------------------- | ------------------------------------------------ |
| A Python function                           | It computes the right answer           | → `python-to-dafny-translator`                   |
| A C/C++ function                            | No overflow, no OOB, correct result    | → `cpp-to-dafny-translator`                      |
| Python/C++, Dafny can't prove it automatically | Deep mathematical property          | → `python-to-lean4-translator` / → `c-cpp-to-lean4-translator` |
| Concurrent/distributed code                 | No race, no deadlock, protocol correctness | → `program-to-tlaplus-spec-generator`        |
| Hardware-adjacent / embedded state machine  | Reachability, CTL branching properties | → `smv-model-extractor`                          |
| A natural-language requirement              | A checkable property (any formalism)   | → `requirement-to-tlaplus-property-generator` or → `specification-to-temporal-logic-generator` |
| A loop that Dafny can't verify              | The loop invariant                     | → `invariant-inference` / → `abstract-invariant-generator` |

## Three questions that route you

1. **Is it sequential or concurrent?**
   - Sequential → Dafny / Lean (prove a function correct)
   - Concurrent → TLA+ / SMV (prove an interleaving safe)

2. **Is the state space finite?**
   - Finite, small → SMV (explicit-state, fast, CTL)
   - Finite, large → TLA+ with TLC (explicit-state, more expressive)
   - Infinite → Dafny / Lean (symbolic, per-function)

3. **Do you need automation or a proof artifact?**
   - "Just tell me if it's right" → Dafny (SMT-backed, mostly automatic)
   - "I need the proof itself" → Lean (interactive, proof term is the output)

## Common misroutings

| You asked for                         | But you actually need                                       |
| ------------------------------------- | ----------------------------------------------------------- |
| "Verify this sorting function" → TLA+ | Dafny. TLA+ is for concurrency, not algorithmic correctness. |
| "Prove no deadlock" → Dafny           | TLA+. Dafny is sequential; deadlock is a concurrency property. |
| "Check this protocol" → Lean          | TLA+ first. Lean proofs of protocols are PhD theses. Model-check first. |
| "Formally specify this API" → any     | Probably not formal methods at all — → `api-design-assistant`. "Formal" often just means "written down carefully." |

## Do not

- **Do not** reach for the heaviest tool first. Model checking (TLA+/SMV) is orders of magnitude less effort than theorem proving (Lean). Try the cheap thing.
- **Do not** formally verify what you can adequately test. Formal methods pay off when the input space is adversarial or infinite. For `parse_date()`, write more tests.
