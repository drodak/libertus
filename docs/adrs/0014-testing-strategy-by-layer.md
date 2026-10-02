---
status: accepted
date: 2026-10-01
decision-makers: drodak
---

# Testing Strategy: Module-Centred Test Layers with Real Infrastructure

## Context and Problem Statement

Principle III of the constitution requires that every acceptance scenario is covered by an
automated test and that the full suite passes before merge. It does not say what kinds of tests
exist, what each kind is allowed to start or replace with a test double, or which kind a given
behaviour belongs in.

Most code in this project is written by AI coding agents. Left without rules, the same
behaviour gets tested with a mock-heavy class test in one module and a full application test in
another, persistence is tested against whatever database is convenient, and coverage is raised
with tests that assert nothing.

Which test layers does the project have, what are the rules of each, and how is the strategy
enforced?

## Decision Drivers

* Tests are the objective evidence that agent-written code does what the spec says
  (Principle III). They must fail when behaviour breaks and stay green when code is refactored.
* The backend is a modular monolith ([ADR-0004](0004-modular-monolith-with-spring-modulith.md)).
  The test layers should follow the module boundaries the architecture already enforces.
* PostgreSQL-specific behaviour and Flyway migrations
  ([ADR-0009](0009-postgresql-as-default-data-store.md),
  [ADR-0010](0010-jpa-with-spring-jdbc-fallback-for-data-access.md)) must be tested against
  what runs in production.
* The frontend and backend meet at a written contract
  ([ADR-0013](0013-openapi-spec-first-api-contract.md)) that both sides must be tested against.
* The suite runs on every pull request ([ADR-0008](0008-github-actions-for-continuous-integration.md))
  and must stay fast enough for that.
* `spec.md` already records every acceptance scenario as Given/When/Then before any code is
  written (Principle I). The tests should be derived from those scenarios without restating
  them in a second place (Principle V).
* Few tools, and no rule that cannot be checked (Principle IV).

## Considered Options

* Module-centred layers: sociable unit tests, module integration tests on real PostgreSQL, API
  tests validated against the contract, and a thin end-to-end layer
* Executable Gherkin specifications (Cucumber) as the acceptance layer
* Strict test-driven development for all code
* Classic test pyramid with solitary, mock-based unit tests and an in-memory database
* Whole-application tests as the main layer
* No recorded strategy beyond Principle III

## Decision Outcome

Chosen option: "Module-centred layers", because it tests each behaviour at the boundary where
the architecture already draws a line — the module, the API contract, the deployed system —
so tests survive refactoring inside those lines, and it uses the real database everywhere a
database is involved.

### Principles

1. **Lowest layer that proves it.** A behaviour is tested in the lowest layer below that can
   prove it. A higher layer does not repeat what a lower layer already proves.
2. **Behaviour through public entry points.** A test drives the code through an entry point
   that a real caller uses — a module's root-package API, an application event, an HTTP
   endpoint, the rendered page — and asserts on what the caller can observe. Tests do not
   assert on private state or on which internal methods were called.
3. **Doubles only at boundaries.** A test double may replace only: the public API of another
   module, an external system, and sources of non-determinism (clock, randomness, ID
   generation). Collaborators inside the module under test are real.
4. **The database is never faked.** Any test that touches persistence runs against PostgreSQL
   in a container, with the schema created by the Flyway migrations. No in-memory database and
   no mocked repository.
5. **The module is the unit of integration.** Backend integration tests start one application
   module, not the whole application. Whole-application tests are limited to the fitness tests
   listed below.
6. **The contract is checked from both sides.** Backend responses are validated against
   `api/openapi.yaml`; the frontend compiles and stubs HTTP against types generated from it.
7. **Scenarios first, and every scenario traceable to a test.** Behaviour is agreed as
   Given/When/Then acceptance scenarios in `spec.md` before code is written, and tests are
   derived from them. `spec.md` is the only place a scenario is written down; the test carries
   the scenario's identifier and follows its given, when, and then (see Scenario tests).
8. **Tests are deterministic and independent.** No dependence on execution order, wall-clock
   time, sleeps, shared mutable data, or the network outside the test's own containers. A flaky
   test is fixed or deleted; it is not retried until green.
9. **Coverage is reported, not gated.** Coverage numbers inform review. They are never a
   pass/fail threshold.
