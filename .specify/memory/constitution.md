<!--
Sync Impact Report
==================
Version change: 1.2.0 → 1.3.0
Bump rationale: MINOR — Principle III and Technology and Security Constraints materially
expanded following acceptance of ADR-0013 (API contract) and ADR-0014 (testing strategy);
Analysis and Merge gates extended. Principle III is tightened, not redefined incompatibly, and
no principle or gate is removed.

Modified principles:
  - III. Verified Behaviour (title unchanged):
      - scenario identifier `<feature>-US<story>-AS<scenario>` and the rule that tests carry it;
      - test-first raised from SHOULD to MUST for acceptance-scenario tests; it stays SHOULD
        for other tests, and the old "the pull request MUST say why it was skipped" clause is
        dropped with it;
      - scenario-test assertions change only together with the scenario in spec.md;
      - tests follow the layers and test-double rules of ADR-0014;
      - coverage is reported, never a threshold.
  - I, II, IV, V unchanged.

Modified sections:
  - Technology and Security Constraints: Repository entry names the shared paths `api/` and
    `e2e/`; new API contract entry (ADR-0013); new Testing entry (ADR-0014); the API contract
    is removed from "Not yet decided", and API authentication/authorisation is added to it.
  - Development Workflow and Quality Gates: Analysis gate checks scenario test tasks and their
    order; Merge gate names code generation, architecture fitness tests, the contract checks,
    the end-to-end workflow, and the scenario traceability check.

Added sections: none
Removed sections: none

ADR status: 0001–0014 are all `accepted`; docs/adrs/README.md index matches the front matter.

Dependent artifacts:
  ✅ .specify/templates/overrides/tasks-template.md — new project override: test tasks are
    mandatory, carry the scenario identifier, and come after the API contract change and
    before implementation; paths follow the repository layout fixed here.
  ✅ .specify/templates/overrides/spec-template.md — new project override: acceptance
    scenarios carry the `<feature>-US<story>-AS<scenario>` identifier, and each user story
    states whether it is a critical journey (ADR-0014 end-to-end scope).
  ⚠ .claude/skills/speckit-tasks/SKILL.md — still says "Tests are OPTIONAL" and "tests (if
    requested)". The constitution and the tasks template override take precedence; the skill
    text is an upstream-managed file and is not changed here.
  ⚠ .specify/templates/overrides/plan-template.md — Technical Context still shows generic
    placeholders (Python, FastAPI, pytest, …); plans must fill it from the stack fixed here.
    Not changed by this amendment.
  ⚠ speckit-analyze — does not yet check that a scenario's test task exists and precedes its
    implementation tasks, as the Analysis gate now requires.

Migration: none — no feature spec, plan, or code exists yet.

Follow-up TODOs:
  - Nothing the gates check exists yet: no backend/, frontend/, api/, e2e/,
    .github/workflows/, Dockerfiles, branch protection, docs/architecture/ model, or scenario
    traceability script. The gates bind from the first feature that creates them.
  - ADR-0013 leaves two points to be confirmed by the plan that scaffolds the backend build:
    that OpenAPI Generator's `spring` output works on Spring Boot 4.1, and that Spring
    Modulith treats the generated package outside the application's base package as an
    ordinary dependency. If either fails, ADR-0013 needs a superseding ADR and this
    constitution an amendment.
  - ADR-0013 names candidates but does not choose the OpenAPI validator for backend tests or
    the Angular client generator; the plan that introduces each records the choice.
  - ADR-0006 and ADR-0008 name the CI command `./gradlew :backend:build`, but ADR-0006 makes
    backend/ its own Gradle build (the wrapper lives in backend/, with no `:backend`
    project). Settle the invocation when the build is scaffolded; this constitution does not
    name the command.
  - ADR-0007 still describes the frontend stack as "TBD by its own ADR" in its conventions;
    ADR-0011 decides it.
  - Hosting/runtime target, deployment, API authentication/authorisation, and GHCR retention
    policy are undecided and each need an ADR.
  - Principles IV–V and the security constraints are general defaults, not derived from
    project decisions; confirm or amend before the first feature spec.
