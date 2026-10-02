<!--
Sync Impact Report
==================
Version change: 1.4.0 → 1.5.0
Bump rationale: MINOR — a new Technology and Security Constraints entry (local and
small-environment deployment, ADR-0016) is added, and the Repository entry, "Not yet decided",
and the Merge gate are extended to match. No principle or gate is removed or redefined
incompatibly.

Modified principles: none — I–V unchanged.

Modified sections:
  - Technology and Security Constraints:
    - Repository: `compose.yaml` and `compose.override.yaml` added to the shared top-level paths.
    - Added entry "Local and small-environment deployment" (ADR-0016): Docker Compose for
      one-host environments and as the only way to run the whole system locally; the split
      between `compose.yaml` and `compose.override.yaml`; no credentials in `compose.yaml`;
      published GHCR images outside a developer's machine; `.env` not committed; the same
      PostgreSQL major version as Testcontainers; tests do not use the Compose services; CI
      path filters and `docker compose config` validation.
    - Not yet decided: "the hosting/runtime target and deployment" narrowed to environments
      outside ADR-0016's scope and to automated deployment; TLS termination, database backup
      and restore, and host secret storage for non-local small environments listed explicitly.
  - Development Workflow and Quality Gates: Merge gate CI line gains the validation of the
    Compose files.

Added sections: none (one entry added inside an existing section)
Removed sections: none

ADR status: 0001–0015 are `accepted`. ADR-0016 is still `proposed` — this command writes only
the constitution. Its front matter and the docs/adrs/README.md index row MUST be set to
`accepted` in the same pull request as this amendment (as was done for ADR-0015).

Dependent artifacts:
  ✅ .specify/templates/overrides/plan-template.md — the Constitution Check is filled from this
    file at plan time; no template change needed. Its Technical Context still shows generic
    placeholders (carried over from 1.3.0).
  ✅ .specify/templates/overrides/spec-template.md, tasks-template.md — no change needed.
  ⚠ speckit-plan, speckit-implement — still do not rebase the branch (carried over from 1.4.0).
  ⚠ speckit-specify — no branch-creating hook configured (carried over from 1.4.0).
  ⚠ .claude/skills/speckit-tasks/SKILL.md, speckit-analyze — open points carried over from
    1.3.0 (tests described as optional; scenario test-task order not checked).

Migration: the feature branch `1-create-backend-skeleton` is open but has no spec, plan, or
code yet. Its plan MUST include `compose.yaml`, `compose.override.yaml`, the `.env` entry in
`.gitignore`, the Compose path filters, and the `docker compose config` validation in the
backend CI workflow. Nothing already merged is affected.

Follow-up TODOs:
  - Set ADR-0016 to `accepted` (front matter and index) in this pull request.
  - Carried over from 1.4.0: repository settings and branch protection required by ADR-0015
    do not exist yet (no remote is configured); no agent instruction file (CLAUDE.md or
    AGENTS.md) states that an agent does not merge pull requests.
  - Carried over from 1.3.0, updated: nothing the gates check exists yet (backend/, frontend/,
    api/, e2e/, .github/workflows/, Dockerfiles, compose.yaml, compose.override.yaml,
    docs/architecture/ model, scenario traceability script); the two ADR-0013 points to confirm
    when the backend build is scaffolded; the OpenAPI validator and Angular client generator
    are not chosen; the `./gradlew :backend:build` invocation in ADR-0006/0008; ADR-0007's
    "TBD" frontend stack wording; hosting beyond one host, automated deployment, API
    authentication/authorisation, and GHCR retention are undecided; Principles IV–V and the
    security constraints are general defaults to confirm before the first feature spec.
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
  shared top-level paths are `docs/`, `.specify/`, `.github/`, `api/` (ADR-0013), `e2e/`
  (ADR-0014), and `compose.yaml` and `compose.override.yaml` (ADR-0016).
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
- **Local and small-environment deployment** (ADR-0016): Docker Compose is the deployment
  method for small environments — one host, no high availability, no horizontal scaling, no
  rolling upgrade — and the only supported way to run the whole system on a developer's
  machine. The system is described by `compose.yaml` in the repository root, with one service
  per deployable and one for PostgreSQL; a new deployable, and any external system the
  application needs in order to start, MUST be added to it in the pull request that introduces
  it. `compose.yaml` MUST NOT contain credentials or defaults for them, and a required variable
  has no default. What applies only to a developer's machine — building the images from the
  working tree, development-only values marked as not secret, and published ports — lives in
  `compose.override.yaml`. Every other environment runs from `compose.yaml` alone: it runs
  images published to GHCR, referenced by version tag or digest, builds nothing on the host,
  takes its values and secrets from the operator, and does not publish the database port;
  `.env` MUST NOT be committed. The PostgreSQL service uses the same major version as the
  Testcontainers database, and automated tests MUST NOT use the Compose services. The CI
  workflow of every deployable includes both files in its path filter and validates them with
  `docker compose config`, with and without the override.
- **Architecture tooling** (ADR-0003): LikeC4 (Node.js >= 22) is a development-time tool for
  the architecture model only; it is not a runtime dependency.
- **Not yet decided**: the hosting/runtime target and deployment of environments outside the
  small-environment scope of ADR-0016; automated deployment (continuous delivery) to any
  environment; for small environments other than a developer's machine, TLS termination,
  backup and restore of the database, and storage of secrets on the host; authentication and
  authorisation of the API; and any technology not listed above. Each MUST be introduced
  through an ADR before code depending on it is merged.
- Secrets, credentials, and personal data MUST NOT be committed to the repository, including
  in specs, fixtures, and example files.
