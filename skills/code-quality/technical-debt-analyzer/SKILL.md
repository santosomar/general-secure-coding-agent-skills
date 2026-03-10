---
name: technical-debt-analyzer
description: Analyzes a codebase to quantify and locate technical debt — where it lives, what it costs, and what order to pay it down in. Use when planning a refactoring sprint, when justifying engineering time to stakeholders, when the user asks where the codebase hurts most, or when onboarding to a legacy system.
license: Apache-2.0
metadata:
  category: "code-quality"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "code-smell-detector, code-refactoring-assistant, dead-code-eliminator"
---

# Technical Debt Analyzer

Technical debt is a **cost metaphor**: structure you'd build differently today, paid for in slower future changes. The analysis question is not "where is the bad code?" — it's "where does the bad code *cost* the most?"

## The cost equation

```
debt_cost(file) = interest_rate(file) × principal(file)
```

- **Principal** = how much work to fix the structure. Size of the refactor.
- **Interest rate** = how often you pay. Change frequency.

A terrible file nobody touches has high principal, zero interest → zero cost. Pay down debt where the *product* is high.

## Signals — gathering the data

| Signal                    | How to measure                                                     | What it tells you                    |
| ------------------------- | ------------------------------------------------------------------ | ------------------------------------ |
| Change frequency          | `git log --since=6.months --format= --name-only \| sort \| uniq -c` | Interest rate                        |
| Churn (lines changed)     | `git log --since=6.months --numstat`                               | Interest rate (finer-grained)        |
| Complexity                | Cyclomatic complexity, nesting depth, LOC per function             | Principal                            |
| Smell density             | → `code-smell-detector` findings per KLOC                          | Principal                            |
| Coupling                  | Fan-in (how many files import this)                                | Blast radius — changes here break many things |
| Bug concentration         | `git log --grep=fix --format= --name-only \| sort \| uniq -c`      | Where bugs actually happen — empirical interest |
| Test coverage             | Coverage report                                                    | Cost of *safe* refactor — low coverage = high risk |
| Author count              | `git log --format=%an -- <file> \| sort -u \| wc -l`               | Knowledge fragmentation — 8 authors, nobody owns it |
| `TODO`/`FIXME`/`HACK`     | `rg 'TODO\|FIXME\|HACK\|XXX'`                                      | Self-reported debt — the team already knows |

## The hotspot map

Plot files on two axes: **complexity** (y) vs **change frequency** (x).

```
              │  (cold+complex)         (HOT+complex) ← pay down here
  complexity  │  low priority            HIGH PRIORITY
              │
              │  (cold+simple)          (hot+simple)
              │  ignore                  healthy — leave alone
              └──────────────────────────────────
                        change frequency
```

The upper-right quadrant is where every sprint loses time. That's the paydown target.

## Worked example

**Input:** Mid-size service, ~200 files. After gathering signals:

| File                    | Commits (6mo) | Cyclomatic | Bugs (6mo) | Coverage | Authors |
| ----------------------- | ------------- | ---------- | ---------- | -------- | ------- |
| `billing/invoice.py`    | 34            | 58         | 7          | 31%      | 6       |
| `api/handlers.py`       | 41            | 22         | 2          | 78%      | 4       |
| `legacy/xml_export.py`  | 2             | 91         | 0          | 0%       | 1       |
| `utils/date.py`         | 19            | 8          | 1          | 95%      | 3       |

**Analysis:**

- `billing/invoice.py`: **Top priority.** High churn + high complexity + high bugs + low coverage = you're paying maximum interest on maximum principal, and you can't safely refactor it. Step 1 is coverage.
- `api/handlers.py`: High churn but *moderate* complexity and good coverage. This is healthy hot code — it changes because the product changes. Leave it.
- `legacy/xml_export.py`: Off-the-charts complexity, zero coverage — but 2 commits in 6 months and zero bugs. It's frozen. Principal is huge but interest is zero. Don't touch.
- `utils/date.py`: Boring. Fine.

**Output:**

> ## Debt hotspots
>
> 1. **`billing/invoice.py`** — 34 commits, cyclomatic 58, 7 bugs, 31% coverage.
>    Every billing change takes 2× longer than it should. Seven bugs in six months is the highest concentration in the codebase.
>    **Paydown plan:** (a) add characterization tests to reach 70% coverage — ~3 days; (b) extract `InvoiceCalculator` from the 200-line `compute()` — ~2 days; (c) split `Invoice` god class by state (draft vs. issued) — ~1 week. Step (a) is a prerequisite; do not skip.
>
> ## Explicitly not paying down
>
> - `legacy/xml_export.py` — complexity 91, but 2 commits in 6 months and zero bugs. Stable. Touching it is risk without reward.

## Debt you can't measure from the code

Some debt is in people's heads, not in the AST:

- **Knowledge debt:** One person understands the payment flow; they're leaving. The code might be clean, but the cost of change just went up 10×.
- **Decision debt:** The team chose Mongo in 2018; the access patterns are now relational. The code is fine; the choice isn't.
- **Dependency debt:** Pinned to a library version that's EOL. The code is fine until the CVE drops.

Ask about these. They don't show up in `git log`.

## Do not

- **Do not** rank by complexity alone. Complex frozen code is free. Simple hot code is also free. Only the product matters.
- **Do not** report a debt score without a paydown plan. "This file scores 847 debt points" is useless. "Refactoring this file will take ~1 week and will stop costing you ~2 hours per billing change" is actionable.
- **Do not** recommend rewriting. Rewrites are how debt doubles. Recommend incremental paydown with tests as the safety net.
- **Do not** ignore low-coverage hotspots. They're the most expensive *and* the hardest to fix safely — which is exactly why they stay expensive. Break the cycle: coverage first.

## Output format

```
## Hotspots (pay down)
<N>. <file>  — <churn> commits, complexity <N>, <N> bugs, <N>% coverage
    Interest: <what this costs per sprint — be concrete>
    Principal: <what the refactor is, roughly how long>
    Prerequisite: <coverage? knowledge? — anything blocking safe refactor>

## Explicitly deferred (high principal, low interest)
- <file> — <why it's safe to ignore>

## Non-code debt (from asking, not measuring)
- <knowledge / decision / dependency debt>
```
