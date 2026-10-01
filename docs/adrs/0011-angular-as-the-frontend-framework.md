---
status: accepted
date: 2026-10-01
decision-makers: drodak
---

# Angular as the Frontend Framework

## Context and Problem Statement

The monorepo ([ADR-0007](0007-github-monorepo-for-backend-and-frontend.md)) includes a frontend
application alongside the Java/Spring Boot backend, with its technology stack deliberately left
open at the time. The frontend needs a framework now.

Which frontend framework should the `frontend/` application be built with?

## Decision Drivers

* Reuse existing team experience — the same driver already applied to the backend's language,
  framework, and build tool choices ([ADR-0005](0005-java-25-and-spring-boot-as-application-platform.md),
  [ADR-0006](0006-gradle-as-the-build-tool.md)).
* Prefer a framework that bundles routing, forms, dependency injection, an HTTP client, and a
  testing setup, rather than one requiring the team to separately select and integrate each of
  those — consistent with the "limit the number of technologies" theme already applied to storage
  ([ADR-0009](0009-postgresql-as-default-data-store.md)) and data access
  ([ADR-0010](0010-jpa-with-spring-jdbc-fallback-for-data-access.md)).
* Must consume the backend's REST API (exposed by the Spring Boot modular monolith,
  [ADR-0004](0004-modular-monolith-with-spring-modulith.md)) comfortably from a typed language.
* There is one frontend application today; the choice should not force adopting monorepo-scale
  frontend tooling for a problem that doesn't exist yet.

## Considered Options

* Angular
* React (with a separately assembled router, forms, and HTTP library)
* Vue.js (with a separately assembled router and state-management library)
* No separate frontend framework — server-rendered views from the backend (e.g. Thymeleaf)

## Decision Outcome

Chosen option: "Angular", because it is the team's existing area of experience — the same
rationale already applied to every backend tooling choice in this stack — and because its
batteries-included design (router, forms, DI, HTTP client, CLI, and testing scaffolding all
shipped as part of the framework) avoids the additional library-selection decisions that React or
Vue would otherwise require.

Conventions:

* The frontend lives at `frontend/` ([ADR-0007](0007-github-monorepo-for-backend-and-frontend.md)),
  written in TypeScript, scaffolded and built with the Angular CLI.
* New code uses Angular's standalone-component APIs rather than `NgModule`-based composition,
  following current Angular convention.
* Angular CLI is the project's build tool; Nx or another monorepo-oriented frontend toolchain is
  **not** adopted now, since there is one frontend application. If a second frontend app or a
  shared frontend library is added later, that is a new ADR revisiting this choice, not a
  default upgrade.
* npm is the package manager, as the zero-extra-setup default that ships with the Angular CLI;
  revisiting it (e.g. for pnpm's disk/install-speed benefits) is a candidate for its own ADR if
  it becomes a real pain point, not a decision made here.
* The frontend consumes the backend's REST API via Angular's `HttpClient`. The API contract
  itself (versioning, typed-client generation from an OpenAPI spec, etc.) is out of scope for
  this ADR.

### Consequences

* Good, because it reuses deep, existing team expertise directly, the same way the backend stack
  choices do.
* Good, because routing, forms, DI, HTTP, and testing come from one framework instead of several
  independently chosen and upgraded libraries, keeping the frontend's dependency surface small.
* Good, because TypeScript gives a typed consumption layer for the backend's API, with a typed
  client generator (e.g. from OpenAPI) a natural, separately-decided follow-up.
* Neutral, because Angular's roughly six-month major-version cadence requires periodic upgrade
  work; `ng update` keeps this largely mechanical.
* Bad, because Angular's learning curve is steeper than a minimal library for someone without
  prior exposure — not a live concern here, since prior experience is the explicit reason for
  choosing it, but worth naming for future hires.
* Bad, because Angular's ecosystem of ready-made UI component libraries is smaller than React's,
  which can mean more build-it-yourself work for a specific niche widget.

### Confirmation

The `frontend/` CI workflow anticipated by [ADR-0008](0008-github-actions-for-continuous-integration.md)
runs the Angular CLI's build and test commands (`ng build`, `ng test`) as required status checks
before merge.

## Pros and Cons of the Options

### Angular

<https://angular.dev/>

* Good, because it matches the team's stated existing experience.
* Good, because routing, forms, DI, HTTP client, CLI, and testing are part of the framework, not
  separately chosen libraries.
* Good, because TypeScript is mandatory, not optional, giving consistent typing across the
  codebase.
* Neutral, because its opinionated structure means less flexibility to deviate from Angular's own
  conventions than a minimal library would allow.
* Bad, because its component-library ecosystem is smaller than React's.

### React

* Good, because of its very large ecosystem and component-library choice.
* Bad, because routing, forms, and an HTTP client are not included — each must be separately
  selected, integrated, and kept up to date, adding exactly the kind of extra technology surface
  the team has chosen to avoid elsewhere in this stack.
* Bad, because it is not the team's stated area of prior experience.

### Vue.js

* Good, because it has a gentler learning curve than Angular for newcomers.
* Bad, because, like React, routing and state management are separate libraries to select and
  integrate rather than bundled in.
* Bad, because it is not the team's stated area of prior experience.

### No separate frontend framework (server-rendered views)

* Good, because it avoids a second language/toolchain (TypeScript/Node) in the repo entirely.
* Bad, because it was already ruled out by the repo already being structured around a dedicated
  frontend application and deployable (ADR-0007), not server-rendered backend views.

## More Information

* Angular's documentation and update guide: <https://angular.dev/> and <https://angular.dev/update-guide>
* State management approach, styling strategy (e.g. a component library, SCSS, Tailwind), and
  typed API client generation are implementation details for the relevant feature's `plan.md`,
  not this ADR — this ADR covers the framework choice only.
