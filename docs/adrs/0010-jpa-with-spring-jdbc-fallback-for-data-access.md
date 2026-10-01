---
status: accepted
date: 2026-10-01
decision-makers: drodak
---

# JPA as the Primary Data Access Technology, Spring JDBC as the Documented Fallback

## Context and Problem Statement

With PostgreSQL as the default store ([ADR-0009](0009-postgresql-as-default-data-store.md)), the
backend needs a data access technology. The team's stated preference is JPA for most access, with
Spring JDBC used when JPA isn't feasible. Left undefined, "when JPA isn't feasible" will be
decided inconsistently per developer; some will drop to JDBC prematurely for things JPA handles
fine, others will fight JPA for cases where JDBC would plainly be simpler and faster.

Which data access technology is the default, under what conditions is the fallback appropriate,
and how is schema evolution managed across both?

## Decision Drivers

* Reuse existing team experience with JPA/Hibernate and Spring Data repositories.
* JPA has known weak points: bulk/batch writes, reporting/analytics-shaped queries, PostgreSQL-
  specific features (`LISTEN`/`NOTIFY`, advisory locks, `RETURNING`, native JSONB operators), and
  pure read-model projections that don't map to an entity graph.
* Must not add a third data-access technology (e.g. jOOQ, MyBatis) when the team's own ask was
  specifically "JPA, falling back to Spring JDBC."
* Schema evolution must have one source of truth regardless of which access technology touches a
  given table.
* Should fit the modular monolith's package-based module boundaries (ADR-0004) — a module's
  repositories and entities, whichever technology they use, stay inside that module's package.

## Considered Options

* JPA only, no fallback
* JPA by default, Spring JDBC fallback by documented criteria (chosen)
* JPA by default, jOOQ fallback instead of Spring JDBC
* MyBatis instead of JPA

## Decision Outcome

Chosen option: "JPA by default, Spring JDBC fallback by documented criteria", because it matches
the team's explicit preference, keeps the data-access surface to exactly two well-understood
technologies instead of three, and a clear set of criteria stops the fallback from being used
arbitrarily in either direction.

Spring Data JPA (`@Entity`, `Repository`) is the default for a module's data access. A module MAY
use Spring JDBC (`JdbcTemplate`/`NamedParameterJdbcTemplate`) instead, for a specific query or
operation, when at least one holds:

1. The query is reporting/analytics-shaped (aggregations, window functions, CTEs) where entity
   mapping adds no value and would hurt clarity or performance.
2. A bulk/batch write materially outperforms JPA's entity lifecycle (large batch inserts, updates,
   upserts), and JPA batching has been tried and found insufficient.
3. A PostgreSQL-specific feature is required that Hibernate doesn't model well (`LISTEN`/`NOTIFY`,
   advisory locks, `RETURNING`, native JSONB operators).
4. The result is a pure read-model projection for a query endpoint that doesn't correspond to any
   persisted aggregate/entity.

Schema migrations: Flyway is the single source of truth for schema evolution, regardless of
whether a table is accessed via JPA, JDBC, or both. Hibernate's automatic DDL generation
(`ddl-auto`) is restricted to `validate` in every profile except local development convenience,
where it MAY be `none` while Flyway migrations are authored by hand.

### Consequences

* Good, because most CRUD-shaped modules get Spring Data JPA's repository productivity with no
  extra technology to learn.
* Good, because the fallback criteria give reviewers a concrete check instead of an open-ended
  "is JDBC appropriate here?" debate.
* Good, because Flyway gives one auditable migration history no matter which access technology
  touches a table.
* Neutral, because mixing JPA entities and hand-written JDBC against the same table within one
  module needs care: Hibernate's first-level/session cache can go stale relative to a JDBC write
  made in the same transaction, and this must be understood by whoever writes that code.
* Bad, because a team member must learn two data-access idioms (JPA repositories and JDBC
  templates) instead of one, even though both are mainstream Spring idioms already.

### Confirmation

A pull request introducing a `JdbcTemplate`/`NamedParameterJdbcTemplate` usage states, in the PR
description or a code comment, which of the four criteria applies; reviewers check this at review
time. CI runs Flyway migrations against a test database (e.g. via Testcontainers) as part of the
test suite, so a migration that doesn't apply cleanly fails the build.

## Pros and Cons of the Options

### JPA by default, Spring JDBC fallback

* Good, because it matches the team's stated preference exactly.
* Good, because it keeps the data-access surface to two technologies, both already idiomatic
  Spring.
* Neutral, because the fallback criteria require occasional judgment calls at the margins.

### JPA only, no fallback

* Good, because it is the simplest rule — one technology, no criteria to apply.
* Bad, because it forces reporting-shaped queries, bulk writes, and PostgreSQL-specific features
  through an ORM not well suited to them, which is the exact friction the team's stated preference
  for a documented fallback was meant to avoid.

### JPA by default, jOOQ fallback

* Good, because jOOQ gives type-safe, fluent SQL that's arguably nicer than JDBC's string-based
  templates for complex queries.
* Bad, because it is a technology the team did not ask for, adding a third thing to learn and
  license (jOOQ's non-free database drivers) when Spring JDBC already covers the stated need.

### MyBatis instead of JPA

* Good, because it gives fine-grained control over SQL for every query, not just fallback cases.
* Bad, because it abandons the team's stated JPA-first preference and existing JPA experience
  entirely, for no driver mentioned in the stated requirements.

## More Information

* Flyway and Spring Boot: <https://docs.spring.io/spring-boot/reference/data/sql.html#data.sql.migration.flyway>
* The JPA/JDBC mixing pitfall (stale session cache vs. a concurrent JDBC write) should be called
  out in onboarding docs for the data-access layer, not just in this ADR.