-->

# Libertus Constitution

## Core Principles

### I. Specification Before Code

- Every non-trivial feature MUST follow the Spec Kit pipeline before implementation:
  `speckit-specify` → `speckit-clarify` (recommended) → `speckit-plan` → `speckit-tasks` →
  `speckit-analyze` → `speckit-implement`.
- `spec.md` MUST describe *what* and *why* only; it MUST NOT name frameworks, languages,
  libraries, or storage technologies. Technology choices belong in `plan.md`.
- Every user story in `spec.md` MUST have acceptance scenarios concrete enough to be tested.
- Exempt from the pipeline: typo and documentation fixes, dependency bumps without behaviour
  change, and bug fixes that restore already-specified behaviour. Anything that adds or changes
  user-visible behaviour is not exempt.

Rationale: the project is built largely with AI coding agents; a reviewed, written spec keeps
intent from being lost with the chat session and gives reviewers something to check code against.

### II. Recorded Architecture Decisions and Model

- Every architecturally significant decision MUST be recorded as an ADR in `docs/adrs/` using
  the MADR format and conventions of ADR-0001. Architecturally significant means: a new
  runtime dependency, a new service or deployable component, a change of data store, protocol,
  or external integration, or a new cross-cutting pattern.
- A `plan.md` that makes such a decision MUST link the ADR rather than carry the rationale only
  inside the plan.
- Accepted ADRs MUST NOT be rewritten in substance; a changed decision gets a new ADR that
  supersedes the old one.

- The LikeC4 model in `docs/architecture/` (ADR-0003) is the single source of truth for system
  structure: actors, systems, containers, components, external systems, and the relationships
  between them. It MUST describe the system as built.
- A structural change — an added, changed, or removed actor, system, container, component,
  external system, or relationship, or a changed technology on one of them — MUST update the
  model in the same pull request as the code.
- Every `plan.md` MUST contain an Architecture Impact section that lists its structural changes
  by fully qualified model element name, or states "None" with a reason.
- The model MUST pass `likec4 format --check` and `likec4 validate`.
- Model elements that an ADR is about SHOULD link to that ADR.
- Architecture diagrams MUST be generated from the model, not drawn by hand.

Rationale: decision rationale that is not written down gets lost or relitigated, and a
structure that is only implied by code and scattered plans drifts and has to be rediscovered
by every reader and agent. ADRs record why; the model records what.

### III. Verified Behaviour

- Every acceptance scenario in `spec.md` MUST be covered by at least one automated test before
  the feature is merged. A scenario is identified as `<feature>-US<story>-AS<scenario>` (for
  example `003-US1-AS2`), and a test that covers it MUST carry that identifier. Scenarios of a
  merged feature MUST NOT be renumbered; a new scenario is appended to its story.
- The test for an acceptance scenario MUST be written before the implementation and observed
  to fail first. Other tests for new behaviour SHOULD be written before or together with the
  code they cover.
- The assertions of an existing scenario test MUST NOT be changed unless the scenario in
  `spec.md` is changed in the same pull request.
- A bug fix MUST include a test that fails without the fix.
- Tests MUST follow the layers and rules of ADR-0014: a behaviour is tested at the lowest
  layer that can prove it; a test double replaces only another module's public API, an
  external system, or a source of non-determinism; a test that touches persistence runs
  against PostgreSQL with the schema created by the Flyway migrations.
- Test coverage MUST be reported on the pull request and MUST NOT be used as a pass/fail
  threshold.
- The full automated test suite MUST pass on the branch before merge.

Rationale: agent-generated code needs objective evidence that it does what the spec says. A
test written after the code tends to assert what the code does instead of what the spec says,
and a coverage threshold invites tests that assert nothing.

### IV. Simplicity and Justified Complexity

- Implement only what the current spec requires; speculative abstractions, extension points,
  and configuration for hypothetical needs MUST NOT be added.
- Prefer the standard library and existing project dependencies over new ones.
- Any deviation from these rules MUST be justified in the plan's Complexity Tracking table,
  naming the simpler alternative and why it was rejected.

