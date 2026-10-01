---
status: accepted
date: 2026-10-01
decision-makers: drodak
---

# Use LikeC4 for Architecture as Code

## Context and Problem Statement

Libertus2 records *why* architectural choices were made in ADRs ([ADR-0001](0001-use-madr-for-architecture-decision-records.md)) and records *what* each feature does and how it is built in Spec Kit specs and plans ([ADR-0002](0002-use-spec-kit-for-spec-driven-development.md)). Nothing yet describes *what the system is*: its components, how they depend on each other, and which external systems they use. Without that, each `plan.md` describes the system again from scratch, diagrams (if any) are drawn by hand and drift from the code, and coding agents have no structured view of the system to plan against.

We want to keep the architecture as code: a text model in the repository, versioned with the code, that diagrams are generated from and that humans, CI, and agents can all read. Which tool should we use for it?

## Decision Drivers

* The model lives in the repository as plain text, is reviewed in pull requests, and follows Principle V of the constitution (repository as single source of truth)
* It is a model, not a set of drawings: elements and relationships are defined once and every diagram is a view of them
* Coding agents can read and query the model while running `speckit-plan`, `speckit-tasks`, and `speckit-analyze`
* CI can validate the model, so a broken or inconsistent model fails the build
* Low setup cost and good editor support (VS Code)
* Flexible enough to model our own concepts rather than only a fixed notation, while still supporting C4-style views
* Open source, actively maintained, and with a licence compatible with the project

## Considered Options

* LikeC4
* Structurizr DSL
* Mermaid or PlantUML C4 diagrams embedded in Markdown
* FINOS CALM (Common Architecture Language Model)
* No architecture model (ADRs and plan prose only)

## Decision Outcome

Chosen option: "LikeC4", because it gives the best tooling for our AI-agent-driven, Spec Kit-based workflow. Its MCP server lets Claude Code query the model directly, `likec4 validate` and `likec4 format --check` work as CI gates, and the VS Code extension offers live preview and a language server. We can also define our own element kinds instead of being limited to fixed C4 levels. It runs on Node.js with no JVM or Docker, and it is MIT-licensed and actively maintained.

Conventions:

* The model lives in `docs/architecture/` as `.c4` files. A `likec4.config.json` there defines the project.
* The specification block (element kinds, relationship kinds, tags) is kept in its own file (`docs/architecture/specification.c4`), so that changes to the vocabulary are easy to see in review.
* Start with C4-style kinds (`actor`, `system`, `container`, `component`) and add project-specific kinds only when a real need appears (Principle IV).
* Elements that an ADR is about link to it with `link ../adrs/NNNN-....md`, so a reader of the model can go from an element to the decisions that shaped it.
* Integration with Spec Kit:
  * `plan.md` gets an **Architecture Impact** section that lists the model elements and relationships the feature adds, changes, or removes, and links any new ADR. This is done by overriding the plan template in `.specify/templates/overrides/plan-template.md`, so upstream templates are not edited.
  * A project skill, `speckit-architecture`, is registered as a mandatory Spec Kit hook in `.specify/extensions.yml`. After `speckit-plan` it applies the Architecture Impact changes to the model and validates it, so the model changes are reviewed together with the plan. After `speckit-implement` it checks that the model matches what was built, fixes drift, and validates the model again.
  * `speckit-analyze` checks, through the constitution, that every structural change in `plan.md` appears in the model. The Spec Kit workflow runs `speckit-analyze` and a review gate before `speckit-implement`.
  * The model is updated in the same pull request as the code that changes the structure.
* The LikeC4 MCP server (`@likec4/mcp`) is registered for Claude Code in the project's `.mcp.json`.
* The constitution is amended (v1.1.0) to require that the model be kept current, extending Principle II and the Plan, Analysis, and Merge gates.

### Consequences

* Good, because there is one structural model that every diagram, plan, and agent works from, instead of each plan describing the system again
* Good, because agents can query the model (elements, relationships, paths between elements) through MCP while planning, instead of guessing structure from code
* Good, because CI validation catches syntax errors and dangling references in the model
* Good, because diagrams are generated from the model and cannot drift from each other
* Good, because it can export to Mermaid, PlantUML, D2, DOT, and draw.io, and can build a static site, so we are not locked into its viewer
* Good, because it needs only Node.js, and the tooling runs locally and in CI with little setup
* Bad, because it adds a Node.js (>= 22) toolchain to the repository before any application stack is chosen. It is a development-time tool only, not a runtime dependency
* Bad, because the model can still drift from the actual code: `likec4 validate` checks that the model is consistent with itself, not with the code
* Bad, because LikeC4 is younger and less widely known than Structurizr, and its DSL and features change quickly between releases
* Bad, because it has no built-in import of our MADR ADRs; linking is done by hand with `link` properties
* Neutral, because flexible element kinds make the model easier to fit to our needs but also easier to make inconsistent; keeping the specification small and reviewing it is our safeguard

