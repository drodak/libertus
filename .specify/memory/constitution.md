<!--
Sync Impact Report
==================
Version change: 1.0.0 → 1.1.0
Bump rationale: MINOR — Principle II materially expanded (architecture model), gates extended;
no principle removed or redefined incompatibly.

Modified principles (old title → new title):
  - II. Recorded Architecture Decisions → II. Recorded Architecture Decisions and Model
    (adds the LikeC4 model as source of truth for system structure, per ADR-0003)
  - V. Repository as Single Source of Truth (architecture model added to versioned artifacts)

Modified sections:
  - Technology and Security Constraints (LikeC4 tooling noted as development-time only)
  - Development Workflow and Quality Gates (Plan, Analysis and Merge gates cover the model)
  - Governance (compliance check includes model validation)

Added sections: none
Removed sections: none

Dependent artifacts:
  ✅ .specify/templates/overrides/plan-template.md — Architecture Impact section
  ✅ .specify/extensions.yml — after_plan / after_implement hooks to speckit-architecture
  ✅ .claude/skills/speckit-architecture/SKILL.md — applies and verifies model changes
  ✅ .specify/workflows/speckit/workflow.yml — analyze step and gate before implement
  ⚠ .specify/templates/tasks-template.md — no change needed (verification runs as a hook)

Follow-up TODOs:
  - ADR-0003 (LikeC4) is still `proposed`; this amendment assumes its acceptance.
  - docs/architecture/ model skeleton does not exist yet; speckit-architecture creates it on
    first use, or it can be added up front.
  - No CI pipeline exists yet to run `likec4 format --check` and `likec4 validate`.
  - Principles III–V and the security constraints are general defaults, not derived from
    project decisions; confirm or amend before the first feature spec.
  - ADR-0002 is still `proposed`; this constitution assumes its acceptance.
  - ADR-0001 front matter says `accepted` but docs/adrs/README.md index lists it as `proposed`.
  - No application stack chosen yet; add stack constraints by amendment once ADRs exist.
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

- No technology stack is fixed yet. Each stack choice (language, framework, data store,
  hosting) MUST be introduced through an ADR before code depending on it is merged.
- LikeC4 (Node.js >= 22) is adopted by ADR-0003 as a development-time tool for the architecture
  model only; it is not a runtime dependency and does not fix the application stack.
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
     violation is justified in Complexity Tracking; required ADRs exist; the Architecture
     Impact section is filled in and its model changes are applied and valid (the
     `speckit-architecture` hook runs after `speckit-plan`).
  3. **Analysis gate** — `speckit-analyze` reports no critical issues before `speckit-implement`;
     a structural change in `plan.md` that is missing from the model is an issue.
  4. **Merge gate** — the pull request links `spec.md`, `plan.md`, and `tasks.md`; tests pass;
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
- Versioning follows semantic versioning:
  - MAJOR — a principle or gate is removed or redefined incompatibly.
  - MINOR — a principle or section is added or materially expanded.
  - PATCH — clarifications and wording fixes with no change in meaning.
- Compliance is checked at the Plan gate (Constitution Check in every `plan.md`), by
  `speckit-analyze`, by architecture model validation, and during pull-request review. Reviewers MUST reject changes that violate
  a principle without a recorded justification.

**Version**: 1.1.0 | **Ratified**: 2026-09-30 | **Last Amended**: 2026-10-01