Rationale: every component and dependency is a long-term maintenance cost; complexity must pay
for itself.

### V. Repository as Single Source of Truth

- Principles, specs, plans, tasks, and ADRs MUST live in the repository as Markdown, and the
  architecture model as LikeC4 `.c4` files, versioned with the code.
- A change that alters specified behaviour MUST update the affected `spec.md` (and `plan.md` if
  the design changed) in the same pull request.
- Decisions made in chat, issues, or meetings are not binding until recorded in the repository.

Rationale: artifacts that drift from the code mislead both humans and agents.

## Technology and Security Constraints

The application stack is fixed by accepted ADRs. The ADR holds the full conventions and
rationale; the rules below are the binding summary. Changing any of them requires a superseding
ADR (Principle II) and an amendment of this section.

- **Repository** (ADR-0007): one GitHub monorepo with one top-level directory per deployable —
  `backend/` and `frontend/`. Each deployable MUST be independently buildable, testable,
  versioned, and released; release tags are prefixed by deployable (`backend-v1.2.0`). The
  shared top-level paths are `docs/`, `.specify/`, `.github/`, `api/` (ADR-0013), and `e2e/`
  (ADR-0014).
- **Architecture** (ADR-0004): the backend is one deployable Spring Boot application structured
  as a modular monolith. Each direct subpackage of the main application package is one Spring
  Modulith application module. A module MUST be used by other modules only through the types
  in its root package or through application events, and MUST declare its allowed dependencies.
  A test asserting `ApplicationModules.of(Application.class).verify()` MUST exist and run in CI.
- **Platform** (ADR-0005): Java 25, Spring Boot 4.1.x, Spring Modulith 2.1.x. Spring Boot and
  Spring Modulith minor versions MUST be upgraded together.
- **Build** (ADR-0006): Gradle with Kotlin DSL. The backend is one Gradle module under
  `backend/` with the Java toolchain pinned to 25; a Gradle subproject per business module MUST
  NOT be introduced.
- **Data store** (ADR-0009): PostgreSQL is the default. A non-relational store MAY be
  introduced only through its own ADR that meets all four criteria of ADR-0009.
- **Data access** (ADR-0010): Spring Data JPA is the default. Spring JDBC MAY be used for a
  specific query or operation when one of the four ADR-0010 criteria applies, and the pull
  request MUST state which. No third data-access technology is used. Flyway is the single
  source of truth for the schema; Hibernate `ddl-auto` MUST be `validate` (or `none` for local
  development) and MUST NOT create or alter the schema.
- **Frontend** (ADR-0011): Angular with TypeScript and standalone components, built with the
  Angular CLI, with npm as the package manager.
- **API contract** (ADR-0013): the contract between frontend and backend is the hand-written
  OpenAPI 3.1 document `api/openapi.yaml`. A pull request that adds or changes an endpoint MUST
  change the contract in the same pull request. All API paths are under `/api/v1`; every
  operation has a unique `operationId` and exactly one tag naming the module that owns it;
  error responses use RFC 9457 Problem Details. The backend API interfaces and model classes,
  and the frontend types and client, MUST be generated from the contract at build time and
  MUST NOT be committed or edited by hand. Controllers MUST implement the generated interfaces
  and MUST NOT declare request mappings of their own; generated model classes MUST NOT be used
  outside controllers. Frontend code MUST call the backend only through the generated client.
  A breaking change to the contract MUST fail CI unless the pull request carries the
  `breaking-api-change` label and updates the frontend in the same pull request.
- **Testing** (ADR-0014): backend unit tests start no Spring context and perform no I/O.
  Backend integration tests start one application module with `@ApplicationModuleTest` against
  PostgreSQL in Testcontainers; an in-memory database or a mocked repository MUST NOT be used.
  Every operation in the contract MUST have API tests, for its success response and for each
  error response it declares, that validate requests and responses against `api/openapi.yaml`.
  Whole-application tests are limited to the architecture fitness tests: Spring Modulith
  verification, the comparison of the contract with the registered request mappings, a full
  application context start, and the ArchUnit rules of ADR-0014. Frontend tests replace only
  the HTTP boundary. End-to-end tests use Playwright, live in `e2e/`, run against the container
  images built from the pull request's commit, and cover only journeys that a spec marks as
  critical. The end-to-end workflow and the scenario traceability check MUST be required
  status checks on `main`.
