---
name: "sdd-implement"
description: "Executes a Spec Kit feature's tasks.md by delegating each phase to agent-selection's decision framework (single agent, delegated, or multi-agent pattern via Herdr), instead of running everything inline like /speckit-implement."
argument-hint: "Optional implementation guidance or task filter (same format as /speckit-implement)"
compatibility: "Requires a Spec Kit project structure (.specify/) with tasks.md generated, and the agent-selection skill in this same repo"
metadata:
  status: experimental
  version: "0.5"
user-invocable: true
disable-model-invocation: false
---

## What this skill does

A replacement for `/speckit-implement` in this repo. Spec Kit lays the groundwork (`spec.md` → `plan.md` →
`tasks.md`) with `/speckit-constitution` → `/speckit-specify` → `/speckit-plan` → `/speckit-tasks`;
this skill takes that groundwork and executes it, but instead of running everything inline in the
current session (as native `speckit-implement` does), it evaluates **each phase of `tasks.md`** with
the decision framework of [[agent-selection]] (`.claude/skills/agent-selection/SKILL.md`) to decide
whether that phase gets resolved in the current session, delegated to a subagent, or handled with a
multi-agent pattern via Herdr.

`speckit-implement` remains installed untouched — it's the native fallback if you ever want to run a
feature without going through agent-selection.

## User Input

```text
$ARGUMENTS
```

If not empty, treat it the same way `speckit-implement` would: implementation guidance or a filter
for which tasks/phases to run.

## Step 1 — prerequisites and context (same as `speckit-implement`, don't reinvent)

Reuse steps 1-4 of `.claude/skills/speckit-implement/SKILL.md` as-is, without rewriting them:

1. Run `.specify/scripts/bash/check-prerequisites.sh --json --require-tasks --include-tasks` and
   resolve `FEATURE_DIR`/`AVAILABLE_DOCS`.
2. Checklist status check (`FEATURE_DIR/checklists/`) — if there are incomplete items, **stop and
   ask** before continuing, same as the native skill.
3. Load context: `tasks.md` (required), `plan.md` (required), `spec.md`,
   `.specify/memory/constitution.md`, and `data-model.md`/`contracts/`/`research.md` if they exist.
4. Project setup verification (ignore files based on the stack detected in `plan.md`) — **with
   judgment, not blindly**: if the feature's actual stack (per `plan.md`) doesn't generate any
   artifact that needs ignoring, explicitly skip this step and say so in the Setup phase report,
   instead of creating a generic `.gitignore` with patterns from stacks this repo doesn't use —
   Constitution Principle II (don't generalize without a concrete need). Confirmed live: a
   maintenance bash script (no build, no dependencies) didn't warrant a `.gitignore` — see
   `CHANGELOG.md` v0.2.

This section is referenced instead of copied so that a future change in `speckit-implement`
(from a `specify` update) doesn't leave this skill with an outdated copy — Constitution Principle
VI (concept over the tool's specific brand/version). This also applies to the rest of Step 1: reusing
by reference doesn't mean applying it blindly — any step of `speckit-implement` that doesn't apply
to the feature's actual case is explicitly skipped, with the reason, not run just because the native
skill does it unconditionally.

## Step 2 — parse `tasks.md` into phases (same as `speckit-implement`)

Extract, in the order they appear in `tasks.md`: **Setup → Foundational → one phase per User
Story (P1, P2, P3...) → Polish**. For each phase, list its tasks `[ID] [P?] [Story] Description`
and mark which ones are tagged `[P]` (parallelizable with each other, touch different files).

## Step 3 — the bridge: each phase is "the task" for agent-selection

For each phase, in its corresponding order (never skip a phase before the previous one finishes,
unless agent-selection decides to run two independent phases in parallel — see below):

1. **Read and apply live** `.claude/skills/agent-selection/SKILL.md`, Step 0 through Step 6 in full,
   treating **this phase's set of tasks** as "the task" that framework evaluates. Don't summarize or
   reinterpret the framework — follow it exactly as written there, including the mandatory Herdr
   check (Step 0, once per `sdd-implement` run, not per phase, unless Step 6 of agent-selection
   detects that Herdr went down midway).
2. **`tasks.md`-specific signals for agent-selection's Step 1/2**:
   - The phase's file count = the distinct files its tasks touch (sum them, don't count per
     individual task).
   - The *risk list* from agent-selection's Step 2 applies here too: if any task in the phase
     touches those patterns (`.env*`, CI/CD, infra, migrations, auth/payments), that phase is never
     Direct inline no matter how few files it has.
   - Tasks marked `[P]` within the phase are the natural signal for **question 4 of agent-selection's
     Step 2** ("is part of the work mechanical/repetitive and separable?") — if there are 2+
     non-trivial `[P]` tasks, that's a concrete reason to escalate to a multi-agent pattern (model
     tiering: orchestrator distributes, minions execute each `[P]` task in parallel) instead of
     running them one by one inline.
   - If the phase is the **Tests** phase of a user story (under TDD, per `tasks.md`), those tasks go
     before the implementation tasks of the same phase — the phase's internal order isn't decided by
     agent-selection, it's still dictated by `tasks.md`.