10. **Scenario tests come before the code.** The test for an acceptance scenario is written
    from `spec.md` before the implementation and seen to fail. A test written after the code
    tends to assert what the code does instead of what the spec says (see Order of work).

### Layers

| Layer | Proves | Starts | Doubles allowed |
|-------|--------|--------|-----------------|
| Backend unit | Domain rules and logic of one module | Plain JUnit, no Spring context, no I/O | Other modules' API, external systems, clock |
| Backend module integration | A module works with its persistence, transactions, events, and Spring wiring | One module via `@ApplicationModuleTest`, PostgreSQL via Testcontainers | Other modules' API, external systems, clock |
| Backend API | A module's HTTP endpoints behave as specified and conform to the contract | As module integration, driven over HTTP | Same as module integration |
| Architecture fitness | Structural rules hold | Static analysis of compiled classes; one full application context | None |
| Frontend unit and component | Component, service, and pure-function behaviour | Angular `TestBed` and the Angular CLI test runner | The HTTP boundary only |
| End-to-end | Critical journeys work across the deployed frontend and backend | Both container images and PostgreSQL | None |

**Backend unit tests**

* Exercise domain logic through the classes that own it, with real collaborators from the same
  module.
* Start no Spring context and touch no database, file system, or network. They run without
  Docker.
* Code that only delegates (a controller passing a request to a service, a repository
  interface) gets no unit test; it is covered by the module integration and API layers.

**Backend module integration tests**

* Start the module under test with Spring Modulith's `@ApplicationModuleTest`.
* Use one PostgreSQL container per test run, started by Testcontainers and migrated by Flyway.
  Each test creates the data it needs and leaves no data behind for the next test.
* Test persistence here: repositories, custom queries, and every Spring JDBC operation allowed
  by ADR-0010.
* Test interaction between modules through events: the test asserts on the events the module
  publishes and publishes the events it consumes, using Spring Modulith's event testing
  support. Another module's synchronous API is replaced by a double of its root-package type.

**Backend API tests**

* Drive a module's endpoints over HTTP, through request mapping, validation, serialisation, and
  error handling.
* Validate every request and response against `api/openapi.yaml` (ADR-0013).
* Every operation in the contract has at least one API test for its success response and one
  for each error response the contract declares.
* Assert on status, body, and observable side effects (stored state, published events), not
  only on schema conformance.

**Architecture fitness tests**

* `ApplicationModules.of(Application.class).verify()` (ADR-0004).
* The contract-to-request-mapping comparison of ADR-0013.
* One test that starts the full application context against PostgreSQL with all migrations
  applied.
* ArchUnit rules for structure inside a module. ArchUnit is already on the test classpath
  through Spring Modulith. The initial rules are:
  * no package cycles inside a module;
  * controllers do not access repositories or the `EntityManager` directly;
  * the model classes generated from the contract (ADR-0013) are referenced only from
    controllers;
  * production code reads the current time only from an injected `java.time.Clock`.
* A new ArchUnit rule is added only together with the ADR or plan that states the rule.

**Frontend unit and component tests**

* Run with the test runner that the Angular CLI scaffolds.
* Test a component through its rendered DOM and user interaction, not through its class
  members.
* Replace only the HTTP boundary, using Angular's HTTP testing support. Stubbed responses are
  typed with the client types generated from the contract, so a stub that no longer matches the
  contract fails to compile.

**End-to-end tests**

* Written with Playwright, in a top-level `e2e/` directory with its own CI workflow, which runs
  when `backend/`, `frontend/`, `api/`, or `e2e/` changes.
* Run against the container images built from the pull request's commit
  ([ADR-0012](0012-container-images-published-to-ghcr.md)) and a PostgreSQL container.
* Cover only journeys that a spec marks as critical and that cross the frontend and the
  backend: at most one test for the main path of each such journey. Validation rules, error
  cases, and variations belong in lower layers.
* An end-to-end test is never the only test for an acceptance scenario that a lower layer can
  prove.

### Order of work

* Work proceeds outside-in, one acceptance scenario at a time. The scenario test is written
  first, at the lowest layer that can prove the scenario — usually the API or module
  integration layer — and is run and seen to fail for the expected reason: the behaviour is
  missing, not the test is broken.
* The implementation is then written until the scenario test passes. Unit tests for domain
  logic are written along the way, before or together with the code they cover.
