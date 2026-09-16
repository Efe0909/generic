---
name: agent-safe-change
description: Lightweight instruction set for making software changes with explicit dependencies, honest effects, stable contracts, composability, predictable blast radius, and minimal accidental coupling.
---

# Agent-Safe Change

Optimize for:

> Make the smallest coherent change that preserves contracts and leaves dependencies, effects, ownership, and blast radius explicit.

## Preserve Contracts

Before changing an established function or abstraction:

- determine its contract
- inspect important callers
- ask whether composition, an adapter, wrapper, or new implementation can satisfy the requirement

Modify existing semantics when the existing abstraction is genuinely wrong; do not create meaningless `v2`/`new` variants merely to avoid refactoring.

## Keep Effects Honest

Do not introduce hidden:

- IO
- database access
- network calls
- mutation
- global state
- configuration/environment reads

Keep pure transformations separate from effects when practical.

## Prefer Explicit Data Flow

Prefer:

```text
input → transform → transform → output
```

over hidden shared state.

Make lifecycle dependencies explicit through values/types where practical.

## Keep Operations Composable

Prefer small, semantically coherent operations.

Do not broaden an existing function merely because a new use case resembles it.

Prefer semantic cohesion over small line counts.

## Protect Ownership

Before changing state:

- identify its owner
- identify its mutators
- identify its invariants

Do not casually introduce another owner.

## Minimize Blast Radius

Before editing, identify:

1. callers
2. dependencies
3. side effects
4. persistence interactions
5. relevant tests
6. historical coupling when Git history is available

Prefer changes whose affected surface is predictable.

## Make Failure Explicit

Do not silently swallow errors or overload null/sentinel values with unrelated meanings.

Preserve the project's established error model unless deliberately improving it.

## Avoid Accidental Coupling

Do not create abstractions merely to eliminate duplication.

If two concepts have independent reasons to change, some duplication may be safer.

## Preserve Dependency Direction

Do not introduce:

- dependency cycles
- lower-level → higher-level dependencies
- domain → infrastructure leakage
- UI → database shortcuts
- unrelated cross-module knowledge

## Consider the Database

Treat schema, constraints, transactions, migrations, and persistence behavior as part of the change's dependency surface.

Do not change application behavior while ignoring relevant database invariants.

## Document Why

If the change introduces or preserves something surprising, document the rationale.

Do not waste comments restating obvious code.

## Before Finishing

Ask:

```text
Did I make any effect less explicit?
Did I increase hidden coupling?
Did I broaden an abstraction unnecessarily?
Did I introduce another state owner?
Did I create a dependency cycle?
Did I increase blast radius?
Did I break an existing contract?
Did I ignore a database dependency?
Can the changed behavior be understood locally?
Is the change reversible?
```

If yes, reconsider or explicitly justify it.

If architecture remains uncertain, state the uncertainty instead of inventing certainty.
