---

description: "Task list template for feature implementation"
---

# Tasks: [FEATURE NAME]

**Input**: Design documents from `/specs/[###-feature-name]/`

**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: Test tasks are MANDATORY (constitution Principle III, ADR-0014). Every acceptance scenario in spec.md has at least one test task that names its identifier (`[###]-US1-AS1`), and that task comes before the implementation tasks of its story. Scenario tests are written first and MUST fail before implementation starts.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions
- A test task names every acceptance scenario it covers by identifier (e.g., `003-US1-AS2`)

## Path Conventions

- **API contract**: `api/openapi.yaml` (ADR-0013)
- **Backend**: `backend/src/main/java/[base package]/[module]/`, tests in `backend/src/test/java/[base package]/[module]/`, migrations in `backend/src/main/resources/db/migration/`
- **Frontend**: `frontend/src/app/`, tests as `*.spec.ts` next to the code they test
- **End-to-end**: `e2e/`
- Backend test class suffixes name the layer (ADR-0014): `…Test` (unit), `…IntegrationTest` (module integration), `…ApiTest` (API), `…ArchitectureTest` (fitness)
- Generated code (backend API interfaces and models, frontend client) is never a task target: it is produced from `api/openapi.yaml` by the build

<!--
  ============================================================================
  IMPORTANT: The tasks below are SAMPLE TASKS for illustration purposes only.

  The /speckit-tasks command MUST replace these with actual tasks based on:
  - User stories and acceptance scenarios from spec.md (with their priorities P1, P2, P3...)
  - Feature requirements from plan.md
  - Entities from data-model.md
  - Operations in api/openapi.yaml

  Tasks MUST be organized by user story so each story can be:
  - Implemented independently
  - Tested independently
  - Delivered as an MVP increment

  Within each story the order is fixed (ADR-0013, ADR-0014):
  1. API contract change, if the story adds or changes an endpoint
  2. Scenario tests, each at the lowest layer that can prove its scenario
  3. Implementation, with unit tests written before or together with the code

  Every acceptance scenario of the story MUST appear by identifier in at least one test
  task. An end-to-end test task is added only for a story that spec.md marks as a
  critical journey.

  DO NOT keep these sample tasks in the generated tasks.md file.
  ============================================================================
-->

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Create project structure per implementation plan
- [ ] T002 Add the dependencies and build configuration the plan introduces
- [ ] T003 [P] Configure linting and formatting tools

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

Examples of foundational tasks (adjust based on your project):

- [ ] T004 Create the application module package and its `@ApplicationModule` declaration in backend/src/main/java/[base package]/[module]/package-info.java
- [ ] T005 [P] Add shared schemas (e.g., the Problem Details error schema) to api/openapi.yaml
- [ ] T006 [P] Create the Flyway migration for tables all stories depend on in backend/src/main/resources/db/migration/V[n]__[name].sql
- [ ] T007 [P] Add architecture fitness rules the plan introduces in backend/src/test/java/[base package]/[Name]ArchitectureTest.java
- [ ] T008 Configure error handling and logging infrastructure

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - [Title] (Priority: P1) 🎯 MVP

**Goal**: [Brief description of what this story delivers]

**Independent Test**: [How to verify this story works on its own]

### API Contract for User Story 1 (only if the story adds or changes an endpoint)

- [ ] T009 [US1] Add [operationId] to api/openapi.yaml with its success and error responses

### Tests for User Story 1 (MANDATORY - write first) ⚠️

> **Write these tests FIRST and run them: each MUST fail because the behaviour is missing, not because the test is broken**