* In `tasks.md`, the test task of a scenario comes before the implementation tasks of that
  scenario.
* A failing scenario test is made to pass by changing the code. Its assertions are changed
  only when the scenario in `spec.md` has changed first, in the same pull request.
* A bug fix starts with a test that fails without the fix (Principle III), at the lowest layer
  that reproduces the bug.
* A strict red-green-refactor cycle for every class is not required.

### Conventions

* Backend tests live in `backend/src/test/java`, in the package of the module they test.
  Class-name suffixes identify the layer: `…Test` (unit), `…IntegrationTest` (module
  integration), `…ApiTest` (API), `…ArchitectureTest` (fitness).
* Backend tests that need Docker carry the JUnit tag `integration`, so they can be excluded
  from a local run. CI always runs every test.
* Frontend tests are `*.spec.ts` files next to the code they test.
* Scenario tests:
  * An acceptance scenario is identified by its feature number, user story number, and
    scenario number in `spec.md`, in the form `<feature>-US<story>-AS<scenario>` (for example
    `003-US1-AS2`). Once a feature is merged its scenarios are not renumbered; a new scenario
    is appended to its story.
  * A test that covers a scenario carries that identifier — as a JUnit `@Tag` on the backend,
    in the test title on the frontend and in end-to-end tests. One scenario may be covered by
    several tests, and a test may cover several scenarios.
  * A scenario test is laid out in three parts that mirror the scenario's given, when, and
    then, and is named for the behaviour it proves, not for the class or method it calls.
  * The scenario text is not copied into the test code or into feature files.
  * `tasks.md` lists the test for each scenario.
  * A traceability check in CI reads the scenario identifiers from every `spec.md` under
    `specs/` and the identifiers carried by the tests in `backend/`, `frontend/`, and `e2e/`.
    It fails when a scenario has no test or a test names a scenario that does not exist.
* Coverage: the backend build produces a JaCoCo report and the frontend test run produces a
  coverage report. CI publishes both on the pull request. No build fails on a coverage number.

### Consequences

* Good, because tests are bound to module APIs, events, and the HTTP contract, so code can be
  restructured inside a module without rewriting its tests.
* Good, because persistence is tested against the same database engine and the same migrations
  as production.
* Good, because starting one module instead of the whole application keeps integration tests
  fast and makes an undeclared dependency on another module visible as a failing test.
* Good, because the contract is enforced on both sides without a separate contract-testing
  service.
* Good, because the traceability check turns Principle III from a review task into a failing
  build.
* Neutral, because the check proves that a test claims a scenario, not that the test proves
  it; whether the assertions match the scenario's "then" remains a review question.
* Neutral, because a sociable unit test that fails points at a behaviour, not at one class; the
  cause can take longer to locate than with one test per class.
* Bad, because Docker is required to run the integration, API, and end-to-end tests locally and
  in CI.
* Bad, because the end-to-end workflow builds both images and is the slowest check on a pull
  request.
* Bad, because without a coverage threshold, untested code is caught only by scenario
  traceability and review.

### Confirmation

* The backend CI workflow runs all backend layers; the frontend CI workflow runs the frontend
  tests; the end-to-end workflow runs the Playwright tests. All three are required status
  checks on `main`.
* The architecture fitness tests enforce the structural rules, and the injected-clock rule that
  principle 3 depends on.
* The traceability check is a required status check on `main`: every acceptance scenario in
  every `spec.md` has at least one test carrying its identifier. The reviewer confirms that
  the test's assertions match the scenario.
* `speckit-analyze` checks that in `tasks.md` each scenario's test task precedes its
  implementation tasks.
* Review rejects a pull request that changes the assertions of an existing scenario test
  without a matching change to the scenario in `spec.md`. That a new scenario test was seen to
  fail first cannot be verified from the pull request; the task order and this review rule are
  the checks that stand in for it.
* Review rejects: a mock of a class from the module under test, a mocked repository, an
  in-memory database, a `@SpringBootTest` used where a module test would do, and an end-to-end
  test for behaviour a lower layer can prove.

## Pros and Cons of the Options

### Module-centred layers

* Good, because the layers match boundaries the architecture already enforces.
* Good, because real collaborators and a real database mean a green test says more about
  production behaviour.
* Neutral, because it relies on Spring Modulith's test support, which ties the test suite to
  that library as ADR-0004 ties the production code to it.
