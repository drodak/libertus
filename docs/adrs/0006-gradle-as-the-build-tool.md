---
status: accepted
date: 2026-10-01
decision-makers: drodak
---

# Gradle as the Build Tool, Single-Module Backend Layout

## Context and Problem Statement

The backend needs a build tool for compiling, testing, and packaging the Spring Boot application,
matching the team's existing experience. The repository is also a monorepo hosting more than one
deployable ([ADR-0007](0007-github-monorepo-for-backend-and-frontend.md)), and the backend's
internal module boundaries are enforced at the package level by Spring Modulith
([ADR-0004](0004-modular-monolith-with-spring-modulith.md)) rather than at the build-module level.
The build layout needs to fit both facts: isolate the backend from other deployables in the repo,
without introducing a Gradle subproject per business module that Spring Modulith doesn't need.

## Decision Drivers

* Reuse existing team experience, per the stated preference for Gradle.
* The backend's module boundaries are already enforced by Spring Modulith at the package level;
  the build tool does not need to duplicate that enforcement via subprojects.
* The build must coexist cleanly with the other deployable(s) in the monorepo
  (ADR-0007) without entangling their build lifecycles.
* Should support a Java toolchain pin (ADR-0005) and reproducible builds in CI
  ([ADR-0008](0008-github-actions-for-continuous-integration.md)).

## Considered Options

* Gradle (Kotlin DSL), single backend module
* Gradle (Groovy DSL), single backend module
* Maven, single backend module

## Decision Outcome

Chosen option: "Gradle (Kotlin DSL), single backend module", because Gradle is the team's stated
preference over Maven, and Kotlin DSL gives IDE-checked, type-safe build scripts — a meaningful
benefit for a project that will have more than one Gradle-built component in the repo over time.

Conventions:

* The backend lives under `backend/` as its own Gradle build (`backend/build.gradle.kts`,
  `backend/settings.gradle.kts`), separate from the frontend application's build tooling at
  `frontend/` (ADR-0007). This is a repo-layout boundary, not a Spring Modulith module boundary.
* The backend is **one** Gradle module/project, not one Gradle subproject per Spring Modulith
  application module. Spring Modulith's package-based boundaries do not require, and would gain
  nothing from, a matching Gradle subproject per business module.
* Shared build conventions (Java toolchain version, common plugins, code-style/quality plugins)
  live in a Gradle convention plugin under `backend/build-logic/`, applied to the single backend
  module, so they are defined once even though there is currently one consumer.
* The Gradle Java toolchain is pinned to Java 25 (ADR-0005).
* If a Spring Modulith application module is later extracted into its own deployable, splitting
  it into its own Gradle subproject (or its own top-level repo entry) happens at that time, as
  part of the ADR that makes that extraction decision — not preemptively.

### Consequences

* Good, because the build stays simple (one module) while Spring Modulith does the actual
  decoupling work, avoiding duplicate, uncoordinated boundary mechanisms.
* Good, because Kotlin DSL catches build-script typos and misused APIs at compile time rather
  than at runtime, and matches the current Spring Initializr default.
* Good, because keeping the backend's Gradle build under `backend/` isolates it from whatever
  build tooling the frontend application uses.
* Neutral, because Kotlin DSL build scripts are initially slower to configure/sync than Groovy in
  some IDEs, though this gap has narrowed significantly in recent Gradle versions.
* Bad, because some existing Gradle plugins and online examples still default to Groovy DSL
  snippets, requiring translation.

### Confirmation

CI ([ADR-0008](0008-github-actions-for-continuous-integration.md)) runs
`./gradlew :backend:build` with the pinned Java 25 toolchain and fails if toolchain resolution
falls back to a different JDK.

## Pros and Cons of the Options

### Gradle (Kotlin DSL)

* Good, because it matches the team's stated tooling preference.
* Good, because type-safe, IDE-navigable build scripts reduce a class of build-file bugs.
* Good, because Gradle's build cache and incremental compilation scale well as the single backend
  module grows.
* Bad, because the DSL is a smaller ecosystem of copy-pasteable examples than Groovy DSL.

### Gradle (Groovy DSL)

* Good, because most existing Gradle documentation and third-party plugin examples use it.
* Bad, because it is dynamically typed; build-script errors surface at execution time, not
  edit time.

### Maven

* Good, because its XML, convention-over-configuration model is extremely stable and well
  understood industry-wide.
* Bad, because it is not the team's stated preference.
* Bad, because expressing "one Java module, custom convention plugin for shared config" is more
  natural in Gradle than in Maven's POM inheritance/parent-POM model.

## More Information

* Gradle Kotlin DSL primer: <https://docs.gradle.org/current/userguide/kotlin_dsl.html>
* This ADR covers build tooling only; Spring Modulith's module convention (ADR-0004) is the
  mechanism that actually enforces internal backend boundaries.
