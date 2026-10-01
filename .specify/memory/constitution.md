<!--
Sync Impact Report
==================
Version change: (unversioned template) → 1.0.0
Bump rationale: initial ratification — all placeholders replaced with concrete governance.

Modified principles (template slot → title):
  - [PRINCIPLE_1_NAME] → I. Specification Before Code
  - [PRINCIPLE_2_NAME] → II. Recorded Architecture Decisions
  - [PRINCIPLE_3_NAME] → III. Verified Behaviour
  - [PRINCIPLE_4_NAME] → IV. Simplicity and Justified Complexity
  - [PRINCIPLE_5_NAME] → V. Repository as Single Source of Truth

Added sections:
  - Technology and Security Constraints (template [SECTION_2_NAME])
  - Development Workflow and Quality Gates (template [SECTION_3_NAME])
  - Governance (filled)

Removed sections: none

Sources: docs/adrs/0001 (MADR), docs/adrs/0002 (Spec Kit); no user-supplied principles.

Follow-up TODOs:
  - Principles III–V and the security constraints are general defaults, not derived from
    project decisions; confirm or amend before the first feature spec.
  - ADR-0002 is still `proposed`; this constitution assumes its acceptance.
  - ADR-0001 front matter says `accepted` but docs/adrs/README.md index lists it as `proposed`.
  - No technology stack chosen yet; add stack constraints by amendment once ADRs exist.
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

### II. Recorded Architecture Decisions

- Every architecturally significant decision MUST be recorded as an ADR in `docs/adrs/` using
  the MADR format and conventions of ADR-0001. Architecturally significant means: a new
  runtime dependency, a new service or deployable component, a change of data store, protocol,
  or external integration, or a new cross-cutting pattern.
- A `plan.md` that makes such a decision MUST link the ADR rather than carry the rationale only
  inside the plan.
- Accepted ADRs MUST NOT be rewritten in substance; a changed decision gets a new ADR that
  supersedes the old one.

Rationale: decision rationale that is not written down gets lost or relitigated.

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

- Principles, specs, plans, tasks, and ADRs MUST live in the repository as Markdown, versioned
  with the code.
- A change that alters specified behaviour MUST update the affected `spec.md` (and `plan.md` if
  the design changed) in the same pull request.
- Decisions made in chat, issues, or meetings are not binding until recorded in the repository.

Rationale: artifacts that drift from the code mislead both humans and agents.

## Technology and Security Constraints

- No technology stack is fixed yet. Each stack choice (language, framework, data store,
  hosting) MUST be introduced through an ADR before code depending on it is merged.
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
     violation is justified in Complexity Tracking; required ADRs exist.
  3. **Analysis gate** — `speckit-analyze` reports no critical issues before `speckit-implement`.
  4. **Merge gate** — the pull request links `spec.md`, `plan.md`, and `tasks.md`; tests pass;
     a reviewer confirms the implementation matches the spec.
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
  `speckit-analyze`, and during pull-request review. Reviewers MUST reject changes that violate
  a principle without a recorded justification.

**Version**: 1.0.0 | **Ratified**: 2026-09-30 | **Last Amended**: 2026-09-30
