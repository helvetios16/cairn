---
name: "sdd-judge"
description: "Runs a blind dual-judge adversarial review, using two independent CLIs launched via Herdr, in one of two modes picked automatically from the feature's real state: design mode (before any code exists, reviewing spec.md/plan.md/tasks.md for gaps and contradictions) or implementation mode (after sdd-implement, reviewing the actual code diff, before sdd-verify). Confirms a finding only when both judges report it independently, applies at most one bounded fix round, and writes a mode-named judgment report with a terminal APPROVED/ESCALATED verdict. Hard-requires an active Herdr session — there is no native-subagent fallback, since the whole point is cross-CLI model independence."
argument-hint: "Directory or folder name of the feature to judge (e.g. specs/002-sdd-gap-skills or 002-sdd-gap-skills). If omitted, resolved the same way as sdd-verify (prerequisites script)."
compatibility: "Requires an active Herdr session (server running, session inside Herdr) and a Spec Kit feature with either spec.md written (design mode) or a real implementation with at least one task marked [X] (implementation mode)"
metadata:
  status: experimental
  version: "0.2"
user-invocable: true
disable-model-invocation: false
---

## What this skill does

Adapts gentle-ai's `judgment-day` skill (see the discovery notes in memory) to this repo's own
multi-agent framework: an adversarial review where **two independent CLIs, blind to each other**,
inspect the same frozen target and a finding only counts as confirmed when **both** report it
independently. It runs in one of two modes, picked automatically from the feature's real state
(Step 1) — never asked of the user:

- **Design mode**: after `spec.md`/`plan.md`/`tasks.md` exist but before any task is implemented.
  Reviews the design itself — matching how Gentle's own `judgment-day` runs *before* codification,
  catching missing edge cases or contradictions between design artifacts while they're still cheap
  to fix, instead of letting a coding pass silently fill the gap (Gentle's own example: a route
  regex that quietly left `/favicon.ico` and internal build files exposed).
