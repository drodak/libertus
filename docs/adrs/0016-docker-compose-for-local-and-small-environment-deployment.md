---
status: accepted
date: 2026-10-03
decision-makers: drodak
---

# Docker Compose as a Deployment Method for Local Development and Small Environments

## Context and Problem Statement

Every deployable is delivered as a container image
([ADR-0012](0012-container-images-published-to-ghcr.md)), and the backend needs PostgreSQL to
start ([ADR-0009](0009-postgresql-as-default-data-store.md)). ADR-0008 and ADR-0012 both left
deployment and the hosting target to a future decision, and the constitution lists them as not
yet decided. As a result there is no defined way to run the system at all: not on a developer's
machine, and not on a single host for a demo or a small installation.

Without one, each developer and each agent session assembles its own way of starting the
database and the application, and what runs locally differs from what is delivered.

How is the whole system — the deployables and the database — started on a developer's machine
and in other small environments, without deciding the hosting of larger environments?

## Decision Drivers

* A developer must be able to start the whole system with one command after cloning the
  repository, with nothing installed besides a container runtime.
* What runs locally should be the delivery artifact of ADR-0012 — the image — and not a second
  way of running the application that is tested nowhere else.
* The same definition should serve small environments that fit on one host, so that a demo or
  a small installation is not a separate piece of work.
* The decision must not pre-decide the hosting of environments that need more than one host;
  that remains open (ADR-0008, ADR-0012).
* Few tools, and nothing to operate (Principle IV). The project has one developer.
* A container runtime is already required locally for the integration tests
  ([ADR-0014](0014-testing-strategy-by-layer.md)).
* No secret is committed to the repository.

## Considered Options

* Docker Compose
* Kubernetes on one node (kind, k3d, or k3s) with manifests or a Helm chart
* Natively run processes: the application started from Gradle, PostgreSQL installed on the host
* Development-time containers started by the application itself (Spring Boot's Testcontainers
  or Docker Compose support at development time)

## Decision Outcome

Chosen option: "Docker Compose", because it starts the images of ADR-0012 and the database with
one command, needs only the container runtime that ADR-0014 already requires, and describes a
single-host environment in one file that works the same on a developer's machine and on a small
server. It is one deployment method, not the only one: it covers environments that fit on one
host, and leaves every larger environment open.

### Scope

* Docker Compose is the deployment method for **small environments**: an environment that runs
  on one host and needs no high availability, no horizontal scaling, and no rolling upgrade.
  This covers a developer's machine, a demo or test environment, and a small self-hosted
  installation.
* It is the only supported way to run the whole system on a developer's machine.
* An environment that needs more than one host, or any of the properties above, is outside this
  ADR. Its deployment method and hosting target are decided by a new ADR. Such a method deploys
  the same images; it does not replace this one for small environments.

### Conventions

* The system is described by `compose.yaml` in the repository root, written to the Compose
  Specification and run with Docker Compose v2 (`docker compose`). The file is a shared
  top-level path in the sense of [ADR-0007](0007-github-monorepo-for-backend-and-frontend.md):
  the CI workflow of every deployable includes it in its path filter.
* `compose.yaml` defines one service per deployable and one service for PostgreSQL. A new
  deployable, and any external system the application needs in order to start, is added to it
  in the pull request that introduces it.
* `compose.yaml` is independent of the environment. It contains no credentials and no defaults
  for them: every value that differs between environments is a variable, and a required one
  has no default, so that starting the system without it fails.
* `compose.override.yaml`, also in the repository root, holds what applies to a developer's
  machine only. Docker Compose reads it automatically, so `docker compose up` works on a fresh
  clone with no further setup. It contains:
  * the instruction to build each deployable's image from the working tree, with the
    `Dockerfile` in the deployable's directory (ADR-0012);
  * development-only values for the required variables, including the database password. These
    are not secrets and are marked as such in the file;
  * the ports published to the host, including the database port.
* Any environment other than a developer's machine is started from `compose.yaml` alone
  (`docker compose -f compose.yaml …`):
  * it runs the images published to GHCR, referenced by a version tag or a digest. Nothing is
    built on the host, and no `latest` tag is used (ADR-0012);
  * its values and secrets are supplied by the operator, through environment variables or an
    `.env` file that is not committed. `.env` is listed in `.gitignore`;
  * the database port is not published.
* Application configuration is passed to the containers as environment variables. The image is
  the same in every environment.
* The PostgreSQL service uses the same major version as the Testcontainers database of
  ADR-0014, keeps its data in a named volume, and declares a health check. A service that
  needs the database starts only after that check passes.
* The schema is created and upgraded by the Flyway migrations when the backend starts
  ([ADR-0010](0010-jpa-with-spring-jdbc-fallback-for-data-access.md)). There is no separate
  migration step.
* Automated tests do not use the Compose services. Integration and API tests start their own
  PostgreSQL with Testcontainers (ADR-0014).
* A developer may run a deployable outside its container — from the IDE or from Gradle —
  against the PostgreSQL service of the Compose project. That is a way of working on one
  deployable, not a second way of running the system.

### Consequences

* Good, because the whole system starts with one command on a fresh clone.
* Good, because what a developer runs is the image that is delivered, built by the same
  `Dockerfile`, so a broken image is found locally and not after publishing.