- New third-party dependencies MUST be actively maintained and have a licence compatible with
  the project; the check is noted in the plan or ADR that introduces them.

## Development Workflow and Quality Gates

Branching, review, and merging are fixed by ADR-0015. The ADR holds the full conventions and
rationale; the rules below are the binding summary. Changing any of them requires a superseding
ADR (Principle II) and an amendment of this section.

- **Branching**: `main` is the only long-lived branch. Every change MUST reach `main` through a
  pull request from its own branch; direct pushes to `main` MUST be rejected by branch
  protection, administrators included. The age of a branch is not limited. A release is a tag
  on a commit of `main`.
- **Features**: each feature is developed on its own branch with a sequentially numbered spec
  directory under `specs/`. A feature MUST be merged complete, in one pull request that
  contains its `spec.md`, `plan.md`, `tasks.md`, the implementation, and the tests of every
  acceptance scenario. A pull request MUST NOT merge part of a feature. A feature that cannot
  be reviewed in one pull request MUST be split into smaller features, each with its own spec
  and each delivering working behaviour on its own.
- **Rebasing**: a feature branch MUST be rebased onto the current `main` before `speckit-plan`,
  before `speckit-implement`, and before it is merged. `main` MUST NOT be merged into a branch.
  If the rebase before `speckit-implement` brings in changes that affect the plan, `plan.md`
  and `tasks.md` MUST be updated before implementation starts. A feature number or a Flyway
  migration version that `main` has taken in the meantime MUST be renumbered on the branch.
- **Review**: every pull request MUST be reviewed by a person before it is merged. Only a
  person merges a pull request: an AI coding agent MAY create the branch, push to it, and open
  the pull request, and MUST NOT merge it; auto-merge MUST be disabled. A review by an agent
  does not count as the approval. A change pushed after the review MUST be reviewed again,
  except a rebase onto `main` that needed no conflict resolution.
  - While the project has one developer, that developer reviews every pull request, including
    those opened under their own account, and merging is the approval; GitHub's required
    approving review is not switched on.
  - Once a second contributor has write access, the approval MUST be given by a person who
    neither wrote the change nor directed the agent that wrote it, and GitHub's required
    approving review MUST be switched on.
- **Merging**: a pull request MUST be merged by squashing, so that it becomes one commit on top
  of `main`. The repository MUST allow squash merging only, and the history of `main` MUST be
  linear. The branch is deleted when it is merged.
- Gates, in order:
  1. **Spec gate** — `spec.md` has no unresolved `[NEEDS CLARIFICATION]` markers.
  2. **Plan gate** — the plan's Constitution Check passes for every principle above, or each
     violation is justified in Complexity Tracking; required ADRs exist; the plan's Technical
     Context conforms to Technology and Security Constraints, or links the ADR that grants the
     exception or supersedes the constraint; the Architecture Impact section is filled in and
     its model changes are applied and valid (the `speckit-architecture` hook runs after
     `speckit-plan`); the branch was rebased onto the current `main` before `speckit-plan`;
     the feature can be reviewed in one pull request, or it is sent back to be split.
  3. **Analysis gate** — the branch is rebased onto the current `main` before
     `speckit-implement`; `speckit-analyze` reports no critical issues before `speckit-implement`;
     a structural change in `plan.md` that is missing from the model is an issue, and so is an
     acceptance scenario whose test task is missing from `tasks.md` or does not precede its
     implementation tasks.
  4. **Merge gate** — the pull request contains the complete feature and links `spec.md`,
     `plan.md`, and `tasks.md`; the branch is rebased onto the current `main`; the CI
     workflow of every affected deployable passes — build including code generation from the
     API contract, full test suite, Spring Modulith verification and the other architecture
     fitness tests, Flyway migrations applied to a test database, the container image build,
     and the validation of the Compose files; when `api/` changes, the contract lint and the breaking-change comparison with
     `main` pass; the end-to-end workflow passes; the scenario traceability check passes;
     the architecture model passes validation and the `speckit-architecture` verification
     after `speckit-implement` reports no unresolved drift; the reviewer (see Review) confirms
     that the implementation matches the spec and that no principle or constraint is violated
     without a recorded justification; a person merges the pull request by squashing.
- Pipeline-exempt changes (Principle I) follow the same branching, review, and merging rules:
  they are rebased onto the current `main` before merge and require passing tests.

## Governance

- This constitution supersedes all other development practices in this repository. Where a
  template, command, or document conflicts with it, the constitution wins and the conflicting
  artifact is corrected.
- Amendments are made with `speckit-constitution` through a pull request that states the
  change and its rationale, and are merged only after review by a project decision-maker.
  Amendments that change a principle or gate MUST describe any migration needed for work in
  progress.
- An ADR that supersedes a decision summarised in Technology and Security Constraints or in
  Development Workflow and Quality Gates MUST be accompanied by an amendment of that section
  in the same pull request.
- Versioning follows semantic versioning:
  - MAJOR — a principle or gate is removed or redefined incompatibly.
  - MINOR — a principle or section is added or materially expanded.
  - PATCH — clarifications and wording fixes with no change in meaning.
- Compliance is checked at the Plan gate (Constitution Check in every `plan.md`), by
  `speckit-analyze`, by architecture model validation, by the required CI workflows, by branch
  protection and the repository's merge settings, and during pull-request review. Reviewers
  MUST reject changes that violate a principle or constraint without a recorded justification.

**Version**: 1.5.0 | **Ratified**: 2026-09-30 | **Last Amended**: 2026-10-03
