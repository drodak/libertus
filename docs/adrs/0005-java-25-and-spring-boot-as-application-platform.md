---
status: accepted
date: 2026-10-01
decision-makers: drodak
---

# Java 25 (LTS) and Spring Boot 4.1 as the Application Platform

## Context and Problem Statement

The backend needs a language and framework platform. The team already has Java and Spring Boot
experience, and the project constraints call for reusing that rather than adopting an unfamiliar
stack. Within "Java and Spring Boot" there is still a version baseline to pick, and that baseline
constrains other decisions — notably Spring Modulith
([ADR-0004](0004-modular-monolith-with-spring-modulith.md)) is released in lockstep with a
specific Spring Boot line (Spring Modulith 2.1 targets Spring Boot 4.1 / Spring Framework 7.0),
and Spring Boot 4.x requires Java 17 at minimum.

The Spring Boot 3.x line is no longer an option for a new project: open-source support for 3.5,
its last minor release, ended in June 2026.

Which Java and Spring Boot versions should the project standardize on?

## Decision Drivers

* Reuse existing team experience in Java and Spring Boot rather than adopt an unfamiliar runtime
  or framework.
* Start on a Spring Boot line that is within its open-source support window, with as much of
  that window ahead of it as possible; Spring Boot 3.5 reached end of OSS support in June 2026.
* Must satisfy Spring Boot 4.x's minimum of Java 17, since Spring Modulith depends on it.
* Prefer a long-term-support (LTS) Java release so the baseline doesn't need revisiting on every
  six-month release cycle.
* Favor a release with enough ecosystem/library maturity behind it, not just the newest LTS by
  date; equally, don't start a new project on an older LTS once the newer one has matured.
* Virtual threads materially simplify blocking I/O code (JDBC, JPA, outbound HTTP) without
  requiring a reactive rewrite.

## Considered Options

* Java 25 (LTS) + Spring Boot 4.1
* Java 21 (LTS) + Spring Boot 4.1
* Java 17 (LTS) + Spring Boot 4.1

## Decision Outcome

Chosen option: "Java 25 (LTS) + Spring Boot 4.1", because it is the current LTS release and,
having been generally available since September 2025, has had a year for libraries, build
tooling, and IDEs to catch up — Spring Boot 4.1 supports it directly. It carries everything Java
21 offers (virtual threads, record patterns, pattern matching for `switch`) plus later
improvements, notably virtual threads that no longer pin their carrier thread inside
`synchronized` blocks, and it has the longest support runway of the available LTS releases.
Java 21 would mean starting a new project one LTS behind for no corresponding benefit, and Java
17 would additionally forgo virtual threads. Spring Boot 4.1 is chosen over the 3.x
line because 3.5 is past its end of open-source support, over 4.0 because 4.0's open-source
support ends in December 2026, and over 4.2 because 4.2 is not yet generally available. Spring
Modulith is used at 2.1, the release line built against Spring Boot 4.1.

### Consequences

* Good, because virtual threads let blocking JPA/JDBC code
  ([ADR-0010](0010-jpa-with-spring-jdbc-fallback-for-data-access.md)) scale under concurrent load
  without a reactive-stack rewrite.
* Good, because Java 25 is inside Spring Boot 4.1's supported range (Java 17 minimum, up to
  Java 26) and Spring Modulith 2.1 follows Spring Boot's baseline, avoiding
  version-compatibility surprises.
* Good, because `synchronized` no longer pins virtual threads (fixed in Java 24), removing the
  most common virtual-thread pitfall of Java 21 for JDBC drivers and older libraries.
* Good, because the project starts on a supported Spring Boot line instead of inheriting an
  end-of-life 3.x baseline and an immediate 3.x → 4.x migration.
* Good, because Java 25 is the newest LTS, so its vendor support window extends furthest out
  from 2026, giving the most headroom before the next mandatory upgrade.
* Neutral, because the team must still budget for an eventual move to the next LTS.
* Neutral, because the build tooling must itself support Java 25: Gradle 9.1 or later
  ([ADR-0006](0006-gradle-as-the-build-tool.md)), and bytecode-processing libraries and plugins
  must be on versions that understand Java 25 class files.
* Neutral, because Spring Boot 4.1's open-source support ends in July 2027, so a minor upgrade
  to 4.2 (with Spring Modulith 2.2) must be scheduled before then; Spring Boot and Spring
  Modulith minors are upgraded together.
* Bad, because Spring Boot 4.x is a major-version step from the 3.x the team knows (Spring
  Framework 7, modularized auto-configuration, Jackson 3), so some existing habits and 3.x-era
  guides no longer apply as-is.
* Bad, because Java 25 has a shorter production track record than Java 21, so the occasional
  third-party library or tool may still lag behind it.
* Bad, because virtual threads still have pitfalls (pinning during native/JNI calls, unbounded
  concurrency against limited resources such as the connection pool) that the team must learn
  to avoid.

### Confirmation

The Gradle toolchain ([ADR-0006](0006-gradle-as-the-build-tool.md)) pins the Java version to 25;
CI fails the build if the toolchain resolves to a different major version. The Spring Boot
and Spring Modulith versions are each declared once in the Gradle build (Spring Boot plugin and
the Spring Modulith BOM) and are on the 4.1.x and 2.1.x lines respectively.

## Pros and Cons of the Options

### Java 25 (LTS) + Spring Boot 4.1

* Good, because it is the newest LTS and has the longest support runway from today.
* Good, because it has been generally available for a year (since September 2025) and is
  supported by Spring Boot 4.1 and current build tooling.
* Good, because virtual threads no longer pin under `synchronized`, and it adds further runtime
  and language improvements over Java 21 (e.g. scoped values, compact object headers).
* Neutral, because it has less accumulated production mileage than Java 21.

### Java 21 (LTS) + Spring Boot 4.1

* Good, because it is a mainstream, widely deployed LTS for Spring Boot projects.
* Good, because virtual threads are stable (finalized in Java 21, not a preview feature).
* Bad, because virtual threads pin their carrier thread inside `synchronized` blocks, which
  needs care with JDBC drivers and older libraries.
* Bad, because it is no longer the newest LTS, so a new project would start with a shorter
  support runway and an LTS upgrade already pending.

### Java 17 (LTS) + Spring Boot 4.1

* Good, because it is the previous, well-established LTS with broad tooling support.
* Bad, because it forgoes virtual threads, the single biggest practical benefit of moving past
  17 for a JDBC/JPA-heavy backend, for no compensating advantage.
* Bad, because it is two LTS releases behind, with the shortest remaining support runway.

## More Information

* Spring Boot system requirements: <https://docs.spring.io/spring-boot/system-requirements.html>
* Spring Boot support timeline (3.5 OSS support ended June 2026; 4.1 ends July 2027): <https://spring.io/projects/spring-boot#support>
* Spring Modulith's Java/Spring Boot baseline: <https://docs.spring.io/spring-modulith/reference/>
* JEP 491, Synchronize Virtual Threads without Pinning (delivered in Java 24): <https://openjdk.org/jeps/491>
* Revisit this ADR (new ADR superseding it) when the next Java LTS is released and has matured
  enough to be the better default, rather than silently drifting the toolchain version.