* Good, because a small environment needs only a host with a container runtime and the two
  commands `docker compose pull` and `docker compose up -d`.
* Good, because it adds no tool to install: Docker Compose ships with the container runtimes
  developers already use for the integration tests.
* Good, because the hosting of larger environments stays open, and whatever is chosen deploys
  the same images.
* Neutral, because the two files have to be kept consistent: a service added to `compose.yaml`
  usually needs its development values in `compose.override.yaml`.
* Neutral, because rebuilding an image on every change is slower than running the application
  from the IDE; the last convention above allows that for day-to-day work.
* Bad, because Docker Compose manages one host. It gives no high availability, no rolling
  upgrade, and no scaling, and an upgrade restarts the service.
* Bad, because in a small environment everything around the containers is left to the
  operator: backups of the database volume, TLS termination, monitoring, and applying upgrades.
  None of it is automated by this decision.
* Bad, because a secret in an `.env` file on the host is protected only by the host's file
  permissions.
* Bad, because a second deployment method added later means two descriptions of the system to
  keep in step.

### Confirmation

* `compose.yaml` and `compose.override.yaml` exist in the repository root, and `.env` is in
  `.gitignore`.
* The CI workflow of each deployable runs `docker compose config` for both forms — with the
  override, and with `compose.yaml` alone and the required variables set — and fails when
  either is invalid.
* On a fresh clone, `docker compose up --build` starts every service, and the backend reports
  that it started with the migrations applied. This is checked by hand when the files change.
* Review rejects a pull request that adds a deployable or a required external system without
  its Compose service, that puts a credential or a default for one in `compose.yaml`, or that
  documents another way of running the whole system locally.

## Pros and Cons of the Options

### Docker Compose

<https://docs.docker.com/compose/>

* Good, because one file describes the whole system and one command starts it.
* Good, because it runs the delivery artifact of ADR-0012 unchanged.
* Good, because it is included with Docker Desktop, OrbStack, and Docker Engine's packages, so
  there is nothing extra to install.
* Good, because it is licensed under Apache-2.0 and actively maintained.
* Neutral, because it is limited to one host, which is the scope decided here.
* Bad, because it does no orchestration: no self-healing beyond a restart policy, no rolling
  upgrade, no scaling across hosts.

### Kubernetes on one node with manifests or a Helm chart

* Good, because the same manifests could later be applied to a real cluster.
* Good, because it provides health-based restarts and rolling upgrades.
* Bad, because it pre-decides Kubernetes as the hosting model, which has not been chosen.
* Bad, because a developer has to install and run a local cluster and load images into it
  before anything starts.
* Bad, because manifests or a chart are far more to write and maintain than one Compose file,
  for a system of two deployables and a database.

### Natively run processes

The backend is started with Gradle, the frontend with the Angular CLI, and PostgreSQL is
installed on the host.

* Good, because the feedback loop is the fastest, and it needs no container runtime to run the
  application.
* Bad, because the delivery artifact is never run locally; a broken `Dockerfile` is found only
  in CI.
* Bad, because every machine needs the right JDK, Node.js, and PostgreSQL versions installed
  and kept in step.
* Bad, because it is not a deployment method: it gives nothing to a demo or a small
  installation.

### Development-time containers started by the application

Spring Boot starts the database itself, from a Testcontainers configuration or through its
Docker Compose support, when the application is run from Gradle or the IDE.

* Good, because the developer starts one process and the database appears.
* Good, because the Testcontainers variant reuses the test configuration of ADR-0014.
* Bad, because it covers the backend only; the frontend and the system as a whole are not
  started.
* Bad, because the application runs from the build, not from its image.
* Bad, because it is a development convenience and gives nothing to any other environment.

## More Information

* Compose Specification: <https://compose-spec.io/>
* Merging of `compose.yaml` and `compose.override.yaml`:
  <https://docs.docker.com/compose/how-tos/multiple-compose-files/merge/>
* This ADR decides part of what ADR-0008 and ADR-0012 deferred: deployment to small
  environments. The hosting and runtime target of larger environments, and automated
  deployment (continuous delivery) to any environment, remain open and need their own ADR.
* Not decided here, each needing its own decision when a small environment other than a
  developer's machine is first set up: TLS termination and the reverse proxy, backup and
  restore of the database volume, and how secrets are stored on the host.
* A health check for the backend service needs an endpoint to call. Which endpoint that is, is
  decided in the plan that adds it; until then only the PostgreSQL service has a health check.
* The end-to-end workflow of ADR-0014 runs the images built from the pull request's commit and
  a PostgreSQL container. It may start them from `compose.yaml`; that is decided with the
  first end-to-end test.
* Revisit this decision when an environment needs more than one host, or when a second
  deployment method makes two descriptions of the system costly to keep in step.
* Accepting this ADR requires amending the constitution:
  * Technology and Security Constraints gains a local and small-environment deployment entry
    that summarises the conventions above, and `compose.yaml` and `compose.override.yaml` are
    added to the shared top-level paths of the Repository entry.
  * "Not yet decided" is narrowed from "the hosting/runtime target and deployment" to the
    hosting and deployment of environments that do not fit on one host, and automated
    deployment.
  * The Merge gate's CI workflow line gains the validation of the Compose files.
  * Principles I–V are unchanged.
