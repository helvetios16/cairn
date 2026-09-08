# Changelog — agent-selection

History of skill changes, grouped into blocks of ~10 versions (one entry per block) — full detail only
in the most recent block, still forming (criterion: a closed block has already served its traceability
function, so there is no need to repeat it in full; recoverable line by line in this file's git history
if the detail is ever needed).

- **v0.1–v0.10 — foundational.** First version, based on Agent Harness Patterns and coordination tests
  with Herdr. Herdr check as mandatory Step 0. Fixed models per CLI (Opus 5 at the time / gpt-5.6-luna /
  DeepSeek V4 Flash Free / Gemini 3.6 Flash). Steps 1/2 realigned to the three-route scheme of
  gentle-ai's `trigger-rules.md`. Convention of launching in the root pane of a new tab, never with
  `pane split`. First real blind review (Codex+opencode): 5 convergent problems fixed plus a Codex
  invocation bug. Blind dual-judge operationalized. Step 0 extended to verify that the session is
  *inside* Herdr, not merely that the server is running. First 4R Lenses test with **Step 6 — midway
  failures** (timeout, Herdr down, degradation with partial results, tab cleanup).
- **v0.11–v0.20 — 4R Lenses closure and first Engram security round.** Findings from the Risk and
  Readability lenses applied (expanded risk list, write guardrails in Step 4, model tiering dedupe,
  Herdr/TUI jargon glossary). First real model tiering test. Changelog moved from `SKILL.md` to this
  file. Design and hardening of the Engram write barrier for external CLIs: `PATH` wrapper
  (`restricted-bin/engram`), later replaced by Codex's real sandbox (`-s workspace-write`) after finding
  that the `PATH` wrapper did not stop it. `TODO.md` created with that round's pending items. Claude Code
  moved from Opus 5 to Sonnet 5, with no model flag. Engram wrapper moved outside the repo tree (critical
  gap: a CLI with write permission could edit/delete it before attempting the bypass).
- **v0.21–v0.30 — Engram security round closed, then reverted.** Security `TODO.md` fully closed: real
  bypass tested and blocked (wrapper already outside the repo), "do not paste secrets into prompts"
  guardrail confirmed (`pane_history` off by default), Step 2 risk list confirmed **without real
  technical backing** (depends on the CLI respecting the prompt). Important finding: a native Task tool
  subagent had unrestricted write access to Engram — `.claude/agents/safe-reviewer.md`
  (`disallowedTools`) created as a real mitigation, verified live to block `mem_save`/`Bash`. Compared
  with gentle-ai (same fundamental subagent problem, no in-house solution). Equivalent custom agents
  tested and confirmed for Codex/opencode (real sandbox blocking the bypass), with no equivalent for
  Agy. **User decision that reverses nearly all of this round (v0.29)**: Engram is no longer registered
  as MCP in external CLIs — the benefit of having them search memory on their own was marginal versus
  the cost of maintaining this barrier; the wrapper and custom agents remain dormant in the filesystem
  in case this is reconsidered.
- **v0.31–v0.40 — first real multi-agent patterns, earpi feedback, and update to Herdr 0.8.0.** First
  complete multi-agent pattern actually run via `sdd-implement` (3 CLIs in parallel): finding that
  `agent send` does not submit Enter; `pane run` must be used. Real findings about Agy (per-action
  confirmation is its default behavior, not something provided by `--sandbox`). Three rounds of real
  earpi feedback closed (`revision` cannot detect pane changes, read-only `status` gotcha in zsh,
  insufficient timeout when the executor verifies its own work, ambiguity of "auth" in the risk list,
  project-level CLI roster), with later confirmation that the fixes worked in a real run. Investigation
  of `herdr --help` (0.7.4): `agent start` did not replace `tab create`+`pane run`, Agy's `agent_status`
  is a purely static fallback, `herdr worktree` confirmed as real isolation for parallel writes,
  `notification show` unusable, `agent rename` vs `pane`/`tab rename` are not interchangeable — plus
  official confirmation of Agy's fallback through Herdr docs. Real update from Herdr 0.7.4 → 0.8.0,
  with Step 4 rewritten for the new API (`agent start --kind/--pane`, `agent prompt --wait`,
  `agent send-keys`, `$HERDR_*` variables) and opencode finding corrected (it does have real lifecycle
  authority; the old note was outdated). Ends with an `SKILL.md` cleanup (495→393 lines) requested by
  the user and correction of 2 broken cross-references.
