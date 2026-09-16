---
name: agent-code-audit
description: Audit a codebase or selected scope for agent-friendly software architecture, reasoning locality, explicit effects, coupling, contracts, change blast radius, and database/persistence hazards. Produces scored findings with confidence and evidence.
---

# Agent-Friendly Code Audit

## Purpose

Audit software for properties that make it easy for an unfamiliar agent or human to safely understand and modify.

Primary objective:

> Minimize the implicit knowledge required to safely change a piece of software.

Do not judge code by ideological adherence to OOP, functional, imperative, or declarative programming. Judge whether dependencies, effects, state, contracts, and change boundaries are discoverable.

## Scope

The audit may target:

- a function
- a file
- a module/package
- a subsystem
- the entire repository

Always state the audited scope.

A local audit must not be presented as proof of repository-wide quality.

## Evidence Discipline

Separate:

- observed facts
- strong inferences
- weak signals
- unknowns

Never interpret lack of evidence as evidence of good architecture.

When context is insufficient, use Unknown.

## Scoring

Score each applicable category from 0–5:

- 5 — excellent
- 4 — good
- 3 — mixed/acceptable
- 2 — concerning
- 1 — poor
- 0 — severe/systemic problem
- N/A — not applicable or insufficient evidence

Report:

- category scores
- overall score
- confidence
- critical findings
- hotspots
- prioritized remediation

The overall score must not conceal critical hazards.

## Audit Categories

### 1. Effect Honesty

Check:

- Can functions perform IO without making it apparent?
- Can `get_*`, `parse_*`, `calculate_*`, etc. mutate state or access external systems?
- Are global/environment/database/network/cache/clock dependencies hidden?
- Are pure transformations separated from effects?

Symptoms:

- `get_config()` secretly reads files/env/network
- constructors perform IO
- utility functions mutate global state
- innocent-looking calls trigger writes

### 2. Dependency Visibility

Check:

- Are important dependencies explicit?
- Are globals, singletons, service locators, thread-local state, or ambient contexts used?
- Can a caller determine important prerequisites from the interface?

Symptoms:

- hidden service acquisition
- global repositories
- implicit dependency injection
- environment-sensitive behavior without a visible boundary

### 3. Locality of Reasoning

Check:

- Can a component be understood mostly from itself and direct dependencies?
- Does a small change require reading distant unrelated code?
- Is behavior hidden in decorators, registries, reflection, callbacks, or runtime wiring?

Symptoms:

- "follow this through many files"
- distant registration changes local behavior
- magic dispatch
- large amounts of tribal knowledge

### 4. Semantic Coupling

Check:

- Are components coupled by undocumented assumptions?
- Does one action have to occur before another?
- Is lifecycle/order dependency explicit?
- Is shared mutable state used as communication?

Symptoms (implicit dependency between steps):

```text
initialize()
do_something()
cleanup()
```

### 5. Composition and Cohesion

Check:

- Do functions represent coherent transformations/effects?
- Are operations composable?
- Are IO, policy, transformation, and persistence unnecessarily combined?
- Are "do everything" functions present?

Judge semantic cohesion rather than line count.

### 6. Extension vs Repurposing

Check:

- Are existing functions repeatedly modified for unrelated use cases?
- Are flags/options accumulating?
- Are established contracts stable?
- Could behavior be added through composition, adapters, wrappers, or new implementations?

Do not reward endless v2/new copies. The goal is stable contracts and clean extension.

### 7. Mutation and State Ownership

Check:

- Is mutation obvious?
- Does important state have one clear owner?
- Can unrelated modules mutate it?
- Are invariants protected at the ownership boundary?

Symptoms:

- shared mutable dictionaries
- global caches
- public mutable state
- multiple modules directly changing the same domain state

### 8. Dependency Direction

Check:

- Do dependencies form a mostly directional graph?
- Are there cycles?
- Does low-level code depend on high-level policy?
- Are layer boundaries violated?

Flag cycles (`A → B → C → A`) and cross-layer shortcuts such as `UI → database`, `domain → HTTP client`, `infrastructure → UI`.

### 9. Data Flow Explicitness

Check:

- Can important data transformations be followed?
- Are intermediate representations visible?
- Is data hidden inside mutable objects?
- Are transformations distinguishable from effects?

Prefer an explicit pipeline (`raw → parsed → validated → normalized → domain → persisted`) over opaque orchestration.

### 10. Failure Explicitness

Check:

- Are failure modes discoverable?
- Are exceptions swallowed?
- Is null/None overloaded?
- Are errors handled at the appropriate layer?

Flag patterns such as `except Exception: return None` when they destroy failure information.

### 11. Policy vs Mechanism

Check:

- Is business policy mixed into infrastructure?
- Do generic mechanisms make domain decisions?
- Can policy change independently from mechanism?

Separate "how" from "what/why".

### 12. Abstraction Quality

Check:

- Does each abstraction represent one coherent concept?
- Are interfaces overly broad?
- Are generic abstractions created prematurely?
- Are many unrelated options/flags compensating for a weak abstraction?

Do not reward abstraction for its own sake.

### 13. Duplication vs Coupling

Do not automatically penalize duplication.

Ask:

- Were unrelated concepts coupled merely to eliminate duplication?
- Would independent copies have clearer contracts?
- Does changing one behavior unexpectedly affect another?

Prefer duplication over accidental semantic coupling.

### 14. Magic and Hidden Control Flow

Check for:

- implicit registration
- reflection
- metaprogramming
- monkey patching
- event buses
- hidden decorators
- runtime discovery
- ambient configuration

Magic is acceptable when its boundary and behavior are discoverable.

### 15. Contract and Invariant Visibility

Check whether important assumptions are encoded in types, schemas, assertions, tests, documentation, or database constraints.

Ask: could an unfamiliar agent determine valid inputs, outputs, invariants, and failure modes without reconstructing tribal knowledge?

### 16. Change Blast Radius

Estimate whether changes are local, module-wide, subsystem-wide, or repository-wide.

Check: number of callers, number of consumers, state ownership, persistence interactions, cross-module coupling, relevant tests.

Prefer predictable blast radius.

### 17. Reversibility

Check:

- Can changes be introduced incrementally?
- Can implementations be swapped behind stable boundaries?
- Can migration steps be tested independently?
- Does a feature require a large irreversible rewrite?

### 18. Documentation of Why

Check:

- Are unusual constraints explained?
- Are architectural decisions recorded?
- Do comments explain rationale rather than syntax?

Ask: would a future agent understand why this weird thing exists?

### 19. Paradigm Fitness

Evaluate whether the chosen programming style makes the dominant relationship explicit: OOP for state + invariants + lifecycle, functional style for transformations, imperative style for explicit effect sequencing, declarative style for desired state/constraints.

Do not reward paradigm purity.

### 20. Persistence and Database Coupling

Treat the database as part of the architecture.

Check:

- Are queries scattered through business logic?
- Does domain logic depend on database details?
- Are tables directly manipulated by many unrelated modules?
- Are transaction boundaries explicit?
- Do DB constraints and application invariants agree?
- Are triggers, cascades, views, stored procedures, generated columns, and migrations relevant to behavior?

Inspect the chain `application → data access → persistence → database schema`, and the reverse influence of database behavior on application semantics.

## Fast Symptom Scan

When operating with little context, answer as many as possible with YES/NO/UNKNOWN:

- Hidden IO/effects?
- Hidden dependencies?
- Global/ambient state?
- Implicit lifecycle/order?
- Large multi-purpose functions?
- Hidden mutation?
- Multiple state owners?
- Circular dependencies?
- Cross-layer shortcuts?
- Broad/flag-heavy abstractions?
- Swallowed/ambiguous errors?
- Magic runtime wiring?
- Scattered database access?
- High historical co-change?
- Large/unpredictable blast radius?

The symptom scan is not itself the score. Use it to locate investigation targets.

## Git History

If Git history is available, analyze it as evidence of empirical coupling. Do not equate co-change with dependency — distinguish static dependency, runtime dependency, data dependency, and historical/temporal coupling.

Useful signals: repeated co-change, conditional co-change probability, change frequency, change fan-out, change entropy, asymmetric temporal coupling. For files A and B, consider `P(B changes | A changes)` and `P(A changes | B changes)`, and normalized co-change measures such as Jaccard similarity.

Strong repeated co-change is a coupling signal requiring investigation.

## Output

### Agent-Friendliness Score

```text
Overall: X/5
Confidence: High/Medium/Low
Scope: ...

Critical hazards: N
Major hazards: N
```

### Category Scores

| Category | Score | Confidence | Evidence |
|---|---:|---|---|
| Effect honesty | X/5 | … | … |
| Dependency visibility | X/5 | … | … |
| *(one row per applicable category)* | | | |

### Fast Symptom Scan

Compact YES/NO/UNKNOWN table.

### Hotspots

For each hotspot:

```text
Location:
Observed behavior:
Why it increases reasoning cost:
Likely blast radius:
Confidence:
Recommended direction:
```

### Historical Coupling

When history is available:

```text
Strongest co-change relationships:
Potential hidden dependencies:
High change-entropy nodes:
High change-fan-out nodes:
Static/history disagreements:
```

### Top 5 Actions

Prioritize actions that reduce implicit coupling and reasoning cost.

### Unknowns

Explicitly state what was not observable.