- [ ] T010 [P] [US1] API test for [operationId] covering [###]-US1-AS1 and each declared error response in backend/src/test/java/[base package]/[module]/[Name]ApiTest.java
- [ ] T011 [P] [US1] Module integration test for [behaviour] covering [###]-US1-AS2 in backend/src/test/java/[base package]/[module]/[Name]IntegrationTest.java
- [ ] T012 [P] [US1] Component test for [behaviour] covering [###]-US1-AS3 in frontend/src/app/[feature]/[name].spec.ts
- [ ] T013 [US1] End-to-end test for the main path of [journey] in e2e/[name].spec.ts (only if spec.md marks this story as a critical journey)

### Implementation for User Story 1

- [ ] T014 [P] [US1] Create the Flyway migration for [table] in backend/src/main/resources/db/migration/V[n]__[name].sql
- [ ] T015 [P] [US1] Implement [domain logic] with its unit tests in backend/src/main/java/[base package]/[module]/ and backend/src/test/java/[base package]/[module]/[Name]Test.java
- [ ] T016 [US1] Implement [repository/persistence] in backend/src/main/java/[base package]/[module]/ (depends on T014)
- [ ] T017 [US1] Implement the controller for the generated [Tag] API interface in backend/src/main/java/[base package]/[module]/ (depends on T009, T015, T016)
- [ ] T018 [US1] Implement [component/page] using the generated client in frontend/src/app/[feature]/
- [ ] T019 [US1] Run the tests of T010–T013 and confirm they pass without changing their assertions

**Checkpoint**: At this point, User Story 1 should be fully functional and testable independently

---

## Phase 4: User Story 2 - [Title] (Priority: P2)

**Goal**: [Brief description of what this story delivers]

**Independent Test**: [How to verify this story works on its own]

### API Contract for User Story 2 (only if the story adds or changes an endpoint)

- [ ] T020 [US2] Add [operationId] to api/openapi.yaml with its success and error responses

### Tests for User Story 2 (MANDATORY - write first) ⚠️

- [ ] T021 [P] [US2] API test for [operationId] covering [###]-US2-AS1 and each declared error response in backend/src/test/java/[base package]/[module]/[Name]ApiTest.java
- [ ] T022 [P] [US2] Component test for [behaviour] covering [###]-US2-AS2 in frontend/src/app/[feature]/[name].spec.ts

### Implementation for User Story 2

- [ ] T023 [P] [US2] Implement [domain logic] with its unit tests in backend/src/main/java/[base package]/[module]/
- [ ] T024 [US2] Implement the controller method for [operationId] in backend/src/main/java/[base package]/[module]/
- [ ] T025 [US2] Implement [component/page] in frontend/src/app/[feature]/
- [ ] T026 [US2] Integrate with User Story 1 components (if needed)

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently

---

## Phase 5: User Story 3 - [Title] (Priority: P3)

**Goal**: [Brief description of what this story delivers]

**Independent Test**: [How to verify this story works on its own]

### API Contract for User Story 3 (only if the story adds or changes an endpoint)

- [ ] T027 [US3] Add [operationId] to api/openapi.yaml with its success and error responses

### Tests for User Story 3 (MANDATORY - write first) ⚠️

- [ ] T028 [P] [US3] API test for [operationId] covering [###]-US3-AS1 and each declared error response in backend/src/test/java/[base package]/[module]/[Name]ApiTest.java
- [ ] T029 [P] [US3] Module integration test for [behaviour] covering [###]-US3-AS2 in backend/src/test/java/[base package]/[module]/[Name]IntegrationTest.java

### Implementation for User Story 3

- [ ] T030 [P] [US3] Implement [domain logic] with its unit tests in backend/src/main/java/[base package]/[module]/
- [ ] T031 [US3] Implement the controller method for [operationId] in backend/src/main/java/[base package]/[module]/
- [ ] T032 [US3] Implement [component/page] in frontend/src/app/[feature]/

**Checkpoint**: All user stories should now be independently functional

---

[Add more user story phases as needed, following the same pattern]

---

## Phase N: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [ ] TXXX [P] Documentation updates in docs/
- [ ] TXXX Code cleanup and refactoring
- [ ] TXXX Performance optimization across all stories
- [ ] TXXX Security hardening
- [ ] TXXX Confirm every acceptance scenario identifier in spec.md is carried by at least one test
- [ ] TXXX Run quickstart.md validation

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3+)**: All depend on Foundational phase completion
  - User stories can then proceed in parallel (if staffed)
  - Or sequentially in priority order (P1 → P2 → P3)
- **Polish (Final Phase)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - No dependencies on other stories
- **User Story 2 (P2)**: Can start after Foundational (Phase 2) - May integrate with US1 but should be independently testable
- **User Story 3 (P3)**: Can start after Foundational (Phase 2) - May integrate with US1/US2 but should be independently testable

### Within Each User Story

- The API contract change comes first: the backend interfaces and the frontend client are generated from it
- Scenario tests MUST be written and FAIL before implementation
- A scenario test is made to pass by changing the code; its assertions change only if the scenario in spec.md changes first
- Unit tests are written before or together with the domain logic they cover
- Migrations and domain logic before persistence, persistence before controllers, backend before the frontend that calls it
- Story complete before moving to next priority

### Parallel Opportunities

- All Setup tasks marked [P] can run in parallel
- All Foundational tasks marked [P] can run in parallel (within Phase 2)
- Once Foundational phase completes, all user stories can start in parallel (if team capacity allows)
- All tests for a user story marked [P] can run in parallel
- Backend and frontend tasks of a story can run in parallel once its contract change is done
- Different user stories can be worked on in parallel by different team members

---

## Parallel Example: User Story 1

```bash
# Launch all tests for User Story 1 together, after the contract change:
Task: "API test for [operationId] covering [###]-US1-AS1 in backend/src/test/java/[base package]/[module]/[Name]ApiTest.java"
Task: "Module integration test for [behaviour] covering [###]-US1-AS2 in backend/src/test/java/[base package]/[module]/[Name]IntegrationTest.java"
Task: "Component test for [behaviour] covering [###]-US1-AS3 in frontend/src/app/[feature]/[name].spec.ts"

# Launch independent implementation tasks for User Story 1 together:
Task: "Create the Flyway migration for [table] in backend/src/main/resources/db/migration/V[n]__[name].sql"
Task: "Implement [domain logic] with its unit tests in backend/src/main/java/[base package]/[module]/"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL - blocks all stories)
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: Test User Story 1 independently
5. Deploy/demo if ready

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready
2. Add User Story 1 → Test independently → Deploy/Demo (MVP!)
3. Add User Story 2 → Test independently → Deploy/Demo
4. Add User Story 3 → Test independently → Deploy/Demo
5. Each story adds value without breaking previous stories

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - Developer A: User Story 1
   - Developer B: User Story 2
   - Developer C: User Story 3
3. Stories complete and integrate independently

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Scenario identifiers in test tasks map each acceptance scenario to its test; the test carries the same identifier (a JUnit `@Tag` on the backend, the test title on the frontend and in end-to-end tests)
- Each user story should be independently completable and testable
- Verify scenario tests fail before implementing
- Test at the lowest layer that can prove the behaviour; use test doubles only for another module's public API, an external system, or the clock; never fake the database
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
