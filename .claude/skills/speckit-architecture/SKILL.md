---
name: "speckit-architecture"
description: "Apply a feature plan's Architecture Impact to the LikeC4 architecture model in docs/architecture/, or verify after implementation that the model matches what was built. Runs as a Spec Kit hook after plan and after implement."
argument-hint: "Optional mode: 'apply' (after plan) or 'verify' (after implement)"
compatibility: "Requires spec-kit project structure with .specify/ directory and Node.js >= 22"
metadata:
  author: "libertus2"
user-invocable: true
disable-model-invocation: false
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Purpose

`docs/architecture/` holds the LikeC4 model of the system ([ADR-0003](../../../docs/adrs/0003-use-likec4-for-architecture-as-code.md)). The constitution (Principle II) requires it to describe the system as built, and requires every structural change to go into the model in the same pull request as the code. This skill keeps the model and the feature artifacts in sync.

## Mode

Choose the mode from the first rule that applies:

1. The user input says `apply` or `verify`: use that mode.
2. You are running as the `after_plan` hook of `speckit-plan`: use **apply**.
3. You are running as the `after_implement` hook of `speckit-implement`: use **verify**.
4. Otherwise, use **verify** if every task in `tasks.md` is checked, and **apply** if not.

## Setup

1. Run `.specify/scripts/bash/check-prerequisites.sh --json --paths-only` from the repository root and parse `FEATURE_DIR`, `FEATURE_SPEC`, `IMPL_PLAN`, and `TASKS`.
2. Read the `## Architecture Impact` section of `IMPL_PLAN`. If the plan has no such section, stop and report that the plan was not created from the project plan template (`.specify/templates/overrides/plan-template.md`); ask the user to add the section.
3. Read all `.c4` files under `docs/architecture/`, starting with `specification.c4`.
4. If `docs/architecture/` does not exist, create the model skeleton first (see [Model skeleton](#model-skeleton)), then continue.

## Apply mode (after plan)

1. If the Architecture Impact section says "None", check that the reason is plausible against `plan.md` (for example, the plan adds no new deployable, external integration, or dependency between components). If it is plausible, report "no model changes" and stop. If it is not, report the mismatch and stop without editing the model.
2. For each row of the Architecture Impact table:
   - **add**: add the element or relationship in the right model file, with `title`, `description`, and `technology` where the plan names one. Use only element kinds defined in `specification.c4`. If the plan needs a kind that does not exist, add it to `specification.c4` and list it in the report, because new kinds need reviewer attention.
   - **change**: update the element or relationship in place.
   - **remove**: delete it and every relationship and view reference that points to it.
   - If the row names an ADR, add `link ../adrs/NNNN-short-title.md` to the element.
3. Update the views listed under **Views affected**. Add a new view only when the plan says so.
4. Do not change model elements that the Architecture Impact table does not list. If you notice that the model is already out of date elsewhere, list it in the report instead of fixing it.
5. Run [Validation](#validation).

## Verify mode (after implement)

1. Compare the implemented code and the completed `tasks.md` with both the Architecture Impact table and the model. Look for:
   - elements or relationships in the table that the implementation did not build
   - structural changes in the code that neither the table nor the model shows (a new deployable, external service, data store, or dependency between components)
   - technology on model elements that differs from what the code uses
2. For each difference, update the model so that it describes what was built, and update the Architecture Impact table in `plan.md` to match (Principle V).
3. If a difference reverses or contradicts an ADR, do not change the model to hide it. Report it as **CRITICAL**: it needs a new ADR or a code change.
4. Run [Validation](#validation).

## Validation

Run from the repository root:

```bash
(cd docs/architecture && npx -y likec4 format --check && npx -y likec4 validate)
```

- If `format --check` fails, run `(cd docs/architecture && npx -y likec4 format)` and validate again.
- If `validate` fails, fix the model and run it again. Do not report success while validation fails.
- If Node.js or `npx` is not available, report that validation could not run. Do not report the model as valid.

## Model skeleton

Create this only when `docs/architecture/` does not exist:

- `docs/architecture/likec4.config.json`: `{ "name": "libertus2" }`
- `docs/architecture/specification.c4`: the element kinds `actor`, `system`, `container`, `component` and the tag `external`
- `docs/architecture/model.c4`: one `system` element named `libertus`, plus actors and external systems known from `spec.md`
- `docs/architecture/views.c4`: a system context view of `libertus` named `index`

## Report

Finish with a short report:

- Mode used, and the model files changed
- Elements and relationships added, changed, or removed
- New element kinds or tags added to `specification.c4` (these need reviewer attention)
- Differences found that were not fixed, with severity
- Validation result (pass, fail, or not run, with the reason)
