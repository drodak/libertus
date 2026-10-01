---
status: accepted
date: 2026-10-01
decision-makers: drodak
---

# Modular Monolith as the Architectural Style, Enforced with Spring Modulith

## Context and Problem Statement

Libertus's backend needs an architectural style that limits complexity without the operational
overhead of distributed systems. A plain monolith with no internal structure tends to erode into
an unstructured "big ball of mud" as it grows — modules reach into each other's internals and
every change risks unrelated breakage. Microservices avoid that erosion by forcing boundaries at
the network layer, but add deployment, versioning, and consistency overhead that is not justified
for a team of this size at this stage.

How do we get enforced internal boundaries — so the codebase stays navigable as it grows — while
keeping a single deployable unit?

## Decision Drivers

* Limit operational complexity: one process, one deployment, one database connection pool to
  reason about (ties into [ADR-0009](0009-postgresql-as-default-data-store.md)).
* Boundaries between business modules must be enforced, not just suggested by folder names, or
  they decay as the codebase grows and more agents/contributors touch it.
* Team has existing Spring Boot experience; the enforcement mechanism should build on that
  rather than introduce an unrelated paradigm.
* Should not foreclose extracting a module into its own deployable later if it genuinely earns
  that (the repo already anticipates more than one deployable — see
  [ADR-0007](0007-github-monorepo-for-backend-and-frontend.md)).
* Low ceremony: the mechanism should be enforceable in CI without a bespoke rule engine.

## Considered Options

* Modular monolith enforced by Spring Modulith
* Modular monolith enforced by Gradle multi-module subprojects + ArchUnit
* Unstructured monolith (package-by-layer or ad hoc, no enforcement)
* Microservices

## Decision Outcome

Chosen option: "Modular monolith enforced by Spring Modulith", because it gives explicit,
automatically verified module boundaries without requiring a distributed system, and it is a
Spring-native tool that fits the team's existing Spring Boot experience
([ADR-0005](0005-java-25-and-spring-boot-as-application-platform.md)) directly.

Conventions:

* The backend is one deployable Spring Boot application. Each direct subpackage of the main
  application package (e.g. `org.libertus.backend.billing`, `org.libertus.backend.catalog`) is
  one Spring Modulith application module.
* A module's public API is whatever it exposes from its root package; types in nested
  sub-packages (e.g. `billing.internal`) are not accessible from other modules. Cross-module
  reads of another module's internals are a boundary violation, not a style preference.
* Modules that must react to something happening in another module do so via Spring application
  events (`ApplicationModuleListener`), not direct calls into that module's internals, unless the
  interaction is a synchronous query against the other module's declared public API.
* A module declares the modules it is allowed to depend on via
  `@ApplicationModule(allowedDependencies = …)`; an undeclared dependency fails verification.
* Because Spring Modulith's module boundary is package-based within one artifact, the backend
  does **not** need separate Gradle subprojects per business module — see
  [ADR-0006](0006-gradle-as-the-build-tool.md) for the build layout this implies.

### Consequences

* Good, because module boundaries are checked by a test, not by convention or code review
  vigilance alone.
* Good, because event-driven inter-module communication keeps modules independently testable and
  mirrors how a module would communicate if later extracted into its own service.
* Good, because it generates module documentation (C4-style diagrams, module canvases) that can
  cross-check against the LikeC4 model from ADR-0003.
* Neutral, because it is an opinionated convention (package layout = module boundary); teams used
  to layered-by-technical-concern packages (`controller`/`service`/`repository`) need to re-learn
  package-by-feature.
* Bad, because it is a smaller, younger project than Spring Framework core, with a narrower pool
  of existing team experience to draw on.
* Bad, because moving to event-driven module communication where a direct call previously existed
  is a non-trivial refactor if done after the fact — it is cheapest to apply from the first
  module onward.

### Confirmation

A test asserting `ApplicationModules.of(Application.class).verify()` passes MUST exist and run in
CI ([ADR-0008](0008-github-actions-for-continuous-integration.md)). A pull request that adds an
illegal cross-module dependency fails this test before it fails code review.

## Pros and Cons of the Options

### Spring Modulith

<https://spring.io/projects/spring-modulith>

* Good, because boundary verification, event-driven decoupling, and documentation generation come
  as one coherent, Spring-native toolset.
* Good, because it requires no change to the Gradle project layout — one artifact, package-based
  modules.
* Neutral, because it imposes its own module-per-top-level-package convention; teams must adopt it
  as-is rather than designing their own.
* Bad, because it is a younger Spring project (first GA in 2023) with a smaller community than
  core Spring.

### Gradle multi-module + ArchUnit

* Good, because Gradle subproject boundaries are enforced by the compiler itself (no access to
  another subproject's code unless it's an explicit dependency), which is stronger than a
  package-visibility convention.
* Good, because ArchUnit is well-established and framework-agnostic.
* Bad, because it requires hand-written ArchUnit rules and ongoing maintenance of the Gradle
  module graph, with no built-in event-based decoupling mechanism — that would need to be built
  separately.
* Bad, because every new module means a new Gradle subproject with its own `build.gradle`,
  raising setup ceremony for small modules.

### Unstructured monolith

* Good, because there is zero setup cost.
* Bad, because nothing stops modules from reaching into each other's internals; boundaries exist
  only as long as everyone remembers and agrees to respect them.
* Bad, because this is the exact "spaghetti code" outcome the team explicitly wants to avoid.

### Microservices

* Good, because boundaries are enforced by the network, which is the strongest possible
  enforcement.
* Bad, because it adds distributed-systems concerns (partial failure, eventual consistency,
  per-service deployment and monitoring) that are not justified at the current team size and
  stage.
* Bad, because it directly contradicts the stated goal of limiting complexity.

## More Information

* Spring Modulith reference docs: <https://docs.spring.io/spring-modulith/reference/>
* If a module is later extracted into its own deployable, that is a new ADR superseding the
  relevant part of this one, not a silent change.
