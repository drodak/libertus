---
status: accepted
date: 2026-10-01
decision-makers: drodak
---

# Container Images Published to GitHub Container Registry as the Delivery Artifact

## Context and Problem Statement

The repository hosts more than one deployable
([ADR-0007](0007-github-monorepo-for-backend-and-frontend.md)): the Spring Boot backend and the
Angular frontend ([ADR-0011](0011-angular-as-the-frontend-framework.md)). CI
([ADR-0008](0008-github-actions-for-continuous-integration.md)) builds and tests them, but
nothing yet defines what a *released* deployable is or where it is stored. ADR-0008 explicitly
left "pushing a container image" out of scope until a follow-up decision.

Each deployable has a different toolchain (JVM/Gradle, Node/Angular CLI), and no hosting target
has been chosen. The delivery artifact therefore needs to be uniform across deployables and
independent of where it eventually runs.

In what form are deployable components delivered, and where are they published?

## Decision Drivers

* One delivery format for every deployable, regardless of its language or build tool.
* The artifact must not pre-decide the hosting/runtime target, which is still open (ADR-0008).
* The repository and CI already live on GitHub (ADR-0007, ADR-0008); publishing should need no
  extra vendor, account, or long-lived credential.
* Each deployable is versioned and released independently (ADR-0007); the artifact store must
  keep them as separate, individually versioned artifacts.
* A released artifact must be traceable to the exact commit it was built from, and must not
  change after release.

## Considered Options

* OCI container images published to GitHub Container Registry (GHCR)
* OCI container images published to Docker Hub
* OCI container images published to a cloud provider's registry (ECR, Artifact Registry, ACR)
* Native build artifacts (executable JAR, static frontend bundle) published to GitHub
  Packages/Releases, with no container image

## Decision Outcome

Chosen option: "OCI container images published to GitHub Container Registry (GHCR)", because a
container image is the one format that packages both deployables the same way and runs on
essentially any hosting target, and GHCR is already available with the GitHub repository:
workflows authenticate with the built-in `GITHUB_TOKEN`, so there is no second vendor and no
registry secret to manage.

Conventions:

* Every deployable component MUST be delivered as an OCI container image published to GHCR. The
  image is the only release artifact; nothing is deployed from a JAR, a bundle, or a build on
  the target host.
* One image per deployable, named `ghcr.io/<owner>/libertus-<deployable>`
  (`libertus-backend`, `libertus-frontend`). A new deployable gets its own image.
* Each deployable directory contains the `Dockerfile` that builds its image (`backend/Dockerfile`,
  `frontend/Dockerfile`), so both deployables are built by the same mechanism.
