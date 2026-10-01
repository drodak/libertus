---
status: proposed
date: 2026-09-30
decision-makers: drodak
---

# Use Spec Kit for Spec-Driven Development

## Context and Problem Statement

Libertus2 is a green-field project that will be built largely with AI coding agents (Claude Code). Without a shared workflow, agent-driven development tends to go straight from a loose prompt to code: requirements stay implicit, design choices are made ad hoc inside a chat session, and the reasoning behind the code is lost when the session ends. We want a Spec-Driven Development (SDD) approach, where a written, reviewable specification is the primary artifact and plans, tasks, and code are derived from it.

Which SDD framework (if any) should we adopt to structure how features move from idea to implementation?

## Decision Drivers

* Green-field project: the framework should help establish project-wide principles and a feature pipeline from zero, not only describe changes to an existing system
* First-class support for Claude Code, our primary coding agent
* All artifacts (principles, specs, plans, tasks) live in the repository as plain Markdown, versioned with the code and readable by humans and agents
* Clear separation of *what/why* (spec) from *how* (plan) from *work breakdown* (tasks), with review points between them
* Agent-agnostic enough that we are not locked into one editor or vendor
* Open source, actively maintained, with low setup cost
* Fits with our ADR practice ([ADR-0001](0001-use-madr-for-architecture-decision-records.md)) instead of replacing it

## Considered Options

* GitHub Spec Kit
* OpenSpec
* BMAD Method
* Kiro (spec-driven IDE)
* No SDD framework (ad-hoc prompting, agent plan mode, and ADRs only)

## Decision Outcome

Chosen option: "GitHub Spec Kit", because it best fits a green-field project: it starts with a project constitution that sets non-negotiable principles, then drives each feature through a structured spec → plan → tasks → implement pipeline, with all artifacts as Markdown in the repo and native Claude Code integration. It is open source, agent-agnostic, and already scaffolded in the repository (`.specify/`, `.claude/skills/speckit-*`).

Conventions:

* Spec Kit configuration and templates live in `.specify/`; the Claude Code integration lives in `.claude/skills/speckit-*`.
* The project constitution (`.specify/memory/constitution.md`) is authored with `speckit-constitution` before the first feature and is the source of project-wide principles. Changes to it are architecturally significant and go through review.
* Every non-trivial feature follows the pipeline: `speckit-specify` → `speckit-clarify` (optional, recommended) → `speckit-plan` → `speckit-tasks` → `speckit-analyze` (recommended) → `speckit-implement`.
* Features are numbered sequentially (per `.specify/init-options.json`), and each gets its own spec directory and branch.
* Specs describe *what* and *why* and stay free of technology choices; technology choices belong in `plan.md`.
* Architecturally significant choices made in a `plan.md` are recorded as ADRs (per ADR-0001), and the plan links to them.
* Small fixes and chores (typos, dependency bumps, trivial bugs) may skip the pipeline.

### Consequences

* Good, because intent, design, and work breakdown are written down and reviewable before code is generated.
* Good, because the constitution gives agents and humans one consistent set of principles to check plans against.
* Good, because artifacts are plain Markdown in the repo, so they version with the code and outlive any single agent session.
* Good, because the step-by-step commands constrain the agent and reduce "vibe coding" drift.
* Good, because it is agent-agnostic, so switching or adding coding agents later is cheap.
* Bad, because it adds overhead per feature; the full pipeline is too heavy for small changes (hence the opt-out above).
* Bad, because spec artifacts can drift from the code if they are not updated when behaviour changes.
* Bad, because Spec Kit is evolving quickly (we are on `1.0.14.dev0`); command names, templates, and layout may change between versions.
* Neutral, because Spec Kit is oriented to building features from scratch; if Libertus2 later becomes a mostly-maintenance codebase we may need to revisit this decision.

### Confirmation

* The constitution is filled in (no `[PLACEHOLDER]` tokens remain) before the first feature spec is created.
* Pull requests implementing a feature link to its `spec.md`, `plan.md`, and `tasks.md`; reviewers check that the implementation matches the spec and that significant plan decisions have ADRs.
* `speckit-analyze` is run before implementation to check spec/plan/tasks consistency.
* Spec Kit version upgrades are done deliberately and reviewed as a normal change.

## Pros and Cons of the Options

### GitHub Spec Kit

<https://github.com/github/spec-kit>

Open-source toolkit from GitHub: a CLI (`specify`) that scaffolds templates and agent commands/skills for a constitution → specify → clarify → plan → tasks → implement workflow.

* Good, because the constitution step suits green-field projects that need to set principles from day one
* Good, because it separates spec, plan, and tasks into distinct, reviewable artifacts
* Good, because it supports many coding agents, including Claude Code
* Good, because it includes quality steps (`clarify`, `analyze`, `checklist`) beyond plain generation
* Good, because it is open source (MIT) with a large, active community
* Neutral, because it relies on shell scripts and templates we may need to customise
* Bad, because it is process-heavy for small changes
* Bad, because it is less suited to describing incremental changes to existing behaviour than delta-based tools

### OpenSpec

<https://github.com/Fission-AI/OpenSpec>

Lightweight SDD tool built around change proposals that describe deltas against a living set of specs.

* Good, because it is lightweight and fast to adopt
* Good, because delta-based change proposals work well for evolving existing systems
* Good, because it is open source and supports several agents
* Bad, because its strengths are in brownfield/iterative change rather than establishing a new project from scratch
* Bad, because it has less structure for project-wide principles and up-front planning

### BMAD Method

<https://github.com/bmad-code-org/BMAD-METHOD>

Agile-style framework that uses multiple specialised agent personas (analyst, PM, architect, scrum master, developer, QA) to produce PRDs, architecture docs, and stories.

* Good, because it covers the full product lifecycle, including product discovery
* Good, because role-based agents give rich planning artifacts
* Bad, because it is significantly heavier (many personas and documents) than we need for a small team
* Bad, because the learning curve and ceremony are higher

### Kiro (spec-driven IDE)

<https://kiro.dev/>

AWS's agentic IDE with built-in spec mode (requirements → design → tasks).

* Good, because spec-driven workflow is built into the editor with a polished UX
* Bad, because it ties the workflow to a specific IDE and vendor rather than working with Claude Code in any editor
* Bad, because the workflow is less transparent and customisable than repository templates

### No SDD framework

Rely on ad-hoc prompts, the agent's built-in plan mode, and ADRs.

* Good, because there is no extra tooling or process
* Good, because it is maximally flexible
* Bad, because requirements and design stay implicit and are lost with the chat session
* Bad, because results vary strongly between sessions and contributors
* Bad, because there is no consistent place for agents to find project principles and feature intent

## More Information

* Spec Kit repository: <https://github.com/github/spec-kit>
* Spec Kit is already initialised in this repository (`.specify/init-options.json`: integration `claude`, script `sh`, sequential feature numbering). Accepting this ADR confirms that setup rather than introducing it.
* Revisit this decision if the per-feature overhead consistently outweighs the benefit, if Spec Kit stops being maintained, or once the project moves mainly into maintenance, where a delta-based tool such as OpenSpec may fit better.
