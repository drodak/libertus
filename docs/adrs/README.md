# Architecture Decision Records

This directory holds the Architecture Decision Records (ADRs) for Libertus2, written in [MADR](https://adr.github.io/madr/) format (see [ADR-0001](0001-use-madr-for-architecture-decision-records.md)).

## Adding a new ADR

1. Copy [`adr-template.md`](adr-template.md) to `NNNN-short-title-with-dashes.md`, using the next free number.
2. Fill in the sections; delete optional ones you don't need.
3. Start with `status: proposed`; change it to `accepted` or `rejected` once decided.
4. Add a row to the index below.

To change an accepted decision, write a new ADR and set the old one's status to `superseded by [ADR-NNNN](NNNN-....md)`.

## Index

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [0001](0001-use-madr-for-architecture-decision-records.md) | Use MADR for Architecture Decision Records | proposed | 2026-09-30 |
| [0002](0002-use-spec-kit-for-spec-driven-development.md) | Use Spec Kit for Spec-Driven Development | proposed | 2026-09-30 |
| [0003](0003-use-likec4-for-architecture-as-code.md) | Use LikeC4 for Architecture as Code | proposed | 2026-10-01 |