- **Implementation mode** (the skill's original v0.1 behavior, unchanged): after `sdd-implement`,
  before `sdd-verify`. Reviews the actual code diff for defects (correctness, edge cases, error
  handling, performance, security, project conventions) — a different question from what
  `sdd-verify` answers (does the implementation satisfy `spec.md`'s criteria).

Both modes share the same blind dual-judge mechanics (Steps 2-5) — only the frozen target, the
judge prompt's criteria, and the report filename change.

**Hard dependency on Herdr — no fallback.** Unlike the other multi-agent patterns in
`agent-selection`, this skill does **not** degrade to same-provider native subagents when Herdr
isn't active. The entire point of dual-judge here is model independence (two different providers,
different weights, uncorrelated blind spots) — a fallback to two native Claude Code subagents
would just be one model reviewing itself twice, which defeats the purpose rather than degrading it
gracefully. If Herdr isn't active, this skill's review simply doesn't run this round (see Step 0).

This is a review tool, not a delivery gate: an `APPROVED` verdict doesn't authorize a commit/push on
its own, and it doesn't block `sdd-verify`/`sdd-archive` — invoke it explicitly when you want the
extra adversarial pass.

## User Input

```text
$ARGUMENTS
```

Resolve which feature to judge the same way `sdd-verify` resolves `$ARGUMENTS`: if it carries the
feature's directory or folder name, use it as-is; if empty, resolve it via
`.specify/scripts/bash/check-prerequisites.sh --json` (Step 1).

## Step 0 — Herdr check (mandatory gate, no fallback)

Run `herdr status` first, exactly as `agent-selection` Step 0 describes, **and** confirm this same
session is running inside Herdr (`$HERDR_ENV` = 1 — see `agent-selection` Step 0 for the exact
check).

- **Both conditions hold** → continue to Step 1.
- **Either one fails** (server down, or this session isn't inside Herdr) → **do not run the
  review**. Report explicitly: "sdd-judge skipped this round — Herdr isn't active" and tell the
  user to proceed straight to `sdd-verify`. Do not start the Herdr server on your own initiative
  (same hard rule as `agent-selection` Step 0), and do not substitute native subagents as a judge
  pair — see *What this skill does* for why that substitution isn't offered here.

## Step 1 — resolve the feature, pick the mode, and freeze the target

1. Resolve `FEATURE_DIR` and `BRANCH` the same way `sdd-verify` Step 1 does (prerequisites script,
   without `--require-tasks`).
2. **Pick the mode from the feature's real state — never ask the user to choose:**
   - **Implementation mode** if `FEATURE_DIR/tasks.md` exists and has at least one task marked
     `[X]`. Takes priority over design mode: once code exists, the code diff is the sharper target.
   - **Design mode** if `FEATURE_DIR/spec.md` exists and implementation mode's condition isn't met
     (`tasks.md` missing, or exists with zero tasks marked `[X]`).
   - **Gate: nothing to judge yet.** If neither condition holds (no `spec.md`, and no implemented
     tasks), stop and report "nothing to judge yet" — same reasoning as `sdd-verify`'s equivalent
     gate. Do not generate a report in that case.
3. **Freeze the target**, once, as one immutable text block — per mode:
   - **Design mode**: the full content of `spec.md`, `plan.md` (if it exists), `tasks.md` (if it
     exists), and `data-model.md`/`contracts/` (if they exist), concatenated with a clear
     `--- <filename> ---` separator before each file.
   - **Implementation mode** (unchanged from v0.1): `git diff $(git merge-base <default-branch>
     HEAD)` — everything the feature branch introduced relative to where it diverged — plus `git
     diff` (uncommitted worktree changes) and `git status --short` for untracked new files, in case
     `sdd-implement` hasn't committed yet.

   This captured text is the target both judges see — identical bytes, captured once. Never
   recapture mid-review: if the design files or the working tree change between launching the two
   judges, one of them would be reviewing something the other never saw, breaking the "same
   target" guarantee the whole pattern depends on.

## Step 2 — launch the two blind judges via Herdr

Reuse `agent-selection`'s Step 3 (*How to operationalize Blind dual-judge so it's actually blind*)
and Step 4 (CLI/model table) verbatim for the actual Herdr mechanics (`tab create`, `agent start`,
sending the identical prompt to both before reading either response, etc.) — this skill doesn't
duplicate that mechanic, it inherits it.

**CLI pair — prefer real model independence.** Default to **Codex + opencode** for the two judges
(not Claude#2 for either seat): per `agent-selection`'s own note, a second Claude Code instance
gives process/context independence but not model independence, and model independence is
specifically what justifies requiring Herdr for this skill in the first place. If one of
Codex/opencode genuinely isn't usable this round, degrade to Codex + Claude#2 and say so
explicitly in the closing report — a reduced-independence pair, not a silent substitution.

**Judge prompt** (same, word for word, to both — criteria depend on the mode picked in Step 1):

Implementation mode (unchanged from v0.1):

```text
You are blind Judge {A|B} for sdd-judge (implementation mode).

Target: {frozen diff from Step 1}
Skills to load: {resolved skill paths, same for both judges}
Criteria: correctness, edge cases, error handling, performance, security, and project conventions.

Inspect only the given target. Do not edit, delegate, or look outside this diff. Return only a
list of findings — no prose, no fixes:

For each finding: location (path:line or path:start-end), severity (CRITICAL/HIGH/MEDIUM/LOW),
claim (the concrete incorrect behavior), and the evidence for it. If clean, say so explicitly.
```

Design mode:

```text
You are blind Judge {A|B} for sdd-judge (design mode).

Target: {frozen design artifacts from Step 1 — spec.md/plan.md/tasks.md/data-model.md/contracts}
Skills to load: {resolved skill paths, same for both judges}
Criteria: missing or ambiguous edge cases, internal contradictions between the design artifacts,
requirements that aren't objectively testable, scope drifting from proposal.md (if present), and
any gap a coding pass would have to silently fill in on its own.

Inspect only the given target. Do not edit, delegate, or look outside it. Return only a list of
findings — no prose, no fixes:

For each finding: location (filename § section, or filename:line for tasks.md), severity
(CRITICAL/HIGH/MEDIUM/LOW), claim (the concrete gap or contradiction), and the evidence for it
(quote the conflicting or ambiguous text). If clean, say so explicitly.
```

## Step 3 — merge into a ledger, confirm only by agreement

Wait for both judges to finish (never proceed on a single response — if one times out or fails,
report it explicitly as "a single opinion, not a complete blind dual-judge", same as
`agent-selection` Step 3 point 6, and decide with the user whether to relaunch it or stop here).

Merge the two responses:

- **Reported independently by both, same location/claim** → confirmed.
- **Reported by only one** → suspect — logged in the report, never auto-fixed.
- **Judges contradict each other on the same location** → contradiction — escalate to the user for
  an explicit decision, don't try to resolve it algorithmically.

Never launch a third agent to referee or refute a finding — the agreement between the two judges
is the corroboration mechanism itself, same explicit rule as gentle-ai's `judgment-day`.

## Step 4 — human confirmation before the first fix round

Before applying anything, stop and show the user the confirmed findings (with their evidence) and
ask for explicit go-ahead — same Principle III pattern `sdd-archive` uses for its own human
confirmation gate. Suspect and contradiction items are shown for context but aren't part of what's
being asked about here.

## Step 5 — bounded fix + one scoped re-judgment (2 rounds max)

If the user approves:

1. This session (the orchestrator) applies the fix — never the judges — one atomic unit per
   confirmed finding, no unrelated refactor, no new findings introduced along the way. In design
   mode the fix is an edit to the design artifact itself (`spec.md`/`plan.md`/`tasks.md`) that has
   the gap or contradiction, not to any code — there shouldn't be code yet. Same *surgical
   single-attempt correction* discipline `agent-selection` already documents: one attempt, not a
   retry loop.
2. Re-launch the same two judges, scoped only to the frozen ledger plus the fix's delta — re-diff
   only the files the fix touched (implementation mode) or recapture only the design files the fix
   touched (design mode) — not the original target again.
3. **Repeat Steps 4-5 once more at most.** Two total rounds (fix + re-judgment) is the hard cap —
   same as `judgment-day`. Any confirmed issue still open after the second round is escalated, not
   given a third attempt.

## Step 6 — write the judgment report and close

Write `FEATURE_DIR/judgment-report-<mode>.md` — `judgment-report-design.md` or
`judgment-report-implementation.md`, matching the mode picked in Step 1. The two modes get
separate files so a later implementation-mode run never overwrites an earlier design-mode
judgment (or vice versa) — both stay readable as part of the feature's history.

```markdown
# Judgment report — <feature>

Mode: design | implementation
Target: <branch> @ <short description of the frozen target, e.g. commit range / capture
  timestamp for implementation mode, or the list of design files captured for design mode>
Judges: <CLI A> + <CLI B>
Rounds: <1 or 2>

## Confirmed
| location | severity | claim | evidence | outcome |
|---|---|---|---|---|

## Suspect (single judge only)
| location | severity | claim | reported by |
|---|---|---|---|

## Contradictions
| location | judge A says | judge B says |
|---|---|---|

## Verdict: APPROVED | ESCALATED
```

Close every Herdr tab opened for this run, including any that never started properly — same tab
cleanup rule as `agent-selection` Step 6. Don't leave orphaned tabs in the user's workspace.

## Notes

- Complements `sdd-verify`, doesn't replace or gate it: this hunts for defects in the design or the
  code itself; `sdd-verify` checks the implementation against `spec.md`'s acceptance criteria.
  Recommended order is `sdd-propose` → `speckit-specify`/`plan`/`tasks` → **`sdd-judge` (design
  mode, optional)** → `sdd-implement` → **`sdd-judge` (implementation mode, optional)** →
  `sdd-verify` → `sdd-archive`. Both invocations of this skill are opt-in — neither is wired as a
  precondition into any other skill.
- Mode is picked from the feature's real state, not asked of the user (Step 1) — once a single
  task gets marked `[X]`, a later invocation automatically switches to implementation mode, even if
  the intent was to re-review the design. Re-reviewing a design after implementation has started
  isn't supported as a distinct mode yet — not a concrete need seen so far (Constitution Principle
  II); revisit if that case comes up in practice.
- No candidate-hashing or receipt infrastructure here, unlike gentle-ai's Go-based `judgment-day`:
  "freezing the target" means capturing a plain `git diff` once, not deriving a signed tree hash.
- The hard Herdr dependency (Step 0) is a deliberate design choice, not a gap to "fix" with a
  fallback later.
