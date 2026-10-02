---
status: accepted
date: 2026-10-01
decision-makers: drodak
---

# OpenAPI Spec-First Contract for the Frontend–Backend API

## Context and Problem Statement

The Angular frontend ([ADR-0011](0011-angular-as-the-frontend-framework.md)) consumes the REST
API of the Spring Boot backend ([ADR-0004](0004-modular-monolith-with-spring-modulith.md)). The
two are separate deployables, versioned and released independently
([ADR-0007](0007-github-monorepo-for-backend-and-frontend.md)). ADR-0011 left the API contract
out of scope, and the constitution lists it as not yet decided.

Without a written contract, the shape of the API exists only in backend controller code and in
frontend code that guesses at it. Nothing fails when the two drift apart, and there is nothing
for an API test to check a response against
([ADR-0014](0014-testing-strategy-by-layer.md)).

Where is the API contract defined, which side follows which, and how is conformance checked?

## Decision Drivers

* The contract must be reviewable before the implementation exists, in line with Principle I
  (Specification Before Code).
* Both sides must be checked against the same artifact automatically; a contract that is only
  documentation drifts.
* Drift should be caught as early as possible — at compile time where it can be, in tests where
  it cannot. Most code is written by AI coding agents, and a compile error constrains an agent
  more reliably than a convention.
* A backend release must not silently break an already released frontend.
* There is one consumer, in the same repository. The mechanism should not need extra
  infrastructure.
* It must not weaken the module boundaries of ADR-0004.

## Considered Options

* OpenAPI spec-first, with backend interfaces and the frontend client generated from the
  contract
* OpenAPI spec-first, with hand-written backend controllers checked against the contract only
  by tests
* OpenAPI code-first: the document is generated from the controllers
* Consumer-driven contracts (Pact)
* No formal contract

## Decision Outcome

Chosen option: "OpenAPI spec-first, with backend interfaces and the frontend client generated
from the contract", because it is the only option in which the contract can be designed and
reviewed in the pull request before any code, and in which both sides then fail to compile when
they disagree with it on paths, parameters, or payload types. What the compiler cannot see is
covered by validating real responses against the same file.

Conventions:

* The contract is one OpenAPI 3.1 document at `api/openapi.yaml`. `api/` is a shared top-level
  path in the sense of ADR-0007: both the backend and the frontend CI workflow include `api/**`
  in their path filters.
* The contract is written by hand and changed first. A pull request that adds or changes an
  endpoint changes `api/openapi.yaml` in the same pull request.
* All API paths are under `/api/v1`. Every operation has a unique `operationId` and exactly one
  tag, naming the backend module that owns it.
* Error responses use one shared schema, RFC 9457 Problem Details
  (`application/problem+json`).
* **Backend**: the API interfaces and the request and response model classes are generated
  from `api/openapi.yaml` during the backend build.
  * Generation uses the OpenAPI Generator Gradle plugin with the `spring` generator in
    interface-only mode: it produces one Java interface per tag, carrying the Spring Web
    mapping annotations, and the model classes. It produces no controllers and no
    implementation.
  * The interfaces are generated without default method bodies, so a controller that
    implements an interface must implement every operation of it to compile.
  * The generated code is written to the build directory, compiled with the main source set,
    not committed, and not edited by hand.
  * The generated code lives in a package outside the application's base package (for example
    `org.libertus.api`), so it is not an application module of ADR-0004. Every module may
    depend on it; it depends on no module.
  * Each module's controllers are written by hand inside that module and implement the
    generated interface for the module's tag. A controller does not declare request mappings
    of its own.
  * Generated model classes are the HTTP representation only. They are used in controllers and
    are not passed into a module's domain logic, persisted, or published in application events.
* **Backend conformance beyond the compiler** is checked by tests:
  * every API test validates the request it sends and the response it receives against
    `api/openapi.yaml` (see ADR-0014). This covers status codes, error bodies, and
    serialisation, which the generated interfaces do not constrain;
  * one test compares the operations in the contract with the request mappings the application
    registers under `/api`, in both directions. A generated interface that no controller
    implements, or an endpoint declared outside the generated interfaces, fails the build.
* **Frontend**: the TypeScript API types and the Angular client services are generated from
  `api/openapi.yaml` during the frontend build. The generated code is not committed and not
  edited by hand. Frontend code calls the backend only through the generated client.
* **Contract checks in CI**, run by both workflows when `api/**` changes:
  * the document is linted with Spectral using its built-in OpenAPI ruleset;
  * the document is compared with the version on `main` using oasdiff. A breaking change fails
    the check.
* A deliberate breaking change is allowed only when the pull request carries the
  `breaking-api-change` label and updates the frontend in the same pull request. Once more than
  one released frontend version has to be supported, a breaking change goes under a new path
  prefix (`/api/v2`) instead; that point is marked by a new ADR.

### Consequences

* Good, because the API is reviewed as a design artifact before it is implemented.
* Good, because a backend controller and the frontend both fail to compile when they disagree
  with the contract on a path, a parameter, or a payload type.
* Good, because request and response types are not written twice; the contract is their only
  definition.
* Good, because breaking changes are detected by a tool rather than by a reviewer reading a
  diff.
