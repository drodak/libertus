---
status: accepted
date: 2026-10-01
decision-makers: drodak
---

# GitHub Actions for Continuous Integration

## Context and Problem Statement

The repository needs a CI system to build and test each deployable
([ADR-0007](0007-github-monorepo-for-backend-and-frontend.md)) on every change, and to run the
automated fitness functions other ADRs rely on (Spring Modulith boundary verification in
[ADR-0004](0004-modular-monolith-with-spring-modulith.md); the LikeC4 model validation already
required by the project constitution). No hosting or deployment target has been chosen yet, so
this decision is scoped to build/test only.

Which CI system should run these pipelines, and what is explicitly in and out of scope for it?

## Decision Drivers

* The repository already lives on GitHub; CI should need no extra account, billing relationship,
  or infrastructure to stand up.
* Must support path-scoped workflows so a backend-only change doesn't trigger the frontend
  pipeline and vice versa (ADR-0007).
* Must run on every pull request before merge, per the project constitution's Merge gate (tests
  pass, architecture model validates).
* Deployment target and containerization are not yet decided; CI scope must not implicitly force
  that decision by baking in a specific deploy step.

## Considered Options

* GitHub Actions, CI only (build, test, quality gates)
* GitLab CI (would require moving the repo or mirroring it off GitHub)
* Self-hosted Jenkins
* GitHub Actions, CI **and** CD in one pipeline

## Decision Outcome

Chosen option: "GitHub Actions, CI only (build, test, quality gates)", because the repository
already lives on GitHub so there is no new vendor or infrastructure to adopt, GitHub Actions'
native path filters and PR integration fit the monorepo layout from ADR-0007 directly, and scoping
this ADR to CI only avoids smuggling in a deployment/hosting decision that hasn't been made.

Conventions:

* Workflows live under `.github/workflows/`, one workflow per deployable
  (`backend-ci.yml`, `frontend-ci.yml`), each path-filtered to its own top-level directory plus
  any genuinely shared paths.
* The backend workflow runs, at minimum: `./gradlew :backend:build`, the Spring Modulith
  verification test, and the project's test suite; a separate job or step runs
  `likec4 validate`/`likec4 format --check` against `docs/architecture/` per the existing
  constitution requirement.
* Deployment (building a release artifact for a hosting target, pushing a container image,
  triggering a deploy) is explicitly **out of scope** for this ADR. It is deferred to a future ADR
  once a hosting/runtime target is chosen, and that ADR may extend or replace these workflows
  rather than starting from nothing.
* Branch protection on `main` requires these CI workflows to pass before merge, matching the
  constitution's Merge gate.

### Consequences

* Good, because there is no new vendor, billing account, or self-hosted runner infrastructure to
  maintain.
* Good, because per-deployable, path-filtered workflows keep CI time proportional to what actually
  changed.
* Good, because keeping CD explicitly out of scope avoids this ADR quietly pre-deciding hosting.
* Neutral, because GitHub Actions minutes have a cost at scale; acceptable for the project's
  current size and revisit if/when it becomes a real constraint.
* Bad, because deployment will need its own follow-up decision and workflow work later rather than
  landing for free as part of this one.

### Confirmation

Branch protection rules on `main` list the backend and frontend CI workflows as required status
checks; a PR cannot merge while either is red.

## Pros and Cons of the Options

### GitHub Actions, CI only

* Good, because it needs zero new infrastructure given the repo is already on GitHub.
* Good, because its path-filter and matrix features map directly onto the monorepo's two
  deployables.
* Neutral, because its managed-runner minutes are a cost model to watch, not a blocker today.

### GitLab CI

* Good, because GitLab CI is also mature and well documented.
* Bad, because it requires either moving the repository off GitHub or mirroring it, adding
  operational complexity for no corresponding benefit — the repo and its GitHub-specific tooling
  (speckit-taskstoissues, PR-based review) already assume GitHub.

### Self-hosted Jenkins

* Good, because it is fully controllable and vendor-independent.
* Bad, because it requires standing up and maintaining CI infrastructure (a server, agents,
  upgrades, security patching) that GitHub Actions provides for free at this scale.

### GitHub Actions, CI and CD together

* Good, because it would deliver a working deploy pipeline in the same piece of work.
* Bad, because no hosting/runtime target has been chosen yet; bundling CD now would force that
  decision prematurely or bake in a provisional choice that has to be undone.

## More Information

* GitHub Actions path filters: <https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#using-filters>
* CD/hosting is intentionally deferred; raise a new ADR once a target environment is chosen.
