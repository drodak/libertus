---
status: accepted
date: 2026-10-01
decision-makers: drodak
---

# PostgreSQL as the Default Data Store, with Criteria for Justified NoSQL Exceptions

## Context and Problem Statement

The team wants to limit the number of storage technologies in operation, defaulting to PostgreSQL
wherever feasible, while leaving room for a NoSQL store where there is a genuinely justified case.
Without explicit criteria, "justified case" is a judgment call that will be made inconsistently —
either every module defaults to Postgres even where it's a poor fit, or NoSQL stores get adopted
ad hoc without anyone weighing the operational cost of a second storage technology.

What is the default data store, and what bar must a use case clear to justify a non-relational
alternative?

## Decision Drivers

* Minimize the number of storage technologies the team operates, monitors, and backs up.
* PostgreSQL's extensions (JSONB, `pg_trgm`, `pgvector`, partitioning, `LISTEN`/`NOTIFY`) already
  cover a large share of use cases that would otherwise look like "we need NoSQL."
* The modular monolith ([ADR-0004](0004-modular-monolith-with-spring-modulith.md)) benefits from
  one shared relational store for transactional consistency across modules in the same process.
* Must not become so rigid that a genuinely poor-fit data shape gets forced into relational
  tables anyway — that trades one kind of complexity for another.
* No concrete NoSQL use case exists yet; the criteria must be concrete enough to apply later
  without this ADR having to be revisited for every new case.

## Considered Options

* PostgreSQL only, no exceptions
* PostgreSQL by default, with documented criteria gating NoSQL exceptions
* Polyglot persistence (pick the best-fit store per module from the start)

## Decision Outcome

Chosen option: "PostgreSQL by default, with documented criteria gating NoSQL exceptions", because
it keeps the default operational footprint to one database engine while giving a concrete,
checkable bar for the cases where that default genuinely doesn't fit, rather than either banning
exceptions outright or leaving "justified" undefined.

A module MAY introduce a non-relational store only when **all** of the following hold:

1. The access pattern doesn't fit relational modeling well even with PostgreSQL's own extensions
   (e.g., genuinely schema-less per-tenant documents, high-ingest time-series at a volume
   partitioned Postgres tables don't comfortably serve, full-text/vector search beyond what
   `pg_trgm`/`pgvector` can serve, graph traversal, or a pub/sub/ephemeral-cache need).
2. The relevant PostgreSQL extension or feature (JSONB, `pg_trgm`, `pgvector`, partitioning,
   `LISTEN`/`NOTIFY`) has been explicitly considered and rejected, with the reason recorded in the
   plan or ADR introducing the exception.
3. The choice is recorded in its own ADR, naming the specific technology, the owning module, and
   its operational cost (who runs/monitors/backs it up).
4. The new store does not become a second system of record for data Postgres already owns — no
   dual-write without a documented reconciliation strategy in that same ADR.

### Consequences

* Good, because the default operational footprint stays at one database engine for the large
  majority of modules.
* Good, because Postgres extensions (JSONB, `pgvector`, etc.) resolve many apparent "we need
  NoSQL" cases without adding a technology at all.
* Good, because the four criteria give reviewers something concrete to check instead of a
  subjective "is this justified?" debate each time.
* Neutral, because applying the criteria to a borderline case still requires judgment; they
  narrow the debate, they don't eliminate it.
* Bad, because a module team that wants a NoSQL store faces real friction (the ADR plus the
  criteria) before they can self-serve that choice — intentional, given the stated goal of
  limiting technology count.

### Confirmation

A pull request introducing a new, non-Postgres storage dependency without a linked ADR addressing
all four criteria is a violation of the project constitution's Principle II (Recorded Architecture
Decisions) and is rejected at review.

## Pros and Cons of the Options

### PostgreSQL only, no exceptions

* Good, because it is the simplest possible rule and the smallest possible operational footprint.
* Bad, because some data shapes (e.g. a true document-per-tenant store, or vector search at a
  scale beyond `pgvector`) will eventually get forced into relational tables anyway, trading one
  complexity for another.

### PostgreSQL by default, criteria-gated exceptions

* Good, because it keeps the default narrow while leaving a real, bounded escape hatch.
* Good, because the criteria double as documentation for why Postgres extensions should be tried
  first.
* Bad, because it is more process than a flat rule — a team must write the exception ADR, not
  just add a dependency.

### Polyglot persistence by default

* Good, because each module could pick the objectively best-fit store from day one.
* Bad, because it directly contradicts the goal of minimizing the number of storage technologies,
  and would multiply operational burden (backup, monitoring, expertise) across the team for
  marginal per-module benefit.

## More Information

* PostgreSQL JSONB and full-text/vector extensions are the first things to evaluate before any
  NoSQL exception ADR is written — see `pg_trgm` and `pgvector` documentation.
* This ADR intentionally names no specific NoSQL technology; the first exception ADR that is
  actually needed picks one, informed by its concrete use case.
