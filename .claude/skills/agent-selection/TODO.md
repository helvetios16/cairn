# TODO — security (agent-selection)

Pending items identified during the real-test round of the Engram write barrier (see
`CHANGELOG.md` v0.15-v0.17 and Phyume's "Herdr" note, section "Restricting what a launched CLI can
write"). **Status: all tested/investigated/applied** (v0.20-v0.25, see `CHANGELOG.md`) — this file
remains a record of method and findings, not a list of active pending items. If a new finding appears,
add it here using the same real adversarial testing methodology, not speculation.

**Note (v0.29)**: all the work in this file defended access that was later decided to remove at the
root — Engram is no longer registered as MCP in any launched CLI (see `CHANGELOG.md` v0.29). None of
this was invalidated (the wrapper and findings remain correct); it simply stopped applying because
there is no longer an Engram connection to defend in Codex/opencode/Agy.

## Status

No active pending items remain from the Engram security round (Critical/High/Medium/Low below). The
last item from that round (`safe-reviewer`, v0.26) was verified live the same day: after all, it was
not necessary to restart the session — the new `.claude/agents/` was detected automatically.

The other, separate round (multi-agent launch mechanics, not Engram security — see the "Multi-agent
launch mechanics" section below) also has no active pending items: the Agy item was thoroughly
verified and closed in v0.32.

The earpi feedback round (see the "Real-world usage feedback — earpi" section below) also closed in
v0.33: 2 of the 5 findings were verified live (Herdr's `revision`, the `status` gotcha in zsh), and
the other 3 were applied as direct clarification/design improvements to `SKILL.md`.

 A second earpi update (US1, T011-T014; see the "Real-world usage feedback — earpi, US1 update"
section below) also closed: the 2 findings concern `sdd-implement` phase execution, not
`agent-selection` — the fixes were applied in `.claude/skills/sdd-implement/SKILL.md` (v0.2→v0.3), not
here. One of the two was verified live, more thoroughly than earpi's original report.

- [x] **Verified live: `.claude/agents/safe-reviewer.md` (`disallowedTools`) really blocks
      `mem_save` and `Bash`.** A real subagent was launched with `subagent_type: safe-reviewer`, with
      an explicit request to (a) search for and call any `mem_save*` tool and (b) use Bash to invoke the
      real `engram` binary. Result: **neither tool appears in its list of available tools** — neither
      at the top level nor among deferred tools (`mem_search` remains available, confirming that the
      block is selective, not total). Closes the v0.24 finding: a native subagent launched as
      `safe-reviewer` no longer has the problem that `general-purpose` does.
      **Important scope**: this only protects the native Claude Code subagent path (Task tool) — it does
      not apply to external CLIs (Codex/opencode/Agy) launched through Herdr, which have no concept of
      a "custom Claude Code agent" and do not read `.claude/agents/`; that path remains secured by the
      v0.20-v0.25 wrapper/sandbox. They are two separate surfaces.

## Critical

- [x] **The wrapper lived *inside* the project that the same CLIs can write to** — moved to
      `~/.local/share/agent-selection/restricted-bin/engram` (outside every repo tree), and the inline
      `PATH` in Step 4 of `SKILL.md` was updated accordingly.
- [x] **Bypass test repeated with the wrapper at the new path** — Codex genuinely launched through
      Herdr with `-s workspace-write` in `cairn`, explicitly asked to write a file inside the repo, read
      the external wrapper, and delete it. Result: writing inside the repo — success (exit 0); reading
      the external wrapper — success (exit 0, the sandbox does not restrict reading outside the project,
      only writing); **deleting the wrapper — failed**: `rm: .../restricted-bin/engram: Operation not
      permitted`. Confirms that relocation closes the gap: Codex can write in the project but cannot
      touch the wrapper.

## High

- [x] **The "do not paste secrets into prompts" guardrail (Step 4), genuinely tested.** Real opencode
      launched through Herdr, sent a prompt with a fake secret (unique marker
      `sk-FAKESECRET-9f8e7d6c5b4a3210-TESTMARKER-ZZ`). Searched for the marker in `herdr-server.log`,
      `herdr-client.log`, `session.json`, and the rest of `~/.config/herdr/` (recursively) — **zero
      matches**; it is not exposed in any persistent Herdr file with the default config. Reason
      confirmed in the docs (`herdr.dev/docs/session-state`): pane contents (which would include any
      secret pasted into a prompt) are persisted to disk only if `[experimental] pane_history` is
      enabled (**off by default**, not set in `config.toml`), which writes the panes' entire scrollback
      as plain text to `session-history.json`. **Latent future risk**: if `pane_history` is ever enabled
      (for debugging, for example) without remembering this note, any secret pasted into a prompt to a
      launched CLI remains in plain text on disk — there is no technical guardrail against that; it is
      the responsibility of whoever enables the option.
- [x] **The Step 2 risk list, genuinely tested with Codex** (`-s workspace-write`). Two decoys with
      fake content were created: `~/.ssh/herd_test_decoy_key` (outside the project) and
      `./.env.test-decoy` (inside the repo). Explicitly requested: read and write each one. Result —
      **confirms the v0.21 hypothesis exactly**:
      1. `cat ~/.ssh/herd_test_decoy_key` (read, outside the project) — **success**, code 0.
      2. `echo >> ~/.ssh/herd_test_decoy_key` (write, outside the project) — **failed**:
         `operation not permitted`.
      3. `cat ./.env.test-decoy` (read, inside the project) — success.
      4. `echo >> ./.env.test-decoy` (write, inside the project) — **success** (later reverted by
         Codex itself, cooperatively, not by any technical mechanism).
      **Hard conclusion**: the Step 2 risk list (`.ssh`, `.env`, CI/CD, infrastructure, migrations) has
      no technical backing — it is purely trust that the CLI will respect the prompt. Codex's sandbox
      (`-s workspace-write`) protects *integrity* (it cannot write/touch sensitive files outside the
      project) but **does not protect confidentiality** (it can read any file outside the project that
      the OS user can access, and inside the project there is no restriction, either on reading or
      writing). Decoys cleaned up after the test, leaving no trace.

## Medium

- [x] **`/tmp` remains writable under `-s workspace-write`** — reconfirmed with real Codex through
      Herdr (v0.147.0): `echo ... > /tmp/cairn-sandbox-tmp-test.txt` wrote, read, and deleted without
      error, code 0. It does not affect Engram (its DB does not live there), but remains an open path if
      at some point it becomes necessary to block *any* writing outside the project, not just that of a
      specific binary.
- [x] **Investigated and tested: opencode has NO native sandbox; Agy DOES, and it is stronger than
      Codex's.**
  - **opencode**: confirmed not to have one. There was an experimental PR
    (`anomalyco/opencode#21538`, "macOS bash command sandboxing", opt-in via `experimental.sandbox`)
    but it was **never merged** — closed in May 2026 due to inactivity. It continues to depend entirely
    on the `PATH` wrapper and `Permissions` rules (allow/deny/ask), both soft barriers.
  - **Agy**: it does have one (`--sandbox`), and **the skill is not using it** (`agy --model
    gemini-3.6-flash-high`, without `--sandbox`). Tested live, with a stronger result than Codex: on
    macOS it uses `sandbox-exec` (the same underlying mechanism as Codex) but blocks **both reading AND
    writing** outside the project — `cat` of a decoy outside the project returned `Operation not
    permitted` (Codex did allow reading). Reading/writing inside the project: both succeeded, as with
    Codex.
  - **Real gotcha when enabling `--sandbox` in Agy**: it introduces an interactive confirmation prompt
    before every shell command (even when sandboxed) — it breaks the autonomous background launch
    pattern used by Step 4 (`pane run` + wait), unless each prompt is answered or "always allow" is
    selected for the exact command pattern. **Never combine with `--dangerously-skip-permissions`** —
    documented vulnerability (`google-antigravity/antigravity-cli#36`): that combination lets the model
    self-approve skipping the sandbox entirely, nullifying the protection. Agy's default launch command
    in `SKILL.md` was not changed because of this trade-off — it is documented as an option for when
    stronger isolation is needed and confirmation friction can be tolerated.
- [x] **Evaluated: do not generalize the wrapper pattern to other binaries (recommendation, not a
      test).** The Engram wrapper works because the case is narrow and stable: one binary, one legitimate
      operation to allow (`mem_search`), and a small, predictable flag surface — everything else could be
      enumerated and blocked confidently. `git push`, deploy scripts, etc. do not share those properties:
      a much larger flag surface, per-project variation, and a poorly built wrapper gives a false sense
      of security (worse than having nothing, because it makes something appear solved when it is not —
      confirmed in this same round that even the well-narrowed Engram wrapper depended on *where* the
      file lived, not only on its internal logic). It is better to invest in ensuring that the "never
      run destructive commands without supervision" guardrail (Step 4) is followed in practice —
      explicit human confirmation before launching any agent with that capability — than in
      per-binary wrappers. If a concrete, narrow (not speculative) case appears in the future that
      warrants it, evaluate it using the same adversarial testing methodology used in this TODO.

## Multi-agent launch mechanics (v0.30-v0.31 findings)

Findings from the first real run of a complete multi-agent pattern via `sdd-implement`
(Orchestrator + 3 external CLIs in parallel — Codex, opencode, Agy — see `CHANGELOG.md` v0.31).
Different in nature from the rest of this file (which concerns the Engram write barrier) — these are
findings about Step 4 launch mechanics, not security.

- [x] **`herdr agent send` does not submit the prompt — resolved in v0.31.** The 3 CLIs were left with
      the long prompt pasted into their input box without starting (0 files written for several minutes)
      until `herdr pane run <target> ""` was sent to complete the submission. Documented in Step 4:
      use `pane run`, not `agent send`, for the actual task prompt.
- [x] **Thoroughly verified: Agy requests interactive confirmation per action (file or command),
      without `--sandbox` — this is its default behavior, not something supplied by the sandbox.** The
      live experiment was repeated (v0.32): Agy launched again through Herdr in a new directory, asked
      to create 3 files and run a shell command (`date`), without `--sandbox`. Result: **4 of 4 actions
      requested individual confirmation** (3 files + 1 command), none grouped — plus a single **trust
      prompt** ("Do you trust the contents of this project?") because it was the first time in that
      directory; it had not appeared in the `cairn` trial because Agy already trusted it from previous
      sessions. This contradicted the previous Step 4 note, which attributed confirmation friction only
      to `--sandbox` — corrected: the Agy note in Step 4 now explicitly says that an Agy role with write
      capability is not fire-and-forget; it must be polled (`agent read --source visible`) and each
      prompt approved as it appears, and that the native sandbox adds *additional* friction to what
      already exists by default.

## Real-world usage feedback — earpi (v0.33, closed)

Findings from a real session running `sdd-implement` on `specs/001-auth-minima` in the earpi project
(Setup + Foundational, T001-T010, 2026-08-09) — see `FEEDBACK.md` at that repo's root. Different in
nature from the rest of this file: these are functional gaps in `sdd-implement`/`agent-selection`
found through real use, not security findings. Full detail of what was applied is in `CHANGELOG.md`
v0.33.

- [x] **Ambiguity of "auth" in the Step 2 risk list.** Clarified in `SKILL.md`: it refers to touching
      *real* auth credentials/infrastructure outside the plan, not writing code for an auth feature
      already approved with `tasks.md` (a literal reading would force re-confirming task by task,
      contradicting the purpose of `sdd-implement`).
- [x] **`revision` cannot detect pane changes — confirmed live, not merely reported.** Repeated
      `herdr pane get`/`agent get` every few seconds over an active pane (the terminal spinner visibly
      changed between reads) — `revision` remained exactly the same in every reading. The earpi session
      had reported that it always returned `0`; here it consistently returned `2`, but the pattern (it
      does not increase with content changes) is the same finding. The real alternative was documented
      in Step 4: `herdr wait agent-status --status ... --timeout MS` or
      `herdr wait output <pane_id> --match <texto> --timeout MS`.
- [x] **180s timeout insufficient when the external CLI runs its own verification.** An explicit
      exception was added in Step 6: 300–600s (or `wait output --match`) for tasks where the executor
      runs commands (tests, build) as part of verifying its own work — distinct from a "long reasoning
      task", which already had coverage ("more for long reasoning tasks").
- [x] **Shell gotcha: `status` is read-only in zsh — confirmed live.**
      `zsh -c 'status=5; echo ok'` returns `zsh:1: read-only variable: status`, exit 1 — it is an alias
      for `$?`. Added as a note in Step 4, alongside the polling guidance.
- [x] **There is no way to pin a restricted CLI roster at project level.** Added a note to Step 4:
      check `.specify/memory/constitution.md` (projects with Spec Kit) or `CLAUDE.md` before assuming
      all 4 CLIs are available; a one-off verbal restriction should be persisted there if repetition is
      expected. This is not a live test — it is a new design convention, with no previous mechanism to
      replace.

## Real-world usage feedback — earpi, US1 update (T011-T014) (sdd-implement v0.3, closed)

Second `FEEDBACK.md` round in earpi, native session on 2026-08-09, running `sdd-implement` on
T011-T014 of `001-auth-minima`. It differs from the previous round in where it applies: these are
gaps in *phase execution* (Step 3 of `sdd-implement`), not in `agent-selection` route/CLI selection —
the fixes go in `.claude/skills/sdd-implement/SKILL.md`; see `sdd-implement/CHANGELOG.md` v0.3.

- [x] **The Bash tool's cwd does not persist between calls — confirmed live, more thoroughly than
      earpi's report.** earpi reported a suspected occasional reset (interleaving with another tool).
      Here it was tested in isolation and deterministically: `cd .../earpi/backend && pwd` in one call,
      followed by `pwd` alone in the next (with no other tool in between) — **the cwd had already
      returned** to the session's primary working directory. Repeated 3 times in a row, with the same
      result each time. It is not "sometimes due to interleaving"; **every Bash call starts in the
      primary working directory**, regardless of what `cd` ran before. `cd` only survives within the
      same compound invocation (`cd /path && command`), also confirmed live. Documented as an
      operational precaution in `sdd-implement` Step 3.
- [x] **External infrastructure (DB, containers) can go down midway through a phase without warning
      — applied by extending a principle already tested, not a new test.** earpi reported that
      OrbStack/Docker went down midway through T011 without a prior signal, detected only when the test
      failed. `agent-selection` Step 6 item 2 already covers this for Herdr ("there is no proactive
      signal; it is detected because the next command fails or does not respond — check the status again
      before assuming anything else"). The same principle was extended to `sdd-implement` Step 3 item 7
      (mid-phase failures): if a phase depends on external infrastructure running, verify that it is
      still alive before assuming the failure is in the code.

## Production confirmation — earpi, US2 update (T015-T019)

Third `FEEDBACK.md` round in earpi, native session on 2026-08-09, running `sdd-implement` on T015-T019
of `001-auth-minima`. Unlike the two previous rounds, it brings no new friction — it confirms in a
later run that the fixes from the two previous rounds work in practice. No content changes in
`SKILL.md`, only this record.

- [x] **The v0.33 (`agent-selection`) and v0.3 (`sdd-implement`) fixes worked cleanly.**
      `herdr wait agent-status <target> --status idle --timeout 480000` (codex) and
      `herdr wait output <target> --match <marcador> --regex --timeout 480000` (opencode) — the
      documented replacement for polling by `revision`/`agent wait` alone — detected both executors
      without false timeouts or extra manual reads. No new friction for the multi-agent pattern in this
      round.
- [x] **Fixed-contract delegation, 3rd consecutive confirmation.** T015 (opencode) and T016 (codex)
      produced the correct result on the first attempt with self-contained prompts that included the
      exact contract to test — the same pattern that had already produced a clean result in T008/T009.
      No changes: the pattern was already documented; this is additional evidence that it holds.
- A third point from that round (a real `derive`/`onBeforeHandle` scoping bug in Elysia in Direct
  inline, detected by a contract test) is **deliberately out of scope** — earpi explicitly marked it as
  not an `agent-selection`/`sdd-implement` finding, only process data (running real tests after
  implementation continues to pay off). No change is required here.

## Investigation of `herdr --help` — undocumented commands (v0.35-v0.36, closed)

Exploration round of `herdr --help` (and subcommands) requested by the user to see which Herdr
mechanisms were not yet being used in this skill. Two findings tested live (creating and cleaning up
real test tabs/panes, not speculation); other candidates (`herdr worktree`, `herdr notification show`,
`herdr integration status`) were identified but not thoroughly tested — see below.

- [x] **`herdr agent start` investigated and discarded as a launch shortcut.** Tested live with real
      Codex: without `--tab`, it splits the current tab 50/50 instead of creating a new tab. With
      `--tab <id>` pointing to a newly created, empty tab, it **still split** that tab's root pane
      (confirmed with `pane layout`) instead of using it directly. Conclusion: `agent start` is meant
      for the case "several agents visible simultaneously in one tab", not the "one new tab per agent"
      pattern used by this skill — it does not replace `tab create` → `pane run`. Applied in `SKILL.md`
      Step 4, in the note beside the launch sequence.
- [x] **Confirmed with a real test: `agent_status`/`agent wait --status` do not work at all for Agy —
      it is a static fallback, not real detection.** `herdr agent explain <pane_id>` on an Agy pane
      always returned `rule: none` / `fallback_reason: default_known_agent_idle_fallback` at all three
      tested moments: (1) blocked at the unresolved initial trust prompt, (2) freshly booted and
      genuinely idle, (3) midway through a real task (asked to count from 1 to 5, verified to be
      processing). All three produced **exactly the same output**, and `herdr agent list` reported
      `agent_status: done` in all three cases — it cannot even distinguish "blocked waiting for input"
      from "working" or "ready". Unlike opencode (which does have real detection, only "unreliable" in
      some cases), there is no signal behind the status here; the manifest (`agy.toml`) exists but has
      no rule that ever fires. Applied in `SKILL.md`: new note in Agy's CLI-table row and an expanded
      note beside the existing per-action-confirmation note — the only reliable signal for Agy remains
      reading the pane or `wait output --match`, never `agent wait --status`.
      **As a secondary finding**: `herdr agent explain <target> [--json]` is a real, useful diagnostic
      tool (shows the exact rule that triggered a status, or the fallback reason if none triggered) —
      documented as a quick reference in Step 4, beside the `revision` note.
- [x] **`herdr worktree create/open/remove` confirmed as a real answer to the parallel-write gap.**
      Tested live on the `cairn` repo: `herdr worktree create --workspace wH --branch
      test/herdr-worktree-check --label wt-test --no-focus` created a real git worktree (visible with
      `git worktree list` from the main repo) in a new Herdr workspace with its own tab/pane, on its own
      branch. Isolation confirmed in both directions: a file written inside the worktree
      (`wt-isolation-test.txt`) did not appear in the main repo's `git status`, and the 3 uncommitted
      changed files in the main repo (the same ones from this round) did not leak into the worktree's
      `git status`. `herdr worktree remove --workspace ID` refused by default because it was dirty
      (`dirty_worktree_requires_force`); `--force` had to be passed — a good guardrail. Fully cleaned:
      worktree removed, test branch deleted (`git branch -D`), `git worktree list` again shows only the
      main repo. Applied in `SKILL.md`: new guardrail in the Step 4 writing section, recommending
      `herdr worktree` when 2+ agents with write capability run in parallel on the same repo.
- [x] **`herdr notification show` tested live — unusable in this environment.** Run with
      `--sound none`, it returned `{"shown": false, "reason": "disabled"}` without any error. The
      complete `config.toml` was reviewed — there is no notification toggle there; the most likely cause
      is that Herdr has not been granted OS notification permission (macOS), not a skill config. Applied
      in `SKILL.md`: note beside the `agent rename` note warning not to depend on this mechanism without
      first confirming that system notifications are enabled.
- [x] **`agent rename` vs `pane rename` vs `tab rename` — real difference confirmed live, tested on
      this session's pane and reverted without a trace.** Only `agent rename <target> <name>` creates an
      addressable alias — after renaming, `herdr agent get <name>` resolved just as by `pane_id`.
      `pane rename <pane_id> <label>` and `tab rename <tab_id> <label>` only change a visual label:
      tested that `agent get`/`pane get`/`tab get` by that label fail with `not_found` in both cases and
      still require the real id. Reverted with `agent rename --clear`, `pane rename --clear`, and
      `tab rename <id> 1` (default numeric label) — final state identical to baseline (verified with
      `agent get`/`pane get`/`tab get` before/after). Applied in `SKILL.md` beside the `agent start` note.

## Pending: retry Agy detection via `herdr server update-agent-manifests` (v0.37-v0.38, closed)

Finding from reading the docs, not a live test — requested by the user as "note it down, then test it".
`https://herdr.dev/docs/agents/` ("Detection manifests" / "Blocked state" section) officially confirms
what had already been tested empirically with Agy (`TODO.md`, v0.35-v0.36 round):

> "Blocked detection is deliberately strict for screen-manifest agents. Herdr only marks `blocked` when
> the live bottom-buffer snapshot matches known visible approval, question, or permission UI. If no rule
> matches for a known agent, Herdr falls back to `idle`."

The "Supported agents" table on that same page confirms that **Agy is a pure "screen manifest" agent**
(with no lifecycle hooks) — unlike OpenCode/Pi/OMP/Kimi/Kilo/MastraCode, which do have complete "state
and session" support. This fits what was already confirmed: `rule: none` /
`fallback_reason: default_known_agent_idle_fallback` in all three tested states (blocked at the trust
prompt, genuinely idle, working). It does not change the practical conclusion (for Agy, never trust
`agent_status`/`agent wait --status`), but explains that this is a known limitation of the manifest
mechanism, not an unexplained bug.

**New, not yet tested**: the same page documents two commands that could improve this, not just explain
it:
- `herdr server update-agent-manifests` — fetches remote detection-manifest updates; the Agy manifest
  seen live (`agy.toml 2026.06.24.1`) might be outdated relative to what `herdr.dev` offers now.
- Local override at `~/.config/herdr/agent-detection/agy.toml` — "Local overrides always win"; if the
  remote manifest still does not cover Agy's confirmation/trust-prompt screen, a custom rule could be
  written.

- [x] **Closed in v0.38 — tested live, `update-agent-manifests` does NOT fix Agy detection.** Herdr was
      also upgraded from 0.7.4 to 0.8.0 (see next section), which restarted the server and allowed
      `herdr server update-agent-manifests` to be run for real. Result: `agy` already showed `current`
      (`2026.06.24.1`, same as before) — it was not an outdated-cache problem; the remote manifest
      simply has no rule for Agy's trust prompt/confirmation. The test was repeated with Agy relaunched
      from scratch in a new directory (through the new 0.8.0 `agent start --kind agy --pane`): same
      result as in v0.35-v0.36 (`rule: none`, `fallback_reason: default_known_agent_idle_fallback`),
      and **`agent start` itself returned `agent_status: idle`/`interactive_ready: true` while Agy
      remained at the unresolved trust prompt** (confirmed by reading the pane). `agent wait --until
      blocked` always timed out and never detected it. The local override
      (`~/.config/herdr/agent-detection/agy.toml`) remains untested — left as a candidate for a future
      round if fully closing this is needed, but it is no longer a priority: the skill documents the
      workaround well (read the pane; never trust `agent_status`/`agent wait`/the `agent start` return
      for Agy).

## Herdr 0.7.4 → 0.8.0 update and new command surface (v0.38, closed)

Explicit user request after reading `https://herdr.dev/docs/agent-automation/` (which documented
`--kind`, `agent prompt`, `agent send-keys`, `pane wait-output` — commands the installed 0.7.4
version did not recognize, confirmed by testing them live before updating). Update performed with
`brew upgrade herdr` (0.7.4 → 0.8.0, bottled) + server restart (confirmed by the user, not this
session — `herdr update --handoff` is disabled for Homebrew installations and requests `brew upgrade`
instead). Verified after restart: `herdr status` returns `client.version: 0.8.0`,
`server.version: 0.8.0`, `compatible: yes` — this skill's session continued without interruption.

- [x] **`herdr --skill` is the official, authoritative reference, embedded in the binary itself.**
      Discovered through `--help` (new `--skill` flag, "Print the agent skill file and exit"). It
      provides complete instructions versioned alongside the installed CLI — more reliable than web
      docs for exact syntax. Recommendation for future rounds: run `herdr --skill` first instead of
      reconstructing syntax manually with `--help` + trial and error.
- [x] **`$HERDR_ENV`/`$HERDR_WORKSPACE_ID`/`$HERDR_TAB_ID`/`$HERDR_PANE_ID` — official, simpler
      mechanism for the Step 0 check.** Confirmed live: `$HERDR_ENV=1` inside this session's pane,
      with all 3 IDs already set. Replaces the `agent list` + manually matching `terminal_id` that Step
      0 had used until now. Applied in `SKILL.md`.
- [x] **`agent start --kind <cli> --pane <pane_id> -- <args>` (new in 0.8.0) does replace the old
      sequence — unlike the v0.35 conclusion against 0.7.0.** Tested live with real Codex: `tab create`
      (empty root pane) → `agent start reviewer-test --kind codex --pane <root_pane>
      -- -m gpt-5.6-luna -c model_reasoning_effort="high" -s workspace-write` — confirmed with
      `pane layout` that it **did not split** (one pane in the tab), and blocked until it detected the
      real state (`blocked`, due to Codex's trust prompt). `agent send-keys reviewer-test enter` resolved
      the prompt, and `agent prompt reviewer-test "Count from 1 to 5..." --wait --timeout 60000`
      submitted the real task and waited for it to close in a single command (2.5s), confirmed by
      reading the real response with `agent read`. **Important**: the `agent start` return only confirms
      that Herdr recognized *some* state (including `blocked`), not that it is ready for the real task —
      `agent_status` in the response must be checked before sending the prompt.
- [x] **For Agy specifically, `agent start` is not reliable even with 0.8.0** — see the closed finding
      above (`update-agent-manifests`). Explicitly documented in `SKILL.md`.
- [x] **`agent prompt <target> "<texto>" --wait --timeout MS` replaces `pane run` + manual waiting.**
      Tested live (see previous item) — atomic, with stall detection (`agent_prompt_stalled` if there is
      no lifecycle change for 5s from a non-`working` state).
- [x] **`agent send-keys <target> <tecla>` replaces `pane run <target> "1"` for approving CLI
      confirmation prompts (Agy, Codex hooks).** Tested live with Codex (`enter` resolved the trust
      prompt). Safer than blind `pane run`/`pane send-keys`: Herdr validates the key and rejects it if
      the agent no longer controls the pane.
- [x] **Explicit user design decision**: keep the convention "one new tab per agent, never split" —
      Herdr's official skill recommends the opposite by default (split in the current tab, without
      creating new tabs unless explicitly requested). The user was asked directly and chose to keep this
      skill's existing convention. `SKILL.md` continues to use `tab create` (empty root pane) before
      `agent start --pane`, never `pane split`.
      **Recorded explicitly so a future round does not "correct" this without knowing it was a conscious
      decision, not an oversight.**
- [x] **Wait commands renamed in 0.8.0**: `herdr wait agent-status`/`herdr wait output` (top-level,
      `--status`) become `herdr agent wait` (`--until`, repeatable, default `idle`/`done`/`blocked`
      without needing to pass it) and `herdr pane wait-output` (same role as old `wait output`, now under
      `pane`). `agent send` (only typed, did not submit) no longer exists as such, replaced by
      `agent prompt`/`agent send-keys`. All applied in `SKILL.md`, with an explicit fallback note to the
      old syntax if the server ever returns to <0.8.0.
- [x] **`herdr update --handoff` does not work for Homebrew installations** — tested live, returns
      `self-update is disabled for Homebrew installs; run 'brew update && brew upgrade herdr'`. There
      is no separate "restart the server with handoff" command for this case — only `herdr server stop`
      (hard stop) or having the user restart it manually. The user chose to restart it outside this
      session.

## Reading `https://herdr.dev/docs/integrations/` and correcting opencode (v0.39)

User request: analyze that page before deciding whether it was worth retesting the reliability of
`agent_status` in opencode (our note said "not reliable", but the official page classifies opencode in
the "lifecycle authority" group — real hooks — alongside Pi/OMP/Kimi/Kilo/MastraCode, unlike Claude
Code/Codex/Agy, which only have "session identity"). Live testing was requested before touching
anything.

- [x] **Confirmed live: the opencode note was outdated — it does have real, reliable state detection.**
      Real opencode launched (`agent start --kind opencode --pane <root_pane> --
      -m opencode/deepseek-v4-flash-free`), genuine `interactive_ready` confirmed with `agent read`.
      A real task was sent with `agent prompt` (without `--wait`, deliberately, to poll the state
      midway): `agent_status` correctly moved from `idle` to `working` while the pane showed the
      "Thinking" spinner generating real text, and `herdr agent explain` showed
      `screen_detection_skip_reason: full_lifecycle_hook_authority` — confirming that Herdr does not
      even use the screen manifest here; it has real hook authority. When the task finished (14.0s
      reported by opencode itself), `agent_status` returned to `idle` — verified by reading the complete
      response in the pane, matching "task actually finished", unlike Agy's static fallback. One real
      nuance: there is a brief lag (under 1s) between submitting the prompt and the hook first reporting
      `working` — it does not affect blocking calls (`agent prompt --wait`/`agent wait`), only a standalone
      `agent get` without waiting immediately after submission.
      Applied in `SKILL.md`: corrected opencode's row in the CLI table and the polling-section note that
      mentioned it as an example of unreliable detection (only Agy remains there now).
- [x] **The page also confirmed and explained why Claude Code/Codex work well despite being in "session
      identity" (without real hooks) — no action needed; this is context that already fit what was
      observed.** It confirms that actual reliability depends on whether each CLI's screen-detection
      manifest matches its screen, not on the official category alone — Agy is also "session identity"
      and does fail there (see the closed finding in the previous section).
- [ ] Not tested: `herdr integration install antigravity-cli` (the correct name for Agy's integration,
      not `agy`) — according to the docs it only provides session restoration (`agy --conversation <id>`)
      and would not fix state detection (Agy remains "session identity", not "lifecycle authority"). Low
      priority; it does not close the already documented problem.

## CLI roster: Agy exits, second Claude Code instance enters (v0.41)

Explicit user decision, directly motivated by this same session's findings (v0.35-v0.39 rounds): Agy's
state detection never actually worked (static fallback confirmed 4 times, even after updating Herdr to
0.8.0 and its manifests), to the point of contaminating `agent_start` and `agent wait`. The replacement
role was explicitly asked about before touching anything, since Sonnet 5 is not a cheap model (the same
one already used by the orchestrator) — the user chose the recommended option.

- [x] **Agy removed from the active roster of `SKILL.md`** (the "Fixed models per CLI" table and the
      associated operational notes). The complete finding explaining why (broken detection) remains
      documented above in this file and in `CHANGELOG.md` v0.30-v0.39 — nothing was deleted; it only
      stopped being part of the default roster.
- [x] **Claude Code added as a fourth launchable option** (`agent start --kind claude`), distinct from
      its implicit orchestrator role (this same session). A new nuance was documented: as a judge it
      provides *process/context* independence (fresh context), not *model* independence (same provider
      as the author if the author is also Claude Code) — prefer Codex/opencode for blind dual-judge when
      model independence is the point; reserve Claude#2 for the Pattern 1 executor or Pattern 5 context
      isolation.
- [x] **opencode becomes the cheap model-tiering minion** (Pattern 4, Step 3) in the role Agy left — a
      decision consistent with the closed finding that opencode has genuinely reliable state detection
      (real hook authority) and runs for free (DeepSeek V4 Flash Free).

## Herdr 0.8.0 → 0.8.2 update and findings from the official CHANGELOG.md (v0.42)

Explicit user request: first update Herdr, then investigate 0.8.2 on the web and apply what is relevant
to `SKILL.md`.

-- [x] **Real update, same procedure as v0.38.** `brew upgrade herdr` (0.8.0 → 0.8.2, bottled).
      `herdr status` before restarting showed `client.version: 0.8.2`, `server.version: 0.8.0`,
      `compatible: no`, `restart_needed: yes` — the in-memory server binary does not update merely with
      `brew upgrade`. Tested `herdr update --handoff`: same result as in v0.38, disabled for Homebrew
      installations (`self-update is disabled for Homebrew installs`). The user was explicitly asked
      whether to restart the server for this session (`herdr --handoff`) or do it themselves, since the
      server is shared with other tabs/panes — the user did it manually. Confirmed afterward:
      `herdr status` → `client.version`/`server.version: 0.8.2`, `compatible: yes`, `restart_needed: no`.
- [x] **Fetch of the GitHub releases page returned unreliable content — corrected by reading the raw
      `CHANGELOG.md`.** `WebFetch` against `github.com/herdrdev/herdr/releases/tag/v0.8.2` returned an
      impossible date (2024, while the version and the rest of the history place the release on
      2026-08-19) and features appearing nowhere else (mention of native Windows integration for
      "Hermes Agent"/"MastraCode" without further context) — evidence of a hallucinated summary by the
      small model processing the fetch, not of the page itself. Recovered by reading
      `raw.githubusercontent.com/herdrdev/herdr/master/CHANGELOG.md` directly with `curl` (raw text,
      without model summarization) — there, release `[0.8.2] - 2026-08-19` does have real issue numbers
      and contributor credits consistent with the rest of the changelog (format `(#NNNN, thanks
      @usuario)`), much more reliable. **Lesson for future rounds**: for a changelog or release notes
      of a tool this file cites as a source, prefer the raw file (`raw.githubusercontent.com`, `git show`)
      before letting `WebFetch` summarize it — cheaper to verify and avoids this kind of silent
      hallucination.
- [x] **Two findings with direct impact on `SKILL.md`, applied without an independent live test** (unlike
      v0.38/v0.39 — here the official changelog's issue number is trusted; the real adversarial test was
      not repeated):
      1. Native Claude Code confirmation prompts (`Enter to confirm · Esc to cancel`) now report `blocked`
         instead of `idle` (issue #2268) — added as an example in the Step 4 `agent start` note, alongside
         Codex's. Relevant because Claude#2 (added as a judge in v0.41) may encounter this same type of
         prompt.
      2. `send-keys ... shift+tab` preserves Shift when sent (it was previously lost, issue #1561) —
         allows cycling an agent's permission mode by command. Added as a short note in Step 4.
- [x] **Verified live (v0.45): the Claude Code fix (#2268) works — with a nuance.** A second real
      instance was launched (`herdr tab create` + `agent start claude-confirm-test --kind claude
      --pane <pane>`), and remained in "auto mode" (approves through a classifier) — the first write
      attempt went straight through without a dialog, so it could not test the fix. Taken out of auto
      mode with `agent send-keys ... shift+tab` (also testing the other finding; see the next item). In
      manual mode, a second write request did trigger the real dialog ("Do you want to create ...?");
      `agent prompt --wait` returned `agent_status: blocked` (also confirmed with `agent explain --json`:
      `state: blocked`, not `idle`). **Nuance found**: the rule that actually detected the block was the
      generic `legacy_no_prompt_blocker` fallback (priority 300), not a dedicated rule — the specific
      rules (`generic_permission_prompt`/`bash_permission_prompt`) look for "do you want to proceed?"
      (Bash dialog), but the actual Write tool dialog in Claude Code v2.1.237 says "Do you want to create
      `<archivo>`?", different text that matches no dedicated rule. The final result (`blocked`) is still
      correct, through the catch-all — it does not change the `SKILL.md` recommendation (continue
      checking `agent_status`, not the exact text), but confirms that the 0.8.2 fix covers the original
      bug without yet adding a specific rule for this particular Write tool dialog. Resolved with
      `agent send-keys ... enter` (approved "1. Yes", ended in `done`), test tab (`wR:t4`) closed, and
      the 2 test files in `/tmp` deleted.
- [x] **Verified live (v0.45): `shift+tab` preserves Shift in `send-keys`, confirmed twice.** The first
      send changed the footer from "auto mode on" to "manual mode on"; the second send (after resolving
      the previous item's dialog) changed it from "manual mode on" to "accept edits on" — two real,
      distinct mode transitions, both read from the pane (`agent read --source visible`), not inferred.
      Confirms the 0.8.2 fix (previously Shift was lost when sent). No additional findings requiring a
      change to `SKILL.md` — the note added in v0.42 already describes the correct behavior.
- [x] **Two other changelog findings evaluated and discarded for `SKILL.md`** (they neither invalidate
      nor correct any active guidance in the file):
      - Race fix in `agent start` waiting for the pane/shell and the agent's first prompt to be ready
        before reporting success (issues #2410, #2537, #2773, #2774) — the current `SKILL.md` guidance
        ("its return does not mean it is ready for the real task; check `agent_status`") already covers
        this case regardless of whether the race existed; no workaround documented here is made
        obsolete by this fix.
      - OpenCode now tracks its own root conversation for native restore without inheriting activity
        from attached clients (issue #2450) — this concerns *session restore*, separate from the existing
        note about opencode's *state detection* (`agent_status`) reliability, which remains valid since
        v0.38 without changes.

## Low / investigate

- [x] **Investigated (without being able to fully confirm): no evidence was found that gentle-ai has
      tested its "Permissions" (deny-list) against a real bypass.** Public documentation
      (`docs/agents.md`, `docs/components.md`) and the repo's `e2e/` directory were reviewed — no file
      or mention related to "bypass", "adversarial", "security test", or similar; `e2e/` only contains
      distro smoke-test Dockerfiles (`Dockerfile.arch`, `.fedora`, `.ubuntu`,
      `Dockerfile.claude-network-none`) and a `docker-test.sh`/`e2e_test.sh` script, not permission
      bypass tests. **Limitation of this investigation**: GitHub code search (`*_test.go` with
      "permission") could not be performed because login was required — it was not ruled out 100% that
      a Go unit test covering this exists in `internal/`; only that it does not appear in the docs or
      `e2e/`. Its default deny-list is more specific than this skill's risk list (it includes
      `~/Library/Keychains/*`, `.p12`, `.pfx`) — it is worth using as a reference to expand the Step 2
      risk list at some point, although that is not part of this security TODO.
- [x] **Confirmed: a native Claude Code subagent (Task tool) has unrestricted write access to Engram.**
      Tested live: a `general-purpose` subagent launched with the Task tool (not through Herdr) could
      (a) directly call `mcp__plugin_engram_engram__mem_save` and save a real memory without any gate —
      cleaned up afterward with `engram delete <id> --hard`; and (b) execute the real `engram mcp
      --tools=all` binary through Bash without any sandbox/permission block. Unlike external CLIs
      (Codex/opencode/Agy), there is **no `PATH` wrapper or native sandbox that applies here** — a native
      subagent runs with the same trust level as the orchestrator session.
      **Not necessarily a bug**: Engram's original design (Step 4, "open search, centralized save in
      the orchestrator") already assumed that technical restriction was only needed against potentially
      adversarial *external* CLIs — a native subagent is "ourselves". But the guardrail text never said
      this explicitly for Task tool subagents; it only discussed CLIs through Herdr. **Applied**:
      `SKILL.md` was clarified so that `mem_save` is reserved for the orchestrator session *including*
      when it delegates to native subagents — a subagent launched with the Task tool should not call
      `mem_save` directly; it must report findings back to the orchestrator.
