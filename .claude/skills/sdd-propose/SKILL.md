---
name: "sdd-propose"
description: "Frames a Spec Kit feature before specifying it, capturing problem, scope, affected files, risks, and rollback in a traceable proposal."
argument-hint: "Describe the feature you want to frame"
compatibility: "Requires a Spec Kit project structure with .specify/ and the script .specify/scripts/bash/create-new-feature.sh"
metadata:
  status: experimental
  version: "0.2"
user-invocable: true
disable-model-invocation: false
---

## What this skill does

First step of the SDD cycle for this repo. It takes a natural-language feature description,
reuses the Spec Kit mechanism to create a numbered directory under `specs/`, and writes there a
`proposal.md` with the minimal framing before moving on to `/speckit-specify`: problem, scope
included, scope excluded, affected files, risks, and rollback.

Numbering and initial directory creation are not reimplemented here: they are always resolved by
`.specify/scripts/bash/create-new-feature.sh`, which also persists `.specify/feature.json`. The
resulting folder is explicitly handed off as `SPECIFY_FEATURE_DIRECTORY` so that the next
run of `/speckit-specify` writes `spec.md` alongside `proposal.md`.

## User Input

```text
$ARGUMENTS
```

The description the user wrote after `/sdd-propose` is the feature input. If it is
empty, report `No feature description provided` and do not create any directory or file.

## Step 1 — validate the input and load the criteria

1. Consider the full description in `$ARGUMENTS` before proposing anything. Extract actors,
   problem, desired action, data, and constraints that are present.
2. Read `.specify/memory/constitution.md` and apply its six principles, especially not
   generalizing without a concrete need, maintaining version traceability, and separating the
   transferable concept from the specific brand or tool.
3. If a decision that materially changes the scope is missing and there is no reasonable
   assumption, mark it with `[NEEDS CLARIFICATION: concrete question]`. Use at most 3 markers total,
   prioritized by scope, security/privacy, and user experience. Do not invent data to
   avoid a clarification. Present the questions and wait for an answer before closing the proposal.
4. Do not use generic placeholders like `[FEATURE NAME]`, `[DATE]`, `TBD`, or `N/A`. A
   `NEEDS CLARIFICATION` marker is the only exception allowed when the ambiguity is material; once
   the answer is received, replace it with a concrete decision.

## Step 2 — ground the proposal in the real repo (conditional)

Applies the "leer y entender" vs. "generar" split from context engineering: exploring the real
repo state and drafting the proposal are two separate steps, so `archivos_afectados` and `riesgos`
get grounded in verified facts instead of guessed patterns.

1. **Decide whether grounding applies.** Check if the description in `$ARGUMENTS` references
   functionality, modules, or components that already exist in this repo (a brownfield touch), as
   opposed to a wholly new, isolated capability with nothing existing to build on (greenfield). A
   quick look at the repo's top-level structure is enough to decide — no need for a deep read yet.
   - **Greenfield** (nothing existing to explore): skip the rest of this step and record in the
     Completion Report `Grounding: omitido (greenfield, sin código existente relacionado)`. Do not
     force an exploration pass when there's nothing to ground — Constitution Principle II.
   - **Brownfield** (touches or extends existing code): continue to point 2.
2. **Delegate the exploration, don't do it inline.** Launch a dedicated read-only agent (the
   `Explore` agent type — fast, read-only, built for "where is X / which files reference Y") scoped
   to the area described in `$ARGUMENTS`: current architecture, stack, and the specific files or
   patterns that already implement the related behavior. This session drafts `proposal.md`
   afterward with those findings as evidence — it does not explore and generate in the same pass,
   so the draft isn't contaminated by unverified assumptions about the repo's current state.
3. **Treat findings as structure, not behavior.** A static read confirms what exists (files,
   modules, call sites) — it does not confirm that code path works end-to-end. Don't let the
   exploration findings claim a feature "already works" or is "X% complete"; that requires running
   something, which is out of scope here. Use them only to ground `archivos_afectados` in real
   paths and `riesgos` in the real surface already in place, in Step 4.
4. Record in the Completion Report which case applied (`omitido` or grounded) so the next skill in
   the chain knows whether `archivos_afectados` reflects verified paths or best-effort guesses.

## Step 3 — create the feature directory with the existing mechanism

1. With the description available, derive a short name of 2 to 4 words, in action or
   concept form, to pass to `create-new-feature.sh` only if it's necessary to set the name. Do not
   invent a numeric prefix or scan `specs/` to number it.
2. Run `.specify/scripts/bash/create-new-feature.sh` with the feature description and
   `--json`. Reuse the script as-is; do not copy or reimplement its numbering algorithm,
   template resolution, `spec.md` creation, or `.specify/feature.json` persistence.
