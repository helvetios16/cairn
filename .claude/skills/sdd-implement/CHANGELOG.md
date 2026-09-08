# Changelog — sdd-implement

- **v0.1** — first version. Bridge between the foundations Spec Kit leaves (`spec.md` → `plan.md` →
  `tasks.md`, via `/speckit-constitution` → `/speckit-specify` → `/speckit-plan` → `/speckit-tasks`)
  and actual execution: instead of running native `/speckit-implement` (everything inline in the
  current session), this skill evaluates **each phase of `tasks.md`** with the complete decision
  framework of `agent-selection` (Steps 0–6) to decide on a single agent / delegate / multi-agent
  pattern via Herdr. Reuses by reference (does not copy) the prerequisites, status checklist, and
  setup steps of `speckit-implement` — prevents a future `specify` update from leaving an outdated
  copy here. New signal: the `[P]` tasks in `tasks.md` directly feed question 4 of Step 2 of
  agent-selection (mechanical/separable work → candidate for model tiering). `speckit-implement`
  remains installed untouched as a native fallback. Does not cover the proposal or review/verify+
  archive phases of the SDD roadmap — they remain archived and out of scope for this version.
- **v0.2** — first live trial (Principle I of the constitution), real case: `scripts/check-skills.sh`
  run end to end with `/speckit-specify` → `/speckit-plan` → `/speckit-tasks` →
  `/sdd-implement` (`specs/001-skill-audit/`). All 5 phases resolved Direct inline (one file,
  mechanical, no risk list) — confirms that the bridge does not over-scale a trivial case, but leaves
  the delegated/multi-agent branch unexercised (pending a second trial with a case that activates it).
  Applied finding: step 4 of Step 1 ("reuse as-is" the ignore-files verification from
  `speckit-implement`) applied blindly would have generated a generic `.gitignore` with stack
  patterns this repo does not use, violating Principle II. Clarified that "reuse by reference" is
  not "apply blindly": any reused step that does not apply to the real case is explicitly omitted,
  with the reason.
- **v0.3** — 2 findings from a second feedback round in earpi closed (US1, T011-T014; see
  `TODO.md` in `agent-selection`, section "Real-world usage feedback — earpi, US1 update"). One
  was verified live, more thoroughly than the original report: earpi suspected that the Bash tool's
  cwd sometimes reset due to interleaving with another tool; tested here in isolation and deterministically
  (`cd .../backend && pwd` in one call, `pwd` alone in the next with no other tool in between, repeated
  3 times) — **the cwd returns to the primary working directory on every call, always**, not just
  sometimes. Added an explicit precaution in Step 3, item 5: prefix phase commands that depend on a
  specific cwd with `cd /absolute/path &&` in the same invocation. The second finding (external
  infrastructure — OrbStack/Docker — going down midway through a phase without warning) was applied
  by directly extending the principle that Step 6 of `agent-selection` already uses for Herdr, added
  to item 7 of Step 3: verify that external infrastructure is still running before assuming that a
  mid-phase failure is in the code.
- **v0.4** — no content changes in `SKILL.md`. The confirmation of a third `FEEDBACK.md` round in
  earpi (US2, T015-T019) was recorded in `TODO.md` of `agent-selection`: the fixed-contract
  delegation pattern (Step 3, `tasks.md` signals for agent-selection) produced the correct result
  for the third consecutive time (T008/T009, T015/T016) — no new phase-execution friction in this
  round. The polling fix (`herdr wait agent-status`/`herdr wait output`) also confirmed clean in this
  round belongs to `agent-selection` v0.33, not this skill — see its `CHANGELOG.md` v0.34.
- **v0.5** — adds a one-line pointer in Notes to the new `sdd-document` skill as an optional next
  step after closing: updates `ARCHITECTURE.md` grounded in this run's per-phase touched files,
  best invoked right here while that data is still fresh in the session instead of re-derived
  standalone later. Not a precondition, no behavior change to this skill's own steps.