* Image layers are ordered from least to most frequently changing, so a typical commit replaces
  only the topmost layer and every layer below it is reused on push and pull.
  * Backend: the executable JAR is never copied into the image as a single layer. It is
    extracted with Spring Boot's layered-JAR support (`java -Djarmode=tools -jar app.jar extract
    --layers`) and each layer is added by its own `COPY`, in this order above the base image:
    `dependencies` (third-party release JARs), `spring-boot-loader`, `snapshot-dependencies`,
    `application` (the project's own classes and resources). The project's own code MUST be the
    top layer, above all third-party JARs.
  * Frontend: above the base image, the web server configuration is one layer and the static
    bundle produced by `ng build` is the top layer. The bundle is not split further, because the
    Angular build merges third-party and project code into the same hashed files.
* Images are built and pushed only by GitHub Actions, in a publish workflow per deployable that
  is path-filtered like its CI workflow (ADR-0008). Workflows log in to GHCR with `GITHUB_TOKEN`
  and the `packages: write` permission; no personal access tokens, and no pushes from developer
  machines.
* Pull requests build the image to prove it still builds, but do not push it.
* A merge to `main` publishes the image tagged with the full commit SHA. A release tag
  (`backend-v1.2.0`, per ADR-0007) publishes the same build tagged with the version (`1.2.0`).
* Version and SHA tags are immutable: they are never overwritten. A `latest` tag is not
  published; consumers reference a version tag or a digest.
* Images carry the standard OCI labels, at minimum `org.opencontainers.image.source` (the
  repository URL), `org.opencontainers.image.revision` (the commit SHA), and
  `org.opencontainers.image.version`, so GHCR links each package to this repository.
* Package visibility and access follow the repository's.

This extends ADR-0008's scope: building and pushing container images is now in scope for the
GitHub Actions workflows. Deploying an image to an environment, and the choice of hosting/runtime
target, remain deferred to a future ADR.

### Consequences

* Good, because backend and frontend are released, stored, and versioned the same way despite
  different toolchains.
* Good, because a container image runs on any container-capable hosting target, so the still-open
  hosting decision is not constrained.
* Good, because publishing needs no new vendor, billing relationship, or long-lived secret.
* Good, because every published image is traceable to a commit through its tag and labels.
* Good, because a commit that changes only project code publishes one small layer; the base
  image and third-party dependency layers are already in GHCR and on the host.
* Neutral, because every merge to `main` produces an image; GHCR storage needs a retention
  policy for old SHA-tagged images so it does not grow without bound.
* Neutral, because a hosting target outside GitHub needs a read credential for GHCR if the
  packages are private.
* Bad, because each deployable now has a `Dockerfile` and a publish workflow to maintain, and
  base images to keep patched.
* Bad, because it ties the artifact store to GitHub, deepening the dependence already created by
  ADR-0007 and ADR-0008.
* Bad, because local development needs a container runtime to build or run the delivery
  artifact.

### Confirmation

Each directory listed as a deployable in ADR-0007 contains a `Dockerfile` and has a publish
workflow under `.github/workflows/`. After a merge to `main` that touches a deployable,
`ghcr.io/<owner>/libertus-<deployable>:<commit-sha>` exists and can be pulled. A pull request
that adds a deployable without both is not merged.

Layering is checked by reading the `Dockerfile` in review: the backend has one `COPY` per
extracted layer in the order above, with `application` last. For two consecutive backend images
that differ only in project code, `docker history` shows identical layer digests for everything
below the `application` layer.

## Pros and Cons of the Options

### OCI container images published to GHCR

* Good, because it is part of the GitHub account the repository already uses.
* Good, because `GITHUB_TOKEN` authentication removes stored registry credentials from CI.
* Good, because packages link to the repository and inherit its access model.
* Neutral, because private-package storage and transfer count against the GitHub plan's
  allowance.

### OCI container images published to Docker Hub

* Good, because it is the default registry most tools assume.
* Bad, because it adds a second vendor, a separate account, and an access token stored as a CI
  secret.
* Bad, because pull rate limits and private-repository limits apply on its lower tiers.

### OCI container images published to a cloud provider's registry

* Good, because the registry sits next to the runtime, with native IAM and fast pulls.
* Bad, because no hosting provider has been chosen; picking one registry now would pre-decide
  that.
* Bad, because CI needs cloud credentials or federation set up before anything can be published.

### Native build artifacts, no container image

* Good, because there is no `Dockerfile` or base image to maintain.
* Bad, because the two deployables would be delivered in two different formats, each needing its
  own runtime provisioned on the target (a JRE; a web server).
* Bad, because the runtime environment is then defined by the host rather than by the artifact,
  so what was tested in CI is not what runs.

## More Information

* Working with the Container registry: <https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry>
* Publishing container images from GitHub Actions: <https://docs.github.com/en/actions/use-cases-and-examples/publishing-packages/publishing-docker-images>
* OCI image annotations: <https://github.com/opencontainers/image-spec/blob/main/annotations.md>
* Spring Boot layered JARs and Dockerfiles: <https://docs.spring.io/spring-boot/reference/packaging/container-images/dockerfiles.html>
* Hosting, deployment, and the runtime target are intentionally still open; raise a new ADR once
  a target environment is chosen.
