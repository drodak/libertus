---
status: accepted
date: 2026-10-01
decision-makers: drodak
---

# GitHub Monorepo Hosting the Backend and Frontend Applications

## Context and Problem Statement

Libertus will consist of at least two deployables from the start: the Java/Spring Boot modular
monolith backend ([ADR-0004](0004-modular-monolith-with-spring-modulith.md),
[ADR-0005](0005-java-25-and-spring-boot-as-application-platform.md)) and a frontend application
[ADR-0011](0011-angular-as-the-frontend-framework.md). The team wants a single GitHub repo
for the whole system rather than coordinating changes across multiple repositories.

How should the repository be structured so two independently built and deployed applications can
share one repo without their builds, CI, and releases becoming entangled?

## Decision Drivers

* Atomic commits/PRs across a backend contract change and its frontend consumer, without
  cross-repo coordination.
* Single place for specs, plans, ADRs, and the LikeC4 architecture model (already the case per
  ADR-0001–0003) to cover the whole system, not just the backend.
* Each deployable must still be independently buildable, testable, versioned, and deployable —
  the repo boundary must not become a release-coupling boundary.
* CI ([ADR-0008](0008-github-actions-for-continuous-integration.md)) must not rebuild/retest the
  frontend on a backend-only change, or vice versa.
* Low coordination overhead for a small team.

## Considered Options

* Monorepo: one GitHub repo, one top-level directory per deployable
* Polyrepo: a separate GitHub repo per deployable
* Monorepo via Git submodules (separate repos, composed into one checkout)

## Decision Outcome

Chosen option: "Monorepo: one GitHub repo, one top-level directory per deployable", because it
keeps cross-cutting changes (a backend API change and its frontend consumer) atomic and reviewable
in a single PR, and keeps the specs/ADRs/architecture model that already live in this repo
covering the whole system instead of being split or duplicated across repos.

Conventions:

* Top-level layout: `backend/` (the Spring Boot modular monolith, ADR-0004–0006), `frontend/`
  (the frontend application, stack TBD by its own ADR), plus the existing shared `docs/`,
  `.specify/`, and `.github/` at the repo root.
* Each deployable has its own build tool invocation and its own CI workflow
  (ADR-0008), triggered by GitHub Actions path filters scoped to that deployable's directory (plus
  any genuinely shared paths, e.g. a shared OpenAPI contract) — a backend-only change must not run
  the frontend pipeline and vice versa.
* Each deployable is versioned and released independently; if tags are used, they are prefixed by
  deployable (e.g. `backend-v1.2.0`, `frontend-v1.2.0`) so a tag unambiguously identifies which
  artifact it describes.
* `docs/adrs/`, `docs/architecture/`, and `.specify/` remain system-wide, covering both
  deployables; an ADR or model element that concerns only one deployable says so.

### Consequences

* Good, because a single PR can change a backend endpoint and its frontend caller together,
  reviewed and merged atomically.
* Good, because specs, ADRs, and the architecture model already in this repo naturally extend to
  cover the frontend without a second set of artifacts to keep in sync.
* Good, because a small team avoids the coordination tax of cross-repo versioning and dependency
  updates.
* Neutral, because CI must be deliberately scoped with path filters; done, this is routine GitHub
  Actions configuration, not a structural problem.
* Bad, because repo-wide operations (e.g. `git log`, blame, search) mix history from unrelated
  deployables, which path-scoped tooling and discipline only partly mitigates.
* Bad, because a misconfigured path filter can silently under- or over-trigger CI; this is a
  review-time check on `.github/workflows/*.yml`, not an automatic guarantee.

### Confirmation

Workflow trigger `paths:`/`paths-ignore:` filters in `.github/workflows/*.yml` are reviewed
whenever a workflow is added or changed, confirming each workflow only fires for the deployable
directory (and genuinely shared paths) it is responsible for.

## Pros and Cons of the Options

### Monorepo, one directory per deployable

* Good, because cross-cutting changes are atomic.
* Good, because the existing spec-kit/ADR/architecture-model setup already assumes one repo.
* Bad, because CI must actively scope itself via path filters or waste time/minutes rebuilding
  everything on every change.

### Polyrepo

* Good, because each deployable's CI is trivially scoped — there is nothing else in the repo to
  accidentally trigger on.
* Bad, because a change spanning backend and frontend needs two PRs, two reviews, and manual
  coordination to land together.
* Bad, because specs, ADRs, and the architecture model would need to either live in a third repo
  or be duplicated/split, undermining the "repository as single source of truth" principle
  already adopted for this project.

### Monorepo via Git submodules

* Good, because it nominally keeps per-deployable repos while presenting one checkout.
* Bad, because submodules are a well-known source of contributor friction (detached-HEAD
  checkouts, easy-to-forget `--recurse-submodules`, pinned-commit drift) for marginal benefit over
  a plain monorepo here, since there's no requirement to reuse either deployable outside this
  repo.

## More Information

* The frontend's technology stack was deliberately out of scope here; it is decided in
  [ADR-0011](0011-angular-as-the-frontend-framework.md) (Angular).
* GitHub Actions path filtering: <https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#using-filters>