### Confirmation

* CI runs `likec4 validate` and `likec4 format --check` on `docs/architecture/` for every pull request, and fails the build on errors.
* Every `plan.md` has an Architecture Impact section. Reviewers check at the Plan gate that the model changes it lists are present in the pull request.
* `speckit-analyze` reports a missing model update as an issue.
* The `speckit-architecture` hooks run after `speckit-plan` and `speckit-implement`, and fail loudly when validation fails.
* Once an application stack is chosen, checking the model against the code with fitness functions (dependency and layering rules as automated tests) will be decided in the stack ADR or in a follow-up ADR. That is out of scope here.

## Pros and Cons of the Options

### LikeC4

<https://likec4.dev/> · <https://github.com/likec4/likec4>

Architecture-as-code DSL and toolchain: a model with user-defined element kinds, views derived from it, a CLI (`serve`, `build`, `validate`, `format`, `export`, `gen`), a VS Code extension with a language server, and an MCP server.

* Good, because the MCP server (`@likec4/mcp`, or built into the VS Code extension) exposes search, element and view reads, and graph queries to coding agents
* Good, because `validate` and `format --check` are designed to run in CI
* Good, because element kinds, relationship kinds, and tags are user-defined, so the model can go beyond fixed C4 levels
* Good, because the VS Code extension offers live preview, navigation, and completion
* Good, because it can export to many formats (Mermaid, PlantUML, D2, DOT, draw.io, PNG, React) and build a static site
* Good, because it is MIT-licensed, actively developed, and needs only Node.js
* Neutral, because it is not the de-facto C4 reference tool, though its views work in the C4 style
* Bad, because it is a younger project with frequent releases, so upgrades need care
* Bad, because it has no built-in ADR integration

### Structurizr DSL

<https://docs.structurizr.com/dsl>

The reference tool for the C4 model, by the author of C4: a `workspace.dsl` file with model, views, documentation, and decisions, viewed with Structurizr Lite, on-premises, or cloud.

* Good, because it is the de-facto standard for C4 and widely known
* Good, because it can import MADR ADRs (`!adrs <path> madr`) and show them next to the model in the Structurizr UI
* Good, because it exports to PlantUML and Mermaid, and the CLI can validate the workspace
* Neutral, because the ADR import only displays decisions; it does not check that they match the model, and the decisions are visible only in Structurizr's viewers
* Bad, because the tooling needs a JVM or Docker
* Bad, because the notation is tied closely to fixed C4 levels, with less room for our own concepts
* Bad, because it has no MCP server, so agents can read only the raw DSL file

### Mermaid or PlantUML C4 diagrams embedded in Markdown

C4 diagram syntax written directly in Markdown files next to specs and plans.

* Good, because there is no extra tooling; Mermaid renders on GitHub
* Good, because it is quick to start with
* Bad, because every diagram is a separate drawing, not a view of a shared model, so diagrams drift from each other
* Bad, because there is nothing meaningful to validate or query

### FINOS CALM (Common Architecture Language Model)

<https://github.com/finos/architecture-as-code>

A JSON/JSON-Schema architecture model from FINOS, aimed at governance, patterns, and compliance controls.

* Good, because models are machine-checkable against patterns and controls
* Good, because it is an open standard backed by a foundation
* Bad, because it is designed for enterprise governance and is far heavier than a small green-field project needs
* Bad, because writing and reading JSON models by hand is awkward, and the diagram tooling is less mature

### No architecture model

Keep structure only in ADRs and in the prose of each `plan.md`.

* Good, because there is no extra tooling or process
* Bad, because structure is spread across many documents and never shown as a whole
* Bad, because agents rebuild their picture of the system from code and scattered plans in every session
* Bad, because nothing can be validated, and drift goes unnoticed

## More Information

* LikeC4 documentation: <https://likec4.dev/>. CLI: <https://likec4.dev/tooling/cli/>. MCP server: <https://likec4.dev/tooling/mcp/>.
* At the time of writing, the latest LikeC4 release is 1.59.4 (MIT, Node.js >= 22). This satisfies the constitution's dependency check (actively maintained, compatible licence).
* This ADR depends on [ADR-0002](0002-use-spec-kit-for-spec-driven-development.md) being accepted, since the conventions above extend the Spec Kit pipeline.
* Follow-up work once this ADR is accepted:
  1. Add `docs/architecture/` with `likec4.config.json`, `specification.c4`, and a system-context model.
  2. Add the CI validation job and register the MCP server in `.mcp.json`.
* Done alongside this ADR (proposed together for review): the plan template override, the `speckit-architecture` skill and its hooks, the analyze step in the Spec Kit workflow, and constitution v1.1.0 (Principle II and the gates).
* Revisit this decision if LikeC4 stops being maintained, if its DSL changes incompatibly in a way that is costly to follow, or if we need governance or compliance checks that a tool such as CALM provides.