* Bad, because integration tests need containers and are slower than mock-based tests.

### Executable Gherkin specifications (Cucumber)

Acceptance scenarios are written as `.feature` files and bound to test code by step
definitions; the other layers stay as in the chosen option.

* Good, because the scenario text itself is executed, so it cannot drift from the test.
* Good, because it produces a pass/fail report per scenario that non-developers can read.
* Bad, because the scenarios would exist twice, in `spec.md` and in feature files, and one of
  them would drift (Principle V). Avoiding that would mean replacing the spec format of
  Spec Kit ([ADR-0002](0002-use-spec-kit-for-spec-driven-development.md)).
* Bad, because step definitions are an extra layer of code to maintain between the scenario
  and the test.
* Bad, because Cucumber scenarios are normally run against the whole application or the
  browser, which works against testing at the lowest layer that proves the behaviour.
* Bad, because its main benefit, scenarios read and written by non-developers, has no audience
  in the project today.

### Strict test-driven development for all code

Every line of production code is written in a red-green-refactor cycle, starting from a failing
unit test; the layers stay as in the chosen option.

* Good, because no code exists without a test that demanded it.
* Good, because it gives the same protection against tests that mirror the implementation as
  the chosen option, at every level and not only for scenarios.
* Bad, because the cycle leaves no trace in a pull request, so the rule cannot be checked
  (Principle IV).
* Bad, because it demands unit tests for code that only delegates, which the chosen option
  deliberately leaves to the module integration and API layers.
* Bad, because a class-by-class cycle pulls towards one test per class, which works against
  testing behaviour through public entry points.

### Classic pyramid with solitary unit tests and an in-memory database

One test class per production class, every collaborator mocked, persistence tested against H2.

* Good, because tests are fast and need no Docker.
* Good, because a failure points at one class.
* Bad, because tests encode how classes call each other; refactoring breaks tests whose
  behaviour did not change, and agents then rewrite the tests to match the code.
* Bad, because H2 does not behave like PostgreSQL and cannot run PostgreSQL-specific migrations
  or the queries ADR-0010 allows Spring JDBC for.
* Bad, because mocked collaborators let a test pass while the real wiring is broken.

### Whole-application tests as the main layer

Most behaviour tested with `@SpringBootTest` or through the browser.

* Good, because every test exercises the real system.
* Bad, because the suite becomes slow as modules are added, since every test starts every
  module.
* Bad, because failures are hard to localise and browser tests are the most likely to be flaky.
* Bad, because it hides undeclared dependencies between modules that a single-module test
  would expose.

### No recorded strategy

* Good, because it costs nothing now.
* Bad, because each feature plan and each agent session chooses its own test style, and the
  suite becomes inconsistent in what a passing test means.

## More Information

* Spring Modulith integration testing: <https://docs.spring.io/spring-modulith/reference/testing.html>
* Testcontainers: <https://testcontainers.com/>
* ArchUnit: <https://www.archunit.org/>
* Playwright: <https://playwright.dev/>
* Test-time dependencies added by this ADR: Testcontainers, JaCoCo, Playwright, and the OpenAPI
  validator of ADR-0013. Their maintenance status, licence, and compatibility with the platform
  of [ADR-0005](0005-java-25-and-spring-boot-as-application-platform.md) are confirmed in the
  plan that introduces each.
* The traceability check is a script in the repository, written with the first feature that
  has tests. It needs no new dependency.
* Executable Gherkin is reconsidered, through its own ADR, if domain experts or a product
  owner start reviewing scenarios and need a readable report per scenario. It would then be
  limited to backend API-level acceptance tests.
* Not covered here, each needing its own decision when it becomes relevant: performance and
  load testing, security testing, accessibility testing, and an automated comparison of the
  LikeC4 model with the code (left open by
  [ADR-0003](0003-use-likec4-for-architecture-as-code.md)).
* Accepting this ADR requires amending the constitution: Principle III gains the layer and
  test-double rules and its test-first rule is raised from SHOULD to MUST for scenario tests
  (it stays SHOULD for unit tests), Technology and Security Constraints gains a testing entry, and the Merge
  gate names the end-to-end workflow and the contract checks.
* `.specify/templates/tasks-template.md` currently calls tests optional, which contradicts
  Principle III and this ADR; it is corrected with that amendment.