- **v0.41** — explicit user decision: Agy leaves the active roster, and a second Claude Code instance
  takes its place. Directly motivated by the v0.35-v0.39 findings (Agy state detection never worked,
  even with Herdr 0.8.0 and updated manifests). Before touching anything, the replacement's role was
  clarified, since Sonnet 5 is not a cheap model — the user chose: Claude Code enters as a fourth
  launchable option (executor/maximum-reliability second opinion), and opencode (already confirmed
  reliable and free in v0.39) takes the cheap model-tiering minion role left by Agy. Applied in
  `SKILL.md`: CLI table updated, Agy's two tactical notes removed (replaced by a short note explaining
  why it was removed, pointing to `TODO.md`/`CHANGELOG.md` for the full history), and a new note about
  Claude#2's limitation as a judge (provides context independence, not model independence — prefer
  Codex/opencode when that matters). Stray Agy mentions in Step 0/glossary/polling notes that no longer
  apply to the active roster also cleaned up. `TODO.md`: new section with the decision and its full
  reasoning.
- **v0.42** — real update from Herdr 0.8.0 → 0.8.2 (`brew upgrade herdr`, same procedure as
  v0.38: self-update disabled in Homebrew installations, server restart performed by the user outside
  this session, confirmed afterward with `herdr status` → `compatible: yes`, `restart_needed: no`; this
  skill's session continued without interruption). At the user's request, the repo's official
  `CHANGELOG.md` (`herdrdev/herdr`) was investigated (read directly via `raw.githubusercontent.com` to
  avoid the hallucinated summary returned by the initial fetch against GitHub's releases page — date and
  content did not match the raw text), and the two findings with direct impact on this skill were applied
  to `SKILL.md`, **without an independent live test** (unlike v0.38/v0.39, here the official changelog
  with an issue number is trusted instead of repeating the adversarial test):
  1. Native Claude Code confirmation prompts (`Enter to confirm · Esc to cancel`) now report
     `agent_status: blocked` instead of `idle` (bug fixed, issue #2268 in the official changelog) —
     relevant because Claude#2 (added as a judge in v0.41) may encounter this same type of prompt. Added
     as a `blocked` example in the `agent start` note of Step 4, alongside the existing Codex example.
  2. `agent send-keys`/`pane send-keys` now preserve Shift when sending `shift+tab` (it was previously
     lost, issue #1561) — allows cycling an agent's permission mode by command. Added as a short note
     after the `send-keys` paragraph in Step 4.
  Other changelog findings (race fix in `agent start` while waiting for the pane/shell to be ready,
  `#2410`/`#2537`/`#2773`/`#2774`; native OpenCode restore by root conversation, `#2450`) were
  evaluated and discarded for `SKILL.md`: they do not correct any active guidance in the file (the
  recommendation to check `agent_status` after `agent start` remains valid regardless of whether the
  race existed) nor change any existing reliability note (the opencode note already documented its
  detection as reliable since v0.38; restore is a separate topic). `TODO.md`: new section with the full
  detail and discarded items, explicitly noting that live verification is still needed if Claude Code's
  fix is ever to be confirmed with a real test (relaunch Claude#2, force a confirmation prompt, read
  `agent_status`).
- **v0.43** — explicit user request: add Herdr's official skill (`herdr --skill`) as its own skill inside
  this repo, and have `agent-selection` reference it instead of instructing users to run it live each
  time. `.claude/skills/herdr/SKILL.md` created with the verbatim output of `herdr --skill` (it already
  includes valid frontmatter — its own `name: herdr`, `description` — ready to load as a skill without
  editing), plus a manually added `metadata` block (not supplied by the binary) with Herdr's version at
  capture time (0.8.2) and an explicit note that the file does not auto-update: it must be regenerated
  by running `herdr --skill > .claude/skills/herdr/SKILL.md` again after each upgrade, because the
  `herdr --skill` output contains no version number inside it that would allow the drift to be detected
  on its own. Applied in `SKILL.md`: the list of "external sources" (preamble) now includes the
  `herdr` skill, and Step 4 replaces the instruction to run `herdr --skill` standalone with a pointer to
  the local file, with the same warning about possible staleness.
- **v0.44** — explicit user request after v0.43 (chosen option from 2 proposals: "option 1, you create
  a script that regenerates the skill here, and then it is copied manually to other places where needed,
  without much trouble"): the manually added `metadata` in `.claude/skills/herdr/SKILL.md` was replaced
  by `.claude/skills/herdr/regenerate.sh`, a script that runs `herdr --skill` and `herdr --version` live
  and builds the frontmatter with the actual version (prevents it from becoming mistyped or stale through
  forgetfulness). Tested: `git diff` against the file manually written in v0.43 was empty — the script
  reproduces exactly the same result, while also being repeatable for the next Herdr upgrade. Applied in
  `SKILL.md` (Step 4): the instruction to regenerate the file now points to the script instead of the
  standalone `herdr --skill > ...` command, and adds a concrete signal for when to run it — if
  `metadata.captured_from_herdr_version` in the local file does not match the version reported by
  `herdr status` in Step 0. Deliberately out of scope (explicit user decision): copying this script/skill
  to other repos was not automated — it remains manual, "without much trouble".
- **v0.45** — explicit user request: close v0.42's pending item live (Claude Code fix #2268) and also
  test the other 0.8.2 finding (Shift in `shift+tab`), in that order. A second real Claude Code instance
  was launched through Herdr (`tab create` + `agent start --kind claude`). **Unexpected finding**: it
  stayed in "auto mode" by default, which approves through a classifier — the first write attempt
  triggered no dialog, invalidating the first test. Resolved by taking it out of auto mode with
  `agent send-keys ... shift+tab` before retrying (this also tested the second finding). With that, a
  second write request did trigger the real dialog and `agent_status` returned `blocked` (confirmed with
  `agent explain --json`) — **#2268 fix confirmed live**, with a nuance: the block was detected by the
  generic `legacy_no_prompt_blocker` fallback, not a dedicated rule (the specific rules look for "do you
  want to proceed?", a Bash phrase; the actual Write tool dialog says "Do you want to create
  `<archivo>`?", different text with no dedicated rule yet) — the final result is correct anyway and
  does not change any `SKILL.md` recommendation. Shift in `shift+tab` confirmed with two real, distinct
  mode transitions (auto→manual, manual→accept edits), read from the pane, not inferred. Cleanup:
  test tab closed, 2 temporary `/tmp` files deleted. `TODO.md`: the v0.42 pending item changed from
  `[ ]` to `[x]` with full detail; a new `[x]` item added for the `shift+tab` finding. No content changes
  in `SKILL.md` — neither test found anything that corrected the guidance already written in v0.42.
- **v0.46** — explicit user request: remove `status: experimental` from the frontmatter `metadata`,
  leaving only `version`. Implicit reason: after 46 versions, a complete versioned history in this file,
  and a live verification round just closed (v0.45), the "experimental" label no longer describes the
  skill's actual state. The frontmatter line and the equivalent prose mention ("Experimental skill,
  continuously adjusted" → "Skill continuously adjusted, versioned in `CHANGELOG.md`", first paragraph
  of "What this skill does") were removed — "continuously adjusted" remains because that is still true;
  only the "experimental" qualification was removed. The unrelated mention in Herdr's guardrails line
  ("`[experimental] pane_history`" — a Herdr config flag, not this skill's status) was not touched.
- **v0.47** — explicit user request: reduce this file's size, which had reached 438 lines / 46 versions
  with full detail from v0.12 onward (only v0.1-v0.11 had been compressed, one per line, since the v0.12
  cut). Scheme chosen by the user (from 3 proposals): group ~10-version blocks, one summary entry per
  block — not one line per version as in the previous scheme. v0.1-v0.10, v0.11-v0.20, v0.21-v0.30,
  and v0.31-v0.40 compressed into four blocks (replacing 40 individual entries, including the 11 that
  were already in one-line format). v0.41 onward remains in full detail untouched — it is the most recent
  decade, still forming, with the work from this same session (Herdr 0.8.2, `herdr` skill, live
  verification). Result: 438 → 140 lines (~68% less). The full detail of the compressed versions remains
  recoverable line by line in this file's git history — no information was lost, only the prose was
  summarized. The preamble criterion was also updated to describe the new block scheme.
- **v0.48** — live finding, user-initiated: confirmed that `agent start` types the launch command into
  the pane's real interactive shell, so it lands in `~/.zsh_history` once that pane's shell exits (not
  immediately — the shell hasn't returned to its prompt while the agent runs in the foreground, so an
  early check can look clean when it isn't). Tested live: without a workaround, `codex -m gpt-5.6-luna
  ...` surfaced in history only after `tab close`; with `herdr pane run <pane_id> "unset HISTFILE"` run
  first, the same launch command and a follow-up prompt did not surface after the pane's shell exited.
  Added as a mandatory step before `agent start` in the launch sequence (Step 4). Scoped explicitly to
  the shell history file only — doesn't hide the command from the pane's visible content, `ps`, or
  `[experimental] pane_history` if that flag is ever enabled.
- **v0.49** — explicit user request: whenever Agy is launched, always pass `--dangerously-skip-permissions`
  (`--kind agy -- --dangerously-skip-permissions`), the Agy equivalent of Claude Code's bypass-permissions
  mode or opencode's build mode. Added as a note next to the existing "Agy removed from active roster
  (v0.40)" entry rather than restoring Agy to the fixed-models table — the v0.40 removal was about
  `agent_status` reliability, a separate concern this flag doesn't resolve on its own, and no live retest
  was done in this session. Flagged as worth retesting: part of the v0.40 unreliability came from Agy
  idling at unresolved trust/confirmation prompts, which this flag may sidestep entirely — left as an
  open question, not a re-verified fact. User first asked for this as a `~/.zshrc` alias
  (`alias agy='command agy --dangerously-skip-permissions'`); redirected to this skill instead once
  clarified that the intent was Herdr-mediated launches, not the user's own interactive shell — the
  `.zshrc` edit was reverted.
