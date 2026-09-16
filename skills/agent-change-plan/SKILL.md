---
name: agent-change-plan
description: Build an evidence-driven, agent-safe change plan from code structure, contracts, data flow, persistence, static dependencies, and Git-derived empirical coupling. Identifies affected scope, invariants, risks, validation steps, migration order, and rollback strategy before implementation.
---

# Agent Change Plan

## Purpose

Turn a requested software change into a **bounded, evidence-driven implementation plan**.

The plan should answer:

> **What must change, what must be understood first, what could be affected, and how can the change be made incrementally and safely?**

This skill is intended to run after architectural analysis and before implementation.

It should use both current codebase evidence and historical evidence from Git when available.

The goal is not to produce the smallest possible plan. The goal is to produce a plan whose **reasoning boundary and blast radius are explicit**.

---

## Core Workflow

Use this pipeline:

```text
REQUEST
   ↓
SCOPE
   ↓
CURRENT ARCHITECTURE
   ↓
STATIC + DATA + STATE + DB GRAPH
   ↓
HISTORICAL CHANGE GRAPH
   ↓
AFFECTED-SCOPE ANALYSIS
   ↓
CONTRACT / INVARIANT ANALYSIS
   ↓
CHANGE STRATEGY
   ↓
INCREMENTAL PLAN
   ↓
VALIDATION + ROLLBACK
```

Do not begin implementation until the plan has identified the important unknowns.

### 1. Understand the Requested Change

Translate the request into explicit change intent.

Identify: desired behavior, current behavior, affected user/system behavior, constraints, compatibility requirements, performance requirements, persistence requirements, API requirements, non-functional requirements, explicit non-goals.

Separate:

```text
Requested: "Add X"
Required interpretation: "What behavior must exist after the change?"
```

Do not infer unnecessary requirements.

If the request is ambiguous, record the ambiguity rather than silently inventing a design.

### 2. Establish the Change Boundary

Identify the smallest plausible starting scope, e.g.:

```text
src/payment.py
src/payment_service.py
tests/test_payment.py
```

Then expand the scope using evidence, into:

- **Directly affected** — known to require modification.
- **Dependency context** — components that must be understood because the target depends on them.
- **Historically coupled** — components that repeatedly change with the target.
- **Contract consumers** — components that depend on behavior/API/schema being changed.
- **Validation surface** — tests, fixtures, migrations, integration environments, or operational tooling required to verify the change.
- **Potentially affected** — plausible candidates with incomplete evidence.

Never silently treat all discovered relationships as required changes.

### 3. Gather Evidence

Use evidence sources in descending order of directness: current source code, types/interfaces/schemas, tests, static dependency graph, runtime/data-flow information, database/persistence relationships, Git history, documentation, naming/convention inference.

Distinguish: observed, strong inference, weak inference, unknown.

Do not convert weak inference into a requirement.

### 4. Use the Architecture Graph

Consult available architecture information: static dependencies, runtime dependencies, data flow, state ownership, persistence, module boundaries, dependency direction, cycles, historical coupling.

The change plan should identify which graph relationships matter to the requested change.

Example: if the requested change modifies payment state, the database node is part of the change boundary even if no application code imports the schema directly (`PaymentService → PaymentRepository → payments table`).

### 5. Use Git History as Empirical Evidence

When Git history is available, inspect historical change behavior: repeated co-change, conditional change probability, Jaccard similarity, temporal ordering, repeated cross-boundary changes, previous implementations of similar changes, reverted attempts, migration sequences, tests repeatedly changed alongside the target.

Example:

```text
payment.py changed in 52 commits
invoice.py changed with payment.py in 31 commits
P(invoice | payment) = 0.60
```

Interpretation: historical coupling exists; the cause is unknown; inspect `invoice.py` before deciding whether it must change.

Do not write "invoice.py is a dependency of payment.py" unless independent architectural evidence supports that claim.

### 6. Identify Change Propagation

Construct a local propagation map, e.g.:

```text
Requested: Change Payment.status

Likely propagation:
Payment
  ├── PaymentService
  ├── PaymentRepository
  ├── payments schema
  ├── API response
  └── payment tests

Historical:
  └── InvoiceService frequently changes with Payment
```

For every propagation edge, classify it: `STATIC`, `RUNTIME`, `DATA`, `STATE`, `DATABASE`, `CONTRACT`, `HISTORICAL`, `UNKNOWN`.

This prevents different kinds of relationships from being collapsed into one dependency concept.

### 7. Identify Contracts and Invariants

Before proposing implementation steps, identify what must remain true.

