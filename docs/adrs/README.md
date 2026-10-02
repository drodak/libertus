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
| [0001](0001-use-madr-for-architecture-decision-records.md) | Use MADR for Architecture Decision Records | accepted | 2026-09-30 |
| [0002](0002-use-spec-kit-for-spec-driven-development.md) | Use Spec Kit for Spec-Driven Development | accepted | 2026-10-01 |
| [0003](0003-use-likec4-for-architecture-as-code.md) | Use LikeC4 for Architecture as Code | accepted | 2026-10-01 |
| [0004](0004-modular-monolith-with-spring-modulith.md) | Modular Monolith as the Architectural Style, Enforced with Spring Modulith | accepted | 2026-10-01 |
| [0005](0005-java-25-and-spring-boot-as-application-platform.md) | Java 25 (LTS) and Spring Boot 4.1 as the Application Platform | accepted | 2026-10-01 |
| [0006](0006-gradle-as-the-build-tool.md) | Gradle as the Build Tool, Single-Module Backend Layout | accepted | 2026-10-01 |
| [0007](0007-github-monorepo-for-backend-and-frontend.md) | GitHub Monorepo Hosting the Backend and Frontend Applications | accepted | 2026-10-01 |
| [0008](0008-github-actions-for-continuous-integration.md) | GitHub Actions for Continuous Integration | accepted | 2026-10-01 |
| [0009](0009-postgresql-as-default-data-store.md) | PostgreSQL as the Default Data Store, with Criteria for Justified NoSQL Exceptions | accepted | 2026-10-01 |
| [0010](0010-jpa-with-spring-jdbc-fallback-for-data-access.md) | JPA as the Primary Data Access Technology, Spring JDBC as the Documented Fallback | accepted | 2026-10-01 |
| [0011](0011-angular-as-the-frontend-framework.md) | Angular as the Frontend Framework | accepted | 2026-10-01 |
| [0012](0012-container-images-published-to-ghcr.md) | Container Images Published to GitHub Container Registry as the Delivery Artifact | accepted | 2026-10-01 |
| [0013](0013-openapi-spec-first-api-contract.md) | OpenAPI Spec-First Contract for the Frontend–Backend API | accepted | 2026-10-01 |
| [0014](0014-testing-strategy-by-layer.md) | Testing Strategy: Module-Centred Test Layers with Real Infrastructure | accepted | 2026-10-01 |
| [0015](0015-trunk-based-development-with-short-lived-feature-branches.md) | Trunk-Based Development with One Short-Lived Branch per Feature | accepted | 2026-10-03 |
