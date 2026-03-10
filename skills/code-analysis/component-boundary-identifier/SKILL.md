---
name: component-boundary-identifier
description: Identifies natural component boundaries inside a monolith by clustering the dependency graph, finding the cuts with minimum coupling. Use when planning to modularize or extract microservices, when deciding what can be deployed independently, or when the user asks where the seams in this codebase are.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "dependency-resolver, code-refactoring-assistant"
---

# Component Boundary Identifier

A good component boundary has **high cohesion inside, low coupling across.** Finding boundaries is a graph clustering problem: build the dependency graph, find cuts that minimize cross-cut edges.

## Build the dependency graph

Nodes are units (functions, classes, or files — pick a granularity). Edges are dependencies:

| Edge type                | Weight hint          | Why                                          |
| ------------------------ | -------------------- | -------------------------------------------- |
| Direct call (A calls B)  | High                 | Runtime coupling — A breaks if B's API changes |
| Import / include         | Medium               | Compile-time coupling                        |
| Shared data type         | Medium               | Schema coupling — both break if the type changes |
| Shared database table    | High                 | Data coupling — hardest to split              |
| Shared config key        | Low                  | Easy to duplicate                            |
| Co-change (git log)      | Medium               | Empirical — these *have* changed together    |

Weight edges by coupling strength. A function call once at startup is weaker than one in a hot loop.

## Clustering — find the cuts

| Method                   | How                                                         | Good for                        |
| ------------------------ | ----------------------------------------------------------- | ------------------------------- |
| Louvain / modularity     | Maximize (edges inside clusters) − (expected random edges)  | General-purpose, no target K    |
| Spectral clustering      | Eigenvectors of the graph Laplacian; cut at the gap         | When K is roughly known         |
| Min-cut between seeds    | Pick two modules you *know* should separate; find the cheapest cut | Extracting one thing specifically |
| Directory-as-prior       | Start from existing folder structure; measure if it's actually a good clustering | Validating current layout |

**Start with directory-as-prior.** The existing layout might already be right. Measure modularity of the current folder structure — if it's high, the work is done. If it's low, the folders are lying.

## Cohesion / coupling metrics

For a proposed boundary around cluster C:

- **Cohesion** = (edges within C) / (possible edges within C). Higher is better.
- **Coupling** = (edges crossing C's boundary) / (total edges touching C). Lower is better.
- **Instability** = (outgoing cross-edges) / (all cross-edges). High instability = depends heavily on others, should be extracted last.

Good cut: cohesion > 0.5, coupling < 0.2. (Rules of thumb — domain varies.)

## Worked example — Django monolith

**Graph:** 340 files, 1200 import edges, 89 shared-model edges.

**Directory-as-prior:**

| Directory    | Cohesion | Coupling | Verdict                                  |
| ------------ | -------- | -------- | ---------------------------------------- |
| `accounts/`  | 0.71     | 0.08     | Clean boundary. Extract as-is.           |
| `orders/`    | 0.64     | 0.31     | Leaky — what's crossing?                 |
| `reports/`   | 0.22     | 0.45     | **Not a real component.** Directory is a lie. |
| `utils/`     | 0.05     | 0.68     | Expected — utils is a grab bag, not a component |

**Drill into `orders/` coupling:**

| Cross-edge                            | Count | Type         |
| ------------------------------------- | ----- | ------------ |
| `orders/views.py → accounts/models.User` | 14  | Shared model |
| `orders/tasks.py → inventory/stock.py`  | 8   | Direct call  |
| `orders/models.py → payments/models.Payment` | 5 | FK relation |

The `User` dependency is fine — every service needs auth. The `inventory` coupling is the problem: `orders` shouldn't be calling inventory synchronously.

**Proposed cut:** `orders` is a component. Its interface is: receives `User` (from auth), emits `OrderPlaced` event (consumed by inventory). The 8 direct `stock.py` calls become event publications.

## The shared-database problem

The hardest coupling to break is shared tables. If `orders` and `inventory` both write to `stock_levels`, you can't cleanly separate them — whoever owns the table owns the other's data.

Options:
- **One owner, one API.** `inventory` owns the table. `orders` calls `inventory`'s API, never touches the table.
- **Event sourcing.** Neither owns; both subscribe to a log of stock-level changes.
- **Duplicate.** Each keeps its own view, synced asynchronously. Accept eventual consistency.

Flag shared-table edges in the output — they're where the extraction actually hurts.

## Extraction order

Extract **most stable first** (lowest instability — depended on, doesn't depend on much). Those become foundation services. Extract **leaf features last** — they depend on everything.

## Do not

- **Do not** propose boundaries that cut through a single database table. That's not a boundary, it's a distributed monolith waiting to happen.
- **Do not** trust the directory structure without measuring. `reports/` having low cohesion is common — it's where miscellaneous features go to die.
- **Do not** ignore co-change data. Two files that always change together are coupled even if there's no import between them — maybe they share a config format, or a protocol.
- **Do not** propose a 15-way split. 2–4 components is a plan. 15 is a reorg.

## Output format

```
## Dependency graph
Nodes: <N> (granularity: <file | class | function>)
Edges: <M> (<breakdown by type>)

## Current structure evaluation
| Directory | Cohesion | Coupling | Keep / Restructure |
| --------- | -------- | -------- | ------------------ |

## Proposed components
### <Component name>
Contents: <files/modules>
Cohesion: <score>
Interface (incoming): <what others call into this component>
Dependencies (outgoing): <what this component needs>
Shared data: <tables/types crossing the boundary — THE HARD PART>

## Extraction order
1. <component> — instability <score>, extract first
...

## Blockers
<shared tables, circular component deps, god objects that touch everything>
```