* Good, because no broker or other service is needed; everything is files in the repository.
* Neutral, because modules share one generated package of interfaces and models. It holds no
  logic and mirrors a contract that is already one shared document, but a change to a shared
  schema recompiles every module that uses it.
* Bad, because the generated model classes follow the generator's conventions (mutable
  classes with its annotations) and cannot be shaped like hand-written records. Mapping between
  them and a module's own types is extra code in each controller.
* Bad, because the backend build depends on the generator keeping pace with the platform of
  [ADR-0005](0005-java-25-and-spring-boot-as-application-platform.md); a Spring Boot upgrade
  may have to wait for a generator release.
* Bad, because the generated interfaces do not constrain run-time behaviour, so response
  validation in tests is still needed and only covers the operations and status codes a test
  exercises. ADR-0014 requires an API test per operation to close this gap.
* Bad, because it adds development-time tools: OpenAPI Generator, an OpenAPI validator for
  tests, a frontend client generator, Spectral, and oasdiff.

### Confirmation

* The backend build generates the interfaces and models from the contract and fails when a
  controller does not match them.
* The backend CI workflow fails when a response does not match the contract or when the
  contract and the registered request mappings differ.
* The Spring Modulith verification of ADR-0004 passes with the generated package in place, and
  an architecture fitness test (ADR-0014) asserts that generated model classes are referenced
  only from controllers.
* The frontend CI workflow generates the client from the contract and fails on a type error.
* Both workflows run the Spectral lint and the oasdiff comparison with `main`.
* Review rejects a pull request that commits or edits generated code, declares a request
  mapping outside a generated interface, or calls the backend without the generated client.

## Pros and Cons of the Options

### OpenAPI spec-first, generated backend interfaces and frontend client

* Good, because the contract exists before the code and can be reviewed on its own.
* Good, because one file drives backend interfaces, frontend types, response validation, and
  breaking-change checks.
* Good, because disagreement with the contract on shape is a compile error on both sides.
* Neutral, because someone has to write OpenAPI by hand.
* Bad, because the backend build depends on a code generator and its support for the current
  Spring Boot version.
* Bad, because generated models are less idiomatic than hand-written types.

### OpenAPI spec-first, hand-written controllers checked by tests

Controllers and their request and response types are written by hand; API tests validate
responses against the contract.

* Good, because the backend build needs no generator and the request and response types can be
  records shaped by the team.
* Good, because nothing is shared between modules.
* Bad, because a controller that disagrees with the contract is found only if an API test
  exercises that operation and that field.
* Bad, because the backend types repeat what the contract already says.
* Bad, because an agent writing a controller from scratch has nothing but a convention telling
  it to follow the contract.

### OpenAPI code-first

The OpenAPI document is generated from the controllers (springdoc) and a snapshot is committed.

* Good, because there is no duplication on the backend; the document always matches the code.
* Good, because there is less to write up front.
* Bad, because the contract is a by-product of the implementation: it cannot be reviewed before
  the code, and the frontend cannot be built against an endpoint that is not implemented yet.
* Bad, because the quality of the document depends on annotations added to the controllers.

### Consumer-driven contracts (Pact)

* Good, because the backend is verified against what the consumer actually uses.
* Bad, because it needs a pact broker or a file exchange between the builds.
* Bad, because its main benefit, coordinating many consumers that release separately, does not
  apply to one consumer in the same repository.
* Bad, because it does not produce a typed client; that would still need an OpenAPI document.

### No formal contract

* Good, because it costs nothing to set up.
* Bad, because drift between frontend and backend is found only at run time or in end-to-end
  tests.
* Bad, because the API tests have no independent definition of a correct response.

## More Information

* OpenAPI Generator: <https://openapi-generator.tech/>. Its `spring` generator has a
  `useSpringBoot4` option, which also switches the generated code to Jackson 3; it is reported
  to be available from version 7.16.0
  (<https://docs.openrewrite.org/recipes/java/spring/boot4/migrateopenapigeneratortospringboot4>).
  The plan that scaffolds the backend build confirms the version and that the generated code
  compiles and runs on the platform of ADR-0005.
* To confirm in the same plan: that Spring Modulith treats a package outside the application's
  base package as an ordinary dependency and not as a module. If it does not, the generated
  package is declared as an open, shared module instead, and that change is recorded in this
  ADR if it is still proposed, or in a superseding ADR if it is accepted.
* Other tools to add when the first endpoint is built. Each is a build-time or test-time
  dependency, and its maintenance status, licence, and compatibility with ADR-0005 are
  confirmed in the plan that introduces it:
  * OpenAPI request and response validation in backend tests: `swagger-request-validator` is
    the candidate.
  * Angular client generation: OpenAPI Generator (`typescript-angular`), which reuses the
    backend's tool but needs a JVM in the frontend build, or `ng-openapi-gen`, which needs only
    Node.js.
  * Spectral: <https://github.com/stoplightio/spectral>
  * oasdiff: <https://github.com/oasdiff/oasdiff>
* The contract may be split into several files joined by `$ref` under `api/` when it grows;
  `api/openapi.yaml` stays the entry point.
* Authentication and authorisation of the API are not decided here.
* Accepting this ADR requires amending the constitution: the API contract is removed from "Not
  yet decided" and summarised under Technology and Security Constraints.