- **Continuous integration** (ADR-0008): GitHub Actions, with one path-filtered CI workflow per
  deployable. These workflows MUST be required status checks on `main`.
- **Delivery** (ADR-0012): every deployable MUST be delivered as an OCI container image
  published to GitHub Container Registry, built from the `Dockerfile` in its own directory. The
  image is the only release artifact. Images MUST be pushed only by GitHub Actions
  authenticating with `GITHUB_TOKEN`; commit-SHA and version tags MUST NOT be overwritten, and
  a `latest` tag MUST NOT be published.
- **Architecture tooling** (ADR-0003): LikeC4 (Node.js >= 22) is a development-time tool for
  the architecture model only; it is not a runtime dependency.
- **Not yet decided**: the hosting/runtime target and deployment, authentication and
  authorisation of the API, and any technology not listed above. Each MUST be introduced
  through an ADR before code depending on it is merged.
- Secrets, credentials, and personal data MUST NOT be committed to the repository, including
  in specs, fixtures, and example files.
- New third-party dependencies MUST be actively maintained and have a licence compatible with
  the project; the check is noted in the plan or ADR that introduces them.

## Development Workflow and Quality Gates

- Each feature is developed on its own branch with a sequentially numbered spec directory
  under `specs/`.
- Gates, in order:
  1. **Spec gate** — `spec.md` has no unresolved `[NEEDS CLARIFICATION]` markers.
  2. **Plan gate** — the plan's Constitution Check passes for every principle above, or each
     violation is justified in Complexity Tracking; required ADRs exist; the plan's Technical
     Context conforms to Technology and Security Constraints, or links the ADR that grants the
     exception or supersedes the constraint; the Architecture Impact section is filled in and
     its model changes are applied and valid (the `speckit-architecture` hook runs after
     `speckit-plan`).
  3. **Analysis gate** — `speckit-analyze` reports no critical issues before `speckit-implement`;
     a structural change in `plan.md` that is missing from the model is an issue, and so is an
     acceptance scenario whose test task is missing from `tasks.md` or does not precede its
     implementation tasks.
  4. **Merge gate** — the pull request links `spec.md`, `plan.md`, and `tasks.md`; the CI
     workflow of every affected deployable passes — build including code generation from the
     API contract, full test suite, Spring Modulith verification and the other architecture
     fitness tests, Flyway migrations applied to a test database, and the container image
     build; when `api/` changes, the contract lint and the breaking-change comparison with
     `main` pass; the end-to-end workflow passes; the scenario traceability check passes;
     the architecture model passes validation and the `speckit-architecture` verification
     after `speckit-implement` reports no unresolved drift; a reviewer confirms the
     implementation matches the spec.
- Pipeline-exempt changes (Principle I) still require passing tests and review.

## Governance

- This constitution supersedes all other development practices in this repository. Where a
  template, command, or document conflicts with it, the constitution wins and the conflicting
  artifact is corrected.
- Amendments are made with `speckit-constitution` through a pull request that states the
  change and its rationale, and are merged only after review by a project decision-maker.
  Amendments that change a principle or gate MUST describe any migration needed for work in
  progress.
- An ADR that supersedes a decision summarised in Technology and Security Constraints MUST be
  accompanied by an amendment of that section in the same pull request.
- Versioning follows semantic versioning:
  - MAJOR — a principle or gate is removed or redefined incompatibly.
  - MINOR — a principle or section is added or materially expanded.
  - PATCH — clarifications and wording fixes with no change in meaning.
- Compliance is checked at the Plan gate (Constitution Check in every `plan.md`), by
  `speckit-analyze`, by architecture model validation, by the required CI workflows, and during
  pull-request review. Reviewers MUST reject changes that violate a principle or constraint
  without a recorded justification.

**Version**: 1.3.0 | **Ratified**: 2026-09-30 | **Last Amended**: 2026-10-01