Look for: public APIs, function signatures, type contracts, database constraints, serialization formats, event schemas, validation rules, state-machine transitions, authorization rules, idempotency requirements, ordering guarantees, backwards compatibility, tests encoding behavior.

Create an invariant list, e.g.:

```text
Invariant 1: A completed payment cannot return to pending.
Invariant 2: The API must continue accepting the existing request format.
Invariant 3: payments.status must contain only valid enum values.
```

The implementation plan must explicitly state which step protects each invariant.

### 8. Determine the Change Strategy

Choose deliberately between:

- **Additive change** — add a new implementation without modifying existing behavior. Prefer when compatibility matters, the existing abstraction is stable, migration can happen incrementally.
- **Adapter / wrapper** — preserve an existing interface while introducing new behavior. Use when old consumers cannot migrate simultaneously, boundaries need to remain stable.
- **Direct modification** — modify the existing implementation. Use when the abstraction itself must change, or maintaining parallel implementations would increase complexity.
- **Refactor first** — restructure before implementing the feature. Use only when the current structure materially prevents a safe change. Do not refactor merely because the code could theoretically be cleaner.

### 9. Minimize Semantic Blast Radius

Prefer plans where each step has a narrow responsibility.

Bad: "Rewrite payment subsystem. Update API. Change database. Refactor tests."

Better:

```text
1. Introduce new status representation.
2. Add compatibility mapping.
3. Update persistence layer.
4. Update service behavior.
5. Update API serialization.
6. Migrate consumers.
7. Remove compatibility layer after validation.
```

The sequence should make intermediate states understandable.

### 10. Respect Dependency Direction

Do not introduce new edges that invert established architectural boundaries without explicitly identifying the reason.

Before adding a dependency `A → B`, ask: is A allowed to know about B? Does B already depend on A? Does this create a cycle? Is the dependency actually data rather than behavior? Should an interface or boundary object be introduced? Would the dependency make future changes broader?

If the dependency direction is intentionally changed, document why.

### 11. Protect State Ownership

For every state touched by the change identify: owner, readers, writers, persistence, lifecycle.

Example:

```text
State: Payment.status
Owner: Payment domain
Readers: PaymentService, InvoiceService, API serializer
Writers: PaymentService
Persistence: payments.status
```

Avoid introducing a second owner. If ownership must move, make the transfer explicit and preferably incremental.

### 12. Treat Database Changes as Migrations

Database changes require their own plan.

For schema changes identify: old schema, new schema, readers, writers, migration order, compatibility window, backfill requirements, rollback limitations.

Prefer expand/contract migration when compatibility is required:

```text
EXPAND → support old + new → MIGRATE → switch consumers → CONTRACT → remove old representation
```

Never assume database changes are equivalent to ordinary source edits.

### 13. Define Implementation Steps

Each implementation step should contain: Step, Target, Purpose, Dependencies, Invariant(s) protected, Expected affected scope, Validation, Rollback.

Example:

```text
Step 3 — Update persistence layer
Target: PaymentRepository
Purpose: Persist the new status representation.
Depends on: Step 1 status type.
Protects: Invariant 3.
Expected scope: PaymentRepository + repository tests.
Validation: Run persistence tests and migration verification.
Rollback: Revert repository mapping while compatibility column remains available.
```

Steps should be ordered so that intermediate commits are understandable and preferably buildable/testable.

### 14. Define Validation at Every Boundary

Do not postpone all validation until the end.

For each step specify the cheapest meaningful validation: unit tests, type checking, linting, schema validation, migration dry-run, integration tests, API contract tests, static dependency analysis, runtime smoke test.

Validation should test both new behavior and preserved behavior when compatibility matters.

### 15. Use Tests as Architectural Evidence

Tests are not only validation artifacts. They reveal contracts, state transitions, expected side effects, coupling, public behavior, important invariants.

If a test repeatedly changes with the target component, treat that as historical behavioral coupling.

If important behavior has no tests, record a **validation gap** rather than assuming the behavior is unimportant.

### 16. Account for Historical Failed Attempts

Git history may contain previous attempts at the same or similar change.

Search for: reverted commits, follow-up fixes, repeated bug fixes, migrations, compatibility shims, commits with similar messages, files that repeatedly changed after the target.

Example: an attempt that changed schema + application simultaneously and required follow-up fixes implies the plan should separate schema compatibility from application rollout.

A historical failure is evidence, not necessarily a rule.

### 17. Identify Risk Concentrations

Highlight areas where multiple signals agree, e.g. a file with high static fan-out + high historical fan-out + high change entropy + shared database tables + multiple state writers is a high-risk change surface.

