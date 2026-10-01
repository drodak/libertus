<!--
Sync Impact Report
==================
Version change: 1.1.0 → 1.2.0
Bump rationale: MINOR — Technology and Security Constraints materially expanded (the
application stack is now fixed by accepted ADRs 0004–0012), Plan and Merge gates extended;
no principle removed or redefined incompatibly.

Modified principles: none (I–V unchanged)

Modified sections:
  - Technology and Security Constraints ("no stack fixed yet" replaced by the stack, delivery
    and repository constraints of ADR-0004–0012; open decisions listed)
  - Development Workflow and Quality Gates (Plan gate checks stack conformance; Merge gate
    names the CI checks of ADR-0004, 0008, 0010 and 0012)
  - Governance (a superseding stack ADR requires a constitution amendment)

Added sections: none
Removed sections: none

ADR status: 0001–0012 are all `accepted`; docs/adrs/README.md index matches the front matter.

Dependent artifacts:
  ⚠ .specify/templates/overrides/plan-template.md — Technical Context still shows generic
    placeholders (Python, FastAPI, …); plans must fill it from the stack fixed here.
    Not changed by this amendment.

Migration: none — no feature spec, plan, or code exists yet.

Follow-up TODOs:
  - Nothing the new gates check exists yet: no backend/, frontend/, .github/workflows/,
    Dockerfiles, branch protection, or docs/architecture/ model. The gates bind from the
    first feature that creates them.
  - ADR-0006 and ADR-0008 name the CI command `./gradlew :backend:build`, but ADR-0006 makes
    backend/ its own Gradle build (the wrapper lives in backend/, with no `:backend`
    project). Settle the invocation when the build is scaffolded; this constitution does not
    name the command.
  - ADR-0007 still describes the frontend stack as "TBD by its own ADR" in its conventions;
    ADR-0011 decides it.
  - Hosting/runtime target, deployment, API contract, and GHCR retention policy are undecided
    and each need an ADR.
  - Principles III–V and the security constraints are general defaults, not derived from
    project decisions; confirm or amend before the first feature spec.
-->

# Libertus2 Constitution

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
  the feature is merged.
- Tests for new behaviour SHOULD be written before the implementation and observed to fail
  first; where this is skipped, the pull request MUST say why.
- A bug fix MUST include a test that fails without the fix.
- The full automated test suite MUST pass on the branch before merge.

Rationale: agent-generated code needs objective evidence that it does what the spec says.

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
  versioned, and released; release tags are prefixed by deployable (`backend-v1.2.0`).
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
- **Continuous integration** (ADR-0008): GitHub Actions, with one path-filtered CI workflow per
  deployable. These workflows MUST be required status checks on `main`.
- **Delivery** (ADR-0012): every deployable MUST be delivered as an OCI container image
  published to GitHub Container Registry, built from the `Dockerfile` in its own directory. The
  image is the only release artifact. Images MUST be pushed only by GitHub Actions
  authenticating with `GITHUB_TOKEN`; commit-SHA and version tags MUST NOT be overwritten, and
  a `latest` tag MUST NOT be published.
- **Architecture tooling** (ADR-0003): LikeC4 (Node.js >= 22) is a development-time tool for
  the architecture model only; it is not a runtime dependency.
- **Not yet decided**: the hosting/runtime target and deployment, the API contract between
  frontend and backend (versioning, typed-client generation), and any technology not listed
  above. Each MUST be introduced through an ADR before code depending on it is merged.
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
     a structural change in `plan.md` that is missing from the model is an issue.
  4. **Merge gate** — the pull request links `spec.md`, `plan.md`, and `tasks.md`; the CI
     workflow of every affected deployable passes — build, full test suite, Spring Modulith
     verification, Flyway migrations applied to a test database, and the container image
     build; the architecture model passes validation and the `speckit-architecture` verification
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

**Version**: 1.2.0 | **Ratified**: 2026-09-30 | **Last Amended**: 2026-10-01
