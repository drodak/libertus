---
status: accepted
date: 2026-09-30
decision-makers: drodak
---

# Use MADR for Architecture Decision Records

## Context and Problem Statement

Libertus2 is at the very start of development (spec-kit scaffolding only). Architecturally significant decisions — technology choices, structural patterns, integration approaches — will soon be made, and we want to capture *why* they were made, not just *what* was decided. Without a shared record, the rationale lives in chat logs and heads, and gets lost or relitigated.

Which format and conventions should we use to record architecture decisions?

## Decision Drivers

* Decisions must live in the repository, versioned alongside the code they affect
* Plain text / Markdown, readable in any editor, on GitHub, and by AI coding agents
* Low ceremony — writing an ADR should take minutes, not hours
* Must capture considered alternatives and trade-offs, not only the outcome
* Should fit alongside the spec-kit workflow (`.specify/`, specs, plans) without duplicating it
* Well-known format with existing tooling and documentation

## Considered Options

* MADR 4.x (Markdown Architectural Decision Records)
* Michael Nygard's original ADR template
* Y-Statements
* No formal format (free-form notes, wiki pages, or decisions only in PR descriptions)

## Decision Outcome

Chosen option: "MADR 4.x", because it is the only option that combines a lightweight Markdown format with explicit sections for decision drivers, considered options, and their pros and cons — which is exactly the rationale we want to preserve — while keeping most sections optional so small decisions stay small.

Conventions:

* ADRs live in `docs/adrs/`.
* Files are named `NNNN-title-with-dashes.md` with a four-digit, monotonically increasing number (e.g. `0002-choose-database.md`).
* New ADRs start from [`adr-template.md`](adr-template.md).
* Status lifecycle: `proposed` → `accepted` | `rejected`; later `deprecated` or `superseded by [ADR-NNNN](NNNN-....md)`.
* Accepted ADRs are immutable in substance; a changed decision gets a new ADR that supersedes the old one (only the status line of the old one is edited).
* Spec-kit plans (`plan.md`) that make an architecturally significant choice should reference or produce an ADR rather than bury the rationale in the plan.

### Consequences

* Good, because decision rationale is versioned with the code and discoverable by humans and agents.
* Good, because the "Considered Options" and "Pros and Cons" sections force alternatives to be written down.
* Good, because optional sections keep the cost of a small ADR low.
* Bad, because it adds a small amount of process to each significant decision.
* Bad, because ADRs can drift from reality if superseding decisions are not recorded.

### Confirmation

Pull requests that introduce an architecturally significant change (new dependency, new service/component, change of data store, protocol, or cross-cutting pattern) are expected to include or link an ADR. Reviewers check this during code review.

## Pros and Cons of the Options

### MADR 4.x

<https://adr.github.io/madr/>

* Good, because it is Markdown with a clear, well-documented structure
* Good, because it explicitly records drivers, options, and trade-offs
* Good, because most sections are optional (a minimal variant exists)
* Good, because YAML front matter makes status/date machine-readable
* Neutral, because it is slightly longer than Nygard's template
* Bad, because the full template can feel heavy if every section is filled in dogmatically

### Michael Nygard's ADR template

<https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions>

* Good, because it is minimal (Title, Status, Context, Decision, Consequences)
* Good, because it is the most widely recognized ADR format
* Bad, because it has no dedicated place for considered alternatives and their pros and cons
* Bad, because the lack of structure leads to inconsistent records

### Y-Statements

* Good, because a decision fits in a single structured sentence
* Bad, because it is too terse to capture meaningful trade-off discussion
* Bad, because it is less familiar and harder to scan as a document collection

### No formal format

* Good, because there is zero process overhead
* Bad, because rationale is scattered across PRs, chats, and wikis, and is easily lost
* Bad, because there is no consistent way to find, supersede, or review decisions

## More Information

* MADR project: <https://adr.github.io/madr/>
* ADR overview: <https://adr.github.io/>
* This ADR is itself written in MADR format and should be marked `accepted` once agreed.