Prioritize investigation where static coupling, historical coupling, and state/persistence coupling overlap.

### 18. Produce a Change Graph

Represent the planned change as a graph, marking edges by type: `[STATIC]`, `[DATA]`, `[DB]`, `[CONTRACT]`, `[HISTORICAL]`.

The graph should describe the change, not the entire repository.

### 19. Estimate Blast Radius

Where historical propagation estimates are available, include: direct scope, historical scope, expected scope, worst observed scope, confidence.

If Monte Carlo simulation is available, report distributions rather than a single prediction (e.g. median affected components, 90th percentile, maximum observed, % crossing subsystem boundary, confidence).

Do not interpret this as a guaranteed future outcome.

### 20. Define Rollback Strategy

Every non-trivial change should have a rollback story.

Identify whether rollback is: trivial, code-revertable, migration-reversible, compatibility-dependent, irreversible.

For database changes explicitly distinguish code rollback from data rollback — they are not necessarily the same operation.

If rollback is impossible, state that explicitly.

### 21. Define the Commit Strategy

Structure implementation into coherent commits, e.g.:

```text
1. Add new type / interface.
2. Add compatibility behavior.
3. Add persistence support.
4. Migrate service logic.
5. Migrate consumers.
6. Remove compatibility behavior.
```

Each commit should have one architectural purpose, be reviewable, be independently understandable, avoid unrelated cleanup, and preserve a useful Git history.

Git history is part of the project's future architectural evidence. Do not destroy useful history with unrelated formatting or mass refactoring.

---

## Final Change Plan Format

Produce the following report:

### Change Intent

Requested / expected behavior / constraints.

### Scope

| Component | Classification | Reason | Confidence |
|---|---|---|---|

### Relevant Architecture

Graph.

### Historical Evidence

Repeated co-change, conditional probabilities, relevant commits, temporal patterns, previous attempts, important caveats.

### Contracts and Invariants

Numbered list.

### Proposed Strategy

Additive / adapter-based / direct modification / refactor-first, and why.

### Implementation Plan

For every step: Target, Purpose, Depends on, Protects, Validation, Rollback.

### Change Graph

Affected components and relationship types.

### Blast Radius

Direct scope, historical scope, expected scope, uncertainty.

### Database / Persistence Plan

Schema changes, migration order, compatibility window, backfill, rollback (if applicable).

### Validation Plan

Grouped into local, integration, contract, database, system.

### Rollback Plan

Code rollback, database rollback, data compatibility, irreversible operations.

### Commit Plan

List recommended coherent commits.

### Risks

Ranked by impact, likelihood, confidence, mitigation. Do not produce an overall "risk score" that hides individual hazards.

### Unknowns

Explicitly list information that could materially change the plan.

### Agent Handoff

End with a compact machine-readable handoff, sufficient for another agent to begin implementation without reconstructing the entire analysis:

```yaml
change:
  summary: "..."
  strategy: additive

scope:
  direct:
    - src/payment.py
  context:
    - src/payment_service.py
  historical:
    - src/invoice.py

contracts:
  - "..."

invariants:
  - "..."

steps:
  - id: 1
    target: "..."
    validation: "..."
  - id: 2
    target: "..."
    validation: "..."

risks:
  - "..."

rollback:
  code: "..."
  database: "..."

unknowns:
  - "..."
```

---

## Rules

- Do not implement while planning.
- Do not claim causality from Git co-change alone.
- Do not silently expand scope.
- Do not treat every dependency as a required modification.
- Do not hide uncertainty.
- Do not introduce abstractions merely to make the graph look cleaner.
- Do not ignore database coupling.
- Do not create a second owner for existing state without explicit justification.
- Do not mix unrelated cleanup into the change plan.
- Prefer incremental, reversible changes.
- Preserve existing contracts unless changing them is part of the request.
- Use historical failures as evidence.
- Make every important planning decision traceable to evidence.
- Optimize for predictable reasoning and change impact, not minimum line count.
- If evidence is insufficient, say Unknown.

## Completion Criteria

A change plan is complete when another capable agent can answer all of these without rereading the entire repository:

- What exactly are we changing?
- What definitely needs to change?
- What needs to be understood first?
- What is historically coupled?
- Which contracts must survive?
- Which invariants must remain true?
- Who owns the affected state?
- Which database objects are involved?
- What is the expected blast radius?
- What is the safest implementation order?
- How will each step be validated?
- How can the change be rolled back?
- What remains unknown?

If these questions cannot be answered, the plan is incomplete.