3. Take the `SPEC_FILE` and `FEATURE_NUM` returned by the script as evidence. The feature
   directory is the directory containing that `SPEC_FILE`; keep the exact path, including
   `specs/` and the numeric prefix. If the script fails, report the error and do not write a
   proposal in another folder.
4. Use the directory resolved by the script as `SPECIFY_FEATURE_DIRECTORY` for everything that
   follows. Do not create a second folder, even if the branch name and the directory name
   differ.

## Step 4 — draft `proposal.md`

Write `SPECIFY_FEATURE_DIRECTORY/proposal.md` with these six exact sections, in this order.
The headings must keep these data-model field names:

1. `## problema` — what concrete situation motivates the feature, who suffers from it, and what value is sought.
2. `## alcance_incluye` — a concrete list of what's in for this feature.
3. `## alcance_excluye` — a concrete list of what's out, including boundaries that prevent
   generalizing beyond the real case.
4. `## archivos_afectados` — paths or file patterns that will likely be created, read, or
   modified. If Step 2 ran a grounded exploration, use its findings here — real paths, not
   guesses. If an exact path can't yet be determined (including when Step 2 was skipped as
   greenfield), describe the pattern and the reason, without turning it into a generic list of
   possible files.
5. `## riesgos` — feature-specific risks, known mitigations, and the review status of the
   `agent-selection` risk list. If Step 2 grounded the proposal, base this on the real existing
   surface it found, not on an assumed one.
6. `## rollback` — concrete steps to revert the feature and restore the prior state; if the
   reversal requires a decision or an irreversible action, flag it.

Write content derived from `$ARGUMENTS`, repo context, the Step 2 grounding findings (when it
ran), and reasonable assumptions. Document assumptions within the relevant section or at the end
of `## problema` as `Supuestos`; do not add a seventh section to the model or leave unresolved
template text. The proposal should be readable for whoever will decide the scope and must not
become a detailed implementation plan.

## Step 5 — explicitly review risk before closing

In `## riesgos`, review the identified files and boundaries against the risk list from Step 2
of `.claude/skills/agent-selection/SKILL.md`: `.env*` and other environment files, SSH or
credentials, CI/CD configuration, infrastructure, database migrations, production/deploy
configuration, and auth, payments, or bulk data deletion.

Leave an explicit conclusion, even when there are no matches:

- `Lista de riesgo de agent-selection: no detectada`, briefly explaining what was reviewed; or
- `Lista de riesgo de agent-selection: detectada`, indicating the specific pattern or file and why
  it applies.

If a risk is detected, do not decide on your own that the feature can move forward. Mark
`CONFIRMACIÓN HUMANA REQUERIDA` in the risks section and in the Completion Report, and stop
any further specification or implementation step until the user explicitly confirms
how to proceed. Detection does not authorize migrations, deploys, exposure of secrets,
auth/payments changes, or deletions.

## Step 6 — Completion Report

Finish with a report to the user that includes:

- `SPECIFY_FEATURE_DIRECTORY`: the exact resolved path value, for example
  `specs/003-user-auth`.
- `SPEC_FILE`: the exact path returned by `create-new-feature.sh`.
- `Grounding`: whether Step 2 ran (brownfield, with a short summary of what it found) or was
  omitted (greenfield).
- `proposal.md`: confirmation that it was written inside `SPECIFY_FEATURE_DIRECTORY` and that it
  contains the six required sections.
- `Riesgo`: explicit result of the `agent-selection` review; if applicable, the pending human
  confirmation gate.
- `Siguiente paso`: the `/speckit-specify` invocation using exactly that value, for example
  `SPECIFY_FEATURE_DIRECTORY=specs/003-user-auth /speckit-specify <feature description>`.

Do not claim that `spec.md` was created by this skill: it is created by `/speckit-specify`. Remember that the
script already persisted the path in `.specify/feature.json`, but still hand off the
explicit `SPECIFY_FEATURE_DIRECTORY` value for the next command.

## Done When

- [ ] Empty input was rejected without creating any files.
- [ ] Grounding (Step 2) was explicitly resolved as either run (brownfield, with findings) or
      skipped (greenfield, with the reason) — never silently omitted.
- [ ] `create-new-feature.sh` was run to resolve the numbered directory and its numbering was
      not reimplemented.
- [ ] `proposal.md` exists in the folder returned by the script.
- [ ] `proposal.md` has `problema`, `alcance_incluye`, `alcance_excluye`, `archivos_afectados`,
      `riesgos`, and `rollback`, with no unresolved generic placeholders.
- [ ] The `agent-selection` risk review was made explicit and any match was
      blocked pending human confirmation.
- [ ] The Completion Report leaves the exact `SPECIFY_FEATURE_DIRECTORY` value for
      `/speckit-specify`.