3. **Report using agent-selection's Step 5 format**, once per phase, before executing it: chosen
   route and why, pattern if applicable, CLI/model per role if an external CLI is involved.
4. **Human confirmation gate** (Constitution Principle III): if agent-selection's Step 1/2 determines
   that the phase touches the risk list or is an irreversible decision, stop and ask for explicit
   user confirmation before executing that phase — don't assume that having started `sdd-implement`
   is already a green light for everything that follows.
5. Execute the phase according to the decided route (inline / single subagent / multi-agent
   pattern), with the same per-file coordination rules that native `speckit-implement` already
   brings: tasks that touch the same file never run concurrently, even if they're in different roles
   of a multi-agent pattern. **The Bash tool's cwd does not persist between calls — confirmed live
   (v0.3)**: every Bash invocation starts in the session's primary working directory, regardless of
   what `cd` ran in a previous call; it only survives within the same compound invocation
   (`cd /absolute/path && command`). For phase commands that depend on a specific cwd (tests, builds,
   scripts that read a relative `.env`), always prefix `cd /absolute/path/of/the/phase &&` in the
   same command — never assume a `cd` from a previous step persists. A failure that looks like a
   code issue (missing environment variable, module not found) could actually be this.
6. **Mark `[X]` in `tasks.md`** for each phase task as it's confirmed done — regardless of whether it
   was executed by this session, a delegated subagent, or a role in a multi-agent pattern. This
   session (the orchestrator) is the only one that edits `tasks.md` — a delegated CLI never writes
   directly to `tasks.md`, it reports back and the orchestrator marks it.
7. **Mid-phase failures**: apply agent-selection's Step 6 as-is (explicit timeouts, never wait
   indefinitely, clean up orphaned tabs, degradation with partial results explicitly reported) —
   same criterion `speckit-implement` already uses of "halt if a non-parallel task fails, continue and
   report if a `[P]` task fails," combined with Herdr's rules. **Extension to external infra (v0.3)**:
   the same principle from agent-selection's Step 6 point 2 ("Herdr can go down midway, with no
   proactive signal — it's detected because the next command fails") applies to any external
   dependency the phase needs (DB, Docker/OrbStack containers, local services) — if a phase that
   depends on external infra fails in an odd way, verify first that infra is still alive before
   assuming the failure is in the code.

## Step 4 — closing

Same as `speckit-implement`'s "Done When", plus the per-phase detail:

- [ ] All `tasks.md` tasks marked `[X]` (or explicitly reported as not completed, with the reason).
- [ ] For each phase: which agent-selection route was used and why (summary, no need to repeat the
      full detail of agent-selection's Step 5 already reported before executing it).
- [ ] Any degradation (agent-selection's Step 6) explicitly reported, never hidden.
- [ ] Implementation validated against `spec.md`/`plan.md`, same as `speckit-implement` requires.

## Notes

- This skill doesn't replace `speckit-converge` or `speckit-analyze` — they keep running the same
  way afterward, because the `tasks.md` format (`[X]` checkboxes) doesn't change.
- It doesn't cover the "proposal" or "review/verify + archive" phases that Spec Kit is missing (see
  the decision memory on the SDD roadmap) — out of scope for now, archived.
- Optional next step once closing is done: `sdd-document`, to update `ARCHITECTURE.md` with what
  this run actually touched — natural to invoke right here, while the per-phase file data from
  Step 2/3 is still fresh in the session, instead of re-deriving it standalone later. Not a
  precondition, never invoked automatically.
