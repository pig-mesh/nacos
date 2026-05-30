<!--
  Copyright 1999-2026 Alibaba Group Holding Ltd.

  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->

# Spring Boot 4 and PigMesh Group Migration Design

Date: 2026-05-29
Repository: `/Users/lengleng/Downloads/nacos-temp`

## Goal

Upgrade the current Nacos 3.2.2 codebase to Spring Boot 4.x while changing the
project Maven group to the PigMesh coordinate:

- Nacos project version remains `3.2.2`.
- Maven group becomes `io.github.pig-mesh.nacos`.
- Spring Boot dependency management target is `4.0.6`.
- Relevant compatibility fixes are synchronized from `pig-mesh/nacos` without
  importing unrelated console or product features.

Maven Central currently lists `4.1.0-RC1` as the latest published
`spring-boot-dependencies` version, but this design intentionally targets the
latest confirmed Spring Boot 4.0 stable line, `4.0.6`, not an RC release.

## Reference Inputs

The design is based on the current local `develop` branch and these relevant
commits from `https://github.com/pig-mesh/nacos`:

- `a3025de7c`: `chore(deps): spring-boot upgrade from 3.4.10 to 4.0.5`
- `9f3fa6272`: `chore(deps): micrometer upgrade from 1.12.8 to 1.13.0`
- `e997fd149`: `refactor: migrate from javax to jakarta annotations in multiple files`
- `5b20ad569`: `refactor(pom): update groupId and version for Nacos dependencies`

The local repository is newer than the reference repository in some areas:

- Local Nacos revision is `3.2.2`.
- Local Spring Boot version is currently `3.5.14`.
- Local dependency versions such as Logback, PostgreSQL, MCP, and Micrometer may
  already be newer than the PigMesh reference.

Therefore, reference commits are treated as migration guidance rather than
literal patches.

## Spec Boundaries

This change affects build coordinates and server runtime dependencies. It must
respect the existing Nacos specs for lifecycle and compatibility:

- The bootstrap and deployment model must keep the existing `core`, `web`,
  `console`, and `ai-registry` startup phase behavior.
- Runtime environment access must continue to go through existing helpers such
  as `EnvUtil` and `ApplicationUtils`.
- Public API, SDK, storage, plugin SPI, and domain semantics are not intended to
  change as part of this migration.
- Client/API/plugin Java 8 compatibility boundaries must be preserved.

## Proposed Approach

Use a targeted migration instead of a broad cherry-pick.

The implementation should sync the minimum Boot 4 and groupId changes needed
for this repository, while preserving the current 3.2.2 branch state. Do not
pull unrelated PigMesh commits such as console promotional navigation or AI
registry disable switches.

## Maven Coordinate Design

Root `pom.xml`:

- Change root `<groupId>` from `com.alibaba.nacos` to
  `io.github.pig-mesh.nacos`.
- Keep `<version>${revision}</version>`.
- Keep `<revision>3.2.2</revision>`.
- Keep the current module list and build structure unless a Boot 4 compile
  failure requires a focused adjustment.
- Update SCM tag to remain revision-based, for example
  `nacos-all-${revision}`, unless the repository's release process requires a
  fixed tag.

Module POMs:

- Update each module parent groupId to `io.github.pig-mesh.nacos`.
- Prefer `${project.groupId}` for internal module dependencies.
- Keep `com.alibaba.nacos` only where the dependency intentionally points to an
  external artifact or a compatibility coordinate that is not produced by this
  reactor.
- Do not downgrade local dependency versions just because the PigMesh reference
  is older.

## Spring Boot 4 Dependency Design

Set:

```xml
<spring-boot-dependencies.version>4.0.6</spring-boot-dependencies.version>
```

Expected dependency adjustments include:

- Replace `spring-boot-starter-aop` with `spring-boot-starter-aspectj` where
  required by Boot 4 module split behavior.
- Add Boot 4 test modules where the old test autoconfigure dependency no longer
  provides the required classes, especially Web MVC test support.
- Update Spring Boot package imports that moved in Boot 4, including web server,
  MVC test, RestTemplate/TestRestTemplate, Jackson customizer, and Mockito test
  annotations as compile errors reveal them.
- Let Spring Boot dependency management control Spring Framework versions unless
  there is a specific local override with documented justification.

## Jakarta Annotation Design

Migrate server-side `javax.annotation` imports to `jakarta.annotation`:

- `PostConstruct`
- `PreDestroy`
- `Resource`

Apply the PigMesh file list first, then scan the current 3.2.2 tree for
additional `javax.annotation` imports introduced after the reference commits.

Do not make broad package rewrites in generated code or public Java 8 client/API
surfaces unless compilation proves they are required and compatible.

## Source Compatibility Design

Spring Boot 4 may require source changes in:

- server startup classes;
- console and server application classes;
- web server listener tests;
- exception handler tests using Web MVC test support;
- HTTP client based integration test helpers;
- native-image reflection configuration for moved Spring Boot classes.

The implementation should make these changes only where compilation or targeted
tests identify a Boot 4 incompatibility. Existing behavior, route shape,
authentication annotations, and response formats are out of scope.

## Error Handling

Primary migration errors are expected to be build-time errors:

- missing Spring Boot classes after package moves;
- missing test support artifacts;
- stale `javax.annotation` imports;
- internal module dependencies still pointing to old group coordinates;
- dependency convergence or Maven model resolution failures.

Each failure should be fixed at the narrowest owning module. If a failure
indicates a real behavior or spec change, pause implementation and identify the
required spec update before continuing.

## Validation Plan

Run validation in increasing scope:

1. Maven model/dependency sanity:
   `mvn -pl core -am -DskipTests compile` or the nearest failing module scope.
2. Affected module compilation:
   `core`, `config`, `console`, `server`, `prometheus`, and integration test
   aggregators as needed.
3. Test compilation for Boot test API changes:
   `mvn -pl test -am -DskipTests test-compile` or narrower equivalents.
4. Spotless after Java edits:
   run `mvn spotless:apply` and `mvn spotless:check` for the affected module or
   nearest aggregator.
5. If time permits, run a focused set of Boot/web tests touched by the migration.

The full pre-submission command remains:

```bash
mvn -B clean compile apache-rat:check checkstyle:check spotbugs:check spotless:check -DskipTests
```

## Out Of Scope

- Upgrading to Spring Boot `4.1.0-RC1` or any other pre-release.
- Changing Nacos version away from `3.2.2`.
- Importing unrelated PigMesh console features.
- Changing API paths, response contracts, auth semantics, storage schema, or
  plugin SPI semantics.
- Refactoring the root POM beyond what is needed for groupId and Boot 4
  compatibility.

## Open Decisions

No open product decisions remain. The accepted direction is:

- project version: `3.2.2`;
- groupId: `io.github.pig-mesh.nacos`;
- Spring Boot version: `4.0.6`;
- implementation style: targeted synchronization from PigMesh reference commits.
