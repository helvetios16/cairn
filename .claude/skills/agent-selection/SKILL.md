---
name: agent-selection
description: >
  Evaluates a work situation and recommends whether it's worth using a single agent or several
  coordinated ones, which orchestration pattern applies, and which CLI/model to assign to each role.
  Trigger: before delegating a task to an agent, when in doubt whether it's worth coordinating multiple
  agents (via Herdr or another mechanism), or when choosing which CLI/model to use for a subagent.
metadata:
  version: "0.49"
---

## What this skill does

Skill under continuous adjustment, versioned in `CHANGELOG.md`. Before launching more than one agent (or
choosing which CLI to use for one), evaluate the situation with the framework below and give an explicit
recommendation: single agent, or which multi-agent pattern + which CLI/model per role.

**External sources this file assumes are consistent, but neither versions nor can verify from here**:
`docs/trigger-rules.md` from [gentle-ai](https://github.com/Gentleman-Programming/gentle-ai) (Step 1), the
`herdr` skill (`.claude/skills/herdr/SKILL.md`, Step 4 — see note below) and the Phyume notes "Herdr" (Step
4) and "Agent Harness Patterns" (Step 3). If anything in this file contradicts what is observed live (a
CLI's banner, `herdr --help`, etc.), trust what's observed and update this file — not the other way around.

## Step 0 — check Herdr (mandatory, before anything else)

Run `herdr status` **first, always**, before evaluating anything else. It defines what options exist for
the rest of the analysis:

- **`server.status: running`** → cross-CLI coordination available (Codex, opencode, and a second instance
  of Claude Code). Continue with the full framework, Step 3 includes the options.
- **Server down, or `herdr` doesn't exist as a command** → no cross-CLI coordination. Only Claude Code's
  native subagent mechanism remains (Task/Agent tool), which runs subagents of the same model/provider.
  **Explicitly flag this limitation** before recommending anything — don't assume Herdr is available or
  propose Codex/opencode as an option if the check didn't confirm the server is running. Model tiering
  (cheap minion via opencode) is ruled out in this case.

**Hard rule: never start the Herdr server on your own initiative.** If `herdr status` shows the server
down, that's the answer — don't run `herdr server ...` or any command that would bring it up. Starting a
background server is a decision for the user, not something this skill triggers on its own. Report the
down state and proceed with the native-subagent fallback.

**Beyond the server, the current session has to be running *inside* Herdr.** `server.status: running`
being true isn't enough — this same Claude Code instance also has to be an agent Herdr recognizes (launched
as `herdr` or inside a pane Herdr manages), otherwise there's no own `workspace_id` from which to do
`tab create`. **Since Herdr v0.8.0, the official and simplest way to confirm this is an environment
variable check** (documented in `herdr --skill`, tested live — v0.37): `$HERDR_ENV` equals `1` inside a
pane managed by Herdr, and `$HERDR_WORKSPACE_ID`/`$HERDR_TAB_ID`/`$HERDR_PANE_ID` already carry this
session's own IDs without needing to run `agent list` and match `terminal_id` by hand.

```bash
test "${HERDR_ENV:-}" = 1   # if this fails, we're not inside a Herdr pane
echo "$HERDR_WORKSPACE_ID" "$HERDR_TAB_ID" "$HERDR_PANE_ID"   # own IDs, ready to use in Step 4
```

If `$HERDR_ENV` is not set (or has a value other than `1`) even though the server is running, treat it the
same as "Herdr not available" for the purposes of Step 3/4 — new tabs can't be created without an own
workspace. `herdr agent list` is still useful to inspect what else is running, but is no longer needed to
confirm the session's own identity.

This check is not optional. It always runs, even if Step 1 ends up resolving to Direct inline and the
result never gets used: it's a single command (`herdr status`), cheaper than discovering only in Step 4
that Herdr wasn't available.

## Step 1 — choose route (aligned with gentle-ai's `trigger-rules.md`)

Classify the task into one of three routes, from least to most delegation — **always use the lightest one
the situation truly requires**, don't jump to a heavier one just because it's "available":

**Before classifying by file count: check the *risk list* in Step 2.** If the file/change touches
something on that list, file count doesn't govern — it's never automatic Direct inline, no matter that
it's a single file. At minimum, treat it as Delegated direct with explicit user confirmation before
applying anything; if it's also an irreversible decision, go straight to question 2 of Step 2. This closes
the gap where a small but dangerous change (a deploy script, a permissions file, something touching auth)
could sneak in as Direct inline just for touching 1-3 files.

1. **Direct inline** — understanding or verifying the change requires **1-3 files**, or it's a mechanical
   change already understood, with no pending research or design decision, **and it doesn't touch anything
   on the risk list**. → continue in the current session, don't launch anything.
2. **Delegated direct** — understanding requires **4+ files**, the reading feeds into a write, broad
   research is needed, or a writer is going to touch **2+ non-trivial files**. → delegate that exploration
   or one-off write to **one** subagent (native to Claude Code, or via Herdr if the role warrants it) —
   still without assembling a multi-agent pattern.
3. **Multi-agent with pattern** — some question in Step 2 comes back yes. → a single delegation isn't
   enough; the specific pattern from Step 3 is needed.

File count describes the context the *current* action needs, not a risk score — risk can justify escalating
to route 3, but never forces a heavier route if the real context stays small.

## Step 2 — when to escalate from Delegated direct to a multi-agent pattern (route 3)

These questions **are not mutually exclusive** — more than one can come back yes at the same time for the
same task. If that happens, combine the corresponding patterns from Step 3 (e.g. an orchestrator that also
uses model tiering for its executors). If there's a real tension over which pattern to apply first,
prioritize **2 and 3 (verification/risk) over 1 and 4 (organization/cost)** — a security finding matters
more than saving on model cost.

1. Does the task have several **genuinely independent** phases/domains that require more than one
   delegation coordinated by a leader — not a single stretch of repetitive work (that's question 4, not
   this one)?
2. Does it touch something on the risk list (see below), or is it an irreversible decision (deletion,
   migration, secret/credential, shipping something to production) — to the point of needing a second
   opinion that doesn't see the first one's output?
3. Is there more than one relevant failure dimension *at the same time* (security, readability, test
   coverage, resilience) that a single review pass doesn't cover well — unlike question 2, here it's not
   about a second opinion on the same thing, but coverage of different angles?
4. Is part of the work mechanical/repetitive **and separable** (can be sent off to run on its own, without
   depending on frontier-model reasoning) — not just "there's something repetitive in the middle of a task
   that still has to be thought through as a whole"?
5. Has the current session already gotten long and risks compaction (silent loss of rules given at the
   start) if context keeps accumulating there?

**Objective criteria** (so 2/3/4 aren't left to free interpretation):
- *"Non-trivial file"* (delegation questions in Step 1 and here): changes logic or adds/removes a flow,
  not just text/formatting/rename.
- *Risk list* for question 2 (and for the prior check in Step 1): paths like `~/.ssh/*`, `*.pem`, `*.key`,
  `.env*`, `~/.aws/credentials`, `~/.config/gh/hosts.yml`; CI/CD configuration (`.github/workflows/*`,
  `.gitlab-ci.yml`, `Jenkinsfile`); infrastructure as code (Terraform, Kubernetes manifests, production
  `Dockerfile`/`docker-compose`); database migrations; any production/deploy configuration file; or
  auth/payments/mass data deletion. Outside that list, treat question 2 as "no" unless explicitly
  justified.
  **Clarification on "auth"**: this refers to touching *real* auth credentials/infrastructure outside an
  already-planned feature — not to writing the code for an auth feature that already has an approved
  `tasks.md` (read literally, it would force re-confirming an entire feature task by task, contradicting
  running `sdd-implement` over an already-approved plan).
- **This list is purely a prompting criterion, with no technical backing** — tested live with Codex
  (`-s workspace-write`): the sandbox protects *integrity* (it can't write outside the project) but not
  *confidentiality* (reading a file outside the project does work), and within the project it doesn't
  restrict anything. It only works if the launched CLI respects the prompt — nothing enforces it
  technically.

If none come back yes → **Delegated direct is enough** (route 2 of Step 1), no multi-agent pattern needed.

## Step 3 — question → pattern mapping

| If the answer to... | ...is yes, use | Roles |
|---|---|---|
| 1 | Orchestrator + specialized subagents | 1 leader (frontier) + N executors |
| 2 | Blind dual-judge | 2-3 independent evaluators, same criteria, without seeing each other |
| 3 | Parallel lenses (4R framework: Risk/Readability/Reliability/Resilience) | N evaluators, each with a different criterion |
| 4 | Model tiering | Orchestrator on a powerful model, mechanical execution on a cheap model (opencode/DeepSeek V4 Flash Free — see Step 4 table). Fallback if Herdr isn't active: see Step 4 |
| 5 | Context isolation (subagent in fresh context) | New subagent, without carrying over the current session's history |

**How to operationalize Blind dual-judge (pattern 2) so it's actually blind** (each judge's CLI/model: see
Step 4 table):

1. The instance running this skill (Claude Code) is the **orchestrator/fix-agent** — never one of the
   judges. If the object under review was written/edited by this same session, one judge has to be a
   different CLI so the review is independent of the author.
2. Launch the `tab create` + `agent start --kind --pane` for **both** judges before reading either one's
   response — don't wait for the first to finish before launching the second (that would already break
   "without seeing each other" if the second prompt is built by citing something from the first).
3. Send the **same prompt, word for word**, to both (`agent prompt <target> "..." --wait`).
4. Only read both responses once both have finished (`agent prompt --wait` already waits for each pane's
   settled state, with a timeout — see Step 6). Never feed one judge the other's response, nor summarize
   it to them.
5. The orchestrator filters which findings are valid (convergent = higher confidence; unique = evaluate
   case by case) and applies the fix — one attempt, not a generation loop (see *Surgical single-attempt
   correction* in Agent Harness Patterns).
6. **If a judge never responds (timeout or failure)**: "convergent vs. unique" stops making sense with a
   single response — don't pretend there was consensus. Report it explicitly as "a single opinion, not a
   complete blind dual-judge" and decide with the user whether that's enough or whether it's worth
   relaunching that judge (see Step 6, degradation with partial results).

## Step 4 — which CLI/model to use per role

Only applies if Step 0 confirmed Herdr is active. The `herdr` skill (`.claude/skills/herdr/SKILL.md`, in
this same repo) is a verbatim copy of `herdr --skill` and the authoritative reference for syntax —
`metadata.captured_from_herdr_version` in its frontmatter says which version it was generated against, but
the file doesn't auto-update. If anything here doesn't match what Herdr does live, or the `herdr status`
from Step 0 shows a different version than that metadata, run `.claude/skills/herdr/regenerate.sh`
(regenerates the file by capturing `herdr --skill` and `herdr --version` live, not by hand) before assuming
the local file is wrong, and update this file if it contradicts what's observed live.

**Quick glossary**: *pane* = an individual terminal; *pane root* = the full-screen pane that already comes
with a tab as soon as it's created; *alt-screen TUI* = an interface that redraws the whole screen instead
of scrolling normally — read with `--source visible`, not `recent` (can come back empty/stale in these
interfaces).

**Launch sequence — a new tab per agent, never a split** (design decision: Herdr's official skill
recommends splitting in the current tab by default, this was explicitly evaluated and this convention is
kept — see `TODO.md`/`CHANGELOG.md` v0.38 before "fixing" this without knowing it was intentional):

```bash
herdr tab create --workspace <ws_id> --label <agent-name> --no-focus
# .result.root_pane.pane_id → full-screen pane, at its interactive shell prompt
herdr pane run <pane_id> "unset HISTFILE"
herdr agent start <agent-name> --kind <claude|codex|opencode|agy|...> --pane <pane_id> -- <native CLI args>
```

**`pane run ... "unset HISTFILE"` before `agent start`, always.** `agent start` types the launch command
(with the fixed model/flags from the table below) into that pane's interactive shell — it's a real command
line, not something invisible to the shell. Confirmed live (v0.48): without this, the exact `codex -m
gpt-5.6-luna ...` invocation ends up in `~/.zsh_history` once that pane's shell exits (e.g. on `tab close`)
— it doesn't show up immediately (the shell hasn't returned to its prompt while the agent runs in the
foreground), which can make an early check look clean when it isn't; it only surfaces once that shell
actually terminates. With `HISTFILE` unset before `agent start` runs, that shell has nowhere to persist
history, so the launch command and anything the agent runs never reach disk. This doesn't hide it from the
pane's own visible content, from `ps`, or from `[experimental] pane_history` if that's ever turned on (see
the security guardrails below) — it only keeps it out of the shell's history file.

`agent start` blocks until it detects the agent in that pane (default 30s, `--timeout` 3000-300000ms) and
the name passed as the first argument becomes an immediately addressable alias. `herdr agent` (no
subcommand) lists the installed `--kind`s. The target pane has to be at its interactive shell prompt — Herdr
never creates, splits, or moves layout on its own, which is why `tab create` comes first.

**`agent start` returning doesn't mean "ready for the actual task"** — only that Herdr recognized some
state (`idle`, `working`, or `blocked`). Always check `.result.agent.agent_status`: if it comes back
`blocked` (a Codex confirmation hook, trust prompt, or Claude Code's native confirmation prompt — `Enter to
confirm · Esc to cancel`, which up through Herdr 0.8.1 was mistakenly reported as `idle`, fixed in 0.8.2 —
relevant for Claude#2 as a judge, see table below), resolve it with `agent send-keys` before continuing.

`agent rename <target> <name>` creates a truly addressable alias (usable afterward as `<target>` in any
`herdr agent ...`); `pane rename`/`tab rename` only change a visual label — tested that they do **not**
work for addressing (`get` by that label returns `not_found`). Use `agent rename` for scripting, the other
two only for human readability when reviewing the session.

`herdr notification show` is not usable in this environment — tested live, it returns `{"shown": false,
"reason": "disabled"}` with no toggle in `config.toml` that explains it (most likely: OS notification
permission not granted to Herdr). Don't rely on this to alert the user about a blocked agent.

**Sending the actual prompt — one command, types + submits + waits:**

```bash
herdr agent prompt <agent-name> "<task text>" --wait --timeout <ms>
```

Default wait: first settled state (`idle`/`done`/`blocked`, no need for `--until`). If there's no
lifecycle change in 5s from a non-`working` state, it returns `agent_prompt_stalled` instead of hanging —
a signal that the prompt wasn't actually submitted. When launching several agents in parallel, fire
`agent prompt --wait` at **all** of them before waiting for any one's response (same criterion as Blind
dual-judge, point 2 of Step 3 above).

To interact with an already-running agent's UI (approving confirmations, canceling) use `herdr agent
send-keys <target> <key>` (`esc`, `enter`, `ctrl+c`) — Herdr validates the key and rejects it if the agent
no longer controls the pane, safer than blindly using `pane run`/`pane send-keys`. Those `pane` commands
are still the right surface for ordinary non-agent processes (tests, builds). Since Herdr 0.8.2, `send-keys
... shift+tab` preserves Shift when sent (previously it was lost) — useful for cycling an agent's
permission mode (e.g. Claude Code) by command, without manual intervention.

**Polling and state waiting.** The `revision` field of `agent get`/`pane get` **doesn't work** for detecting
pane changes (confirmed live: it stays the same while the pane's content changes) — don't use it for manual
polling. Instead: `herdr agent wait <target> [--until <state>] [--timeout MS]` (blocks on state transition;
without `--until` uses the same default as `agent prompt --wait`) or `herdr pane wait-output <pane_id>
--match <text> [--regex] [--timeout MS]` (blocks on pane content — needed if some CLI in the roster doesn't
have reliable state detection; without `--timeout` waits indefinitely). `herdr agent explain <target>
[--json]` shows which rule triggered an `agent_status` (or whether it fell back) — faster than guessing.

**Shell gotcha (not Herdr's)**: in zsh, `status` is a read-only variable (alias for `$?`) — use another
name (`agent_status`, `estado`) in your own polling scripts.

**Security guardrails when launching agents with write capability:**

- **By default, read/review tasks** (judges, lenses, exploration) — not writing. If the specific role
  requires the launched CLI to *modify* files or run commands (not just read and report), that's a
  separate decision that requires **explicit user confirmation before launching it**, just like any other
  write action outside this skill — don't assume implicit permission just because the multi-agent pattern
  was already approved.
- **Never let a launched agent run destructive or production commands unsupervised** (deploys, migrations,
  mass deletions, infrastructure changes) — that always goes through the same human confirmation check it
  would if this session did it directly, with no shortcuts for being "delegated" to another CLI.
- **Don't paste secrets or credentials into the prompts** sent to the CLIs — if the task needs to reference
  a sensitive file (from the Step 2 risk list), describe it by path/name, don't copy its content. Sending
  the same context to 2-4 different CLIs (blind dual-judge, 4R lenses) multiplies the exposure surface if
  the prompt includes sensitive data. Tested live with a fake secret: with Herdr's default config it isn't
  exposed in any persistent file (`herdr-server.log`, `herdr-client.log`, `session.json`) — but if
  `[experimental] pane_history` is ever turned on (off by default), the panes' content, secrets included,
  ends up in plain text in `session-history.json`. Don't enable that option with this skill's agents
  running.
- **A fix that touches something on the risk list isn't applied on the orchestrator's approval alone**
  (even if it comes from blind dual-judge or any verification pattern) — confirm with the user before
  applying it. Adversarial verification reduces the risk of a logic error, but it isn't security
  authorization.
- **Two or more write-capable agents in parallel on the same repo → isolate each one in its own `herdr
  worktree`, don't launch them directly on the same checkout.** Confirmed live (v0.35, `herdr worktree
  create --workspace ID --branch NAME --label TEXT`): creates a real git worktree (not simulated — visible
  with `git worktree list` from the main repo) in a new Herdr workspace with its own tab/pane, on its own
  branch. Isolation tested in both directions: a file written inside the worktree doesn't show up in the
  main repo's `git status`, and uncommitted changes in the main repo don't leak into the worktree. `herdr
  worktree remove --workspace ID` refuses by default if anything is left uncommitted
  (`dirty_worktree_requires_force`, `--force` has to be passed) — a good guardrail against losing an
  agent's work during cleanup. Without this, 2+ agents writing on the same checkout compete for the same
  files with no technical guardrail — only human confirmation, which doesn't resolve the merge conflict.

**Shared memory (Engram) — exclusive to the orchestrator, not registered on the launched CLIs (decision
v0.29, reverses the v0.15-v0.28 design).** No external CLI (Codex/opencode/Agy) has Engram as an MCP — the
orchestrator does `mem_search` before delegating and passes the context directly in the prompt ("push"
model, not "pull"). Why: the benefit of each CLI searching memory on its own was marginal (the orchestrator
already searches before delegating) against the cost of keeping that access secure. History of the earlier
design (wrapper via `PATH`, custom `safe-reviewer` agents for Codex/opencode) is in `CHANGELOG.md`
v0.15-v0.28 — still on the filesystem, dormant, in case the decision is reverted.

**CLI roster restricted per project**: before assuming all 4 options in the table, check whether the
project fixes a smaller roster in `.specify/memory/constitution.md` or `CLAUDE.md`. Without that, use the
full table. A one-off verbal restriction from the user only applies to that session, unless it's persisted
in one of those files — today there's no other mechanism to fix it durably per project.

**Fixed models per CLI** (user decision, always use these — don't improvise a different model):

| CLI | Fixed model | `--kind` + native args (after `--`) | Default role |
|---|---|---|---|
| **Claude Code** | Sonnet 5 (CLI default, no flag) | `--kind claude` (no native args) | Default, orchestrator/leader (this same session) — doesn't launch itself. Can also **launch a second instance** via `agent start --kind claude` as executor/highest-reliability second opinion (same native mechanism as the orchestrator, no external gotchas) — see note below on its limits as a judge for independence |
| **Codex** | gpt-5.6-luna · high | `--kind codex -- -m gpt-5.6-luna -c model_reasoning_effort="high" -s workspace-write` | Independent second opinion/executor. `-s workspace-write` limits its sandbox to writing only within the project — a general security guardrail, not specific to Engram (which is no longer registered, see above). On first launch it may ask for hook/trust confirmation (blocks the pane until resolved — see Codex note below) |
| **opencode** | DeepSeek V4 Flash Free | `--kind opencode -- -m opencode/deepseek-v4-flash-free` | Independent executor **and cheap model-tiering minion** (pattern 4, Step 3). **`agent wait`/`agent_status` are reliable** (fixed in v0.38, earlier note was outdated — see note below) |

Claude#2 note — **as a judge, it gives process/context independence, not model independence.** A second
instance of Claude Code runs in a fresh context (it doesn't see the orchestrating session's reasoning), but
shares the same provider/model (Sonnet 5) as the author if the author is also this session — it doesn't
have the same value as Codex/opencode for blind dual-judge, where the independence being sought is
precisely from model biases. Prefer Codex/opencode when model independence matters; reserve Claude#2 for
roles where fresh context is enough (pattern 1 executor, pattern 5 context isolation) or when state-detection
reliability is what's needed above everything else.

**Agy (Antigravity) was removed from the active roster (v0.40)** — user decision after this live testing
session: its `agent_status` never reflected the real state (static fallback confirmed at 4 different points,
even after updating Herdr and its detection manifests), also contaminating `agent_start`/`agent wait`. The
full finding and investigation history remain in `TODO.md`/`CHANGELOG.md` (v0.30-v0.39) in case it's
reconsidered in the future — nothing was deleted from there, it just stopped being part of this table's
default roster.

**If Agy is launched at all (outside the default roster, or if reconsidered later), always pass
`--dangerously-skip-permissions`** (user decision, v0.49): `--kind agy -- --dangerously-skip-permissions`.
Equivalent to Claude Code's bypass-permissions/auto mode or opencode's build mode — auto-approves tool
permission requests instead of prompting. Worth retesting the v0.40 removal reason against this flag: part
of the unreliable `agent_status` came from Agy sitting at unresolved trust/confirmation prompts (see
`TODO.md` v0.30-v0.39), which this flag may avoid by never prompting in the first place — not yet verified
live.

opencode note — **`agent_status` is genuinely reliable, real lifecycle authority via hook** (fixed in
v0.38, the earlier note was outdated). Confirmed at `https://herdr.dev/docs/integrations/`: opencode is in
the "lifecycle authority" group (with Pi/OMP/Kimi/Kilo/MastraCode), unlike Claude Code/Codex ("session
identity", no real authority — although in practice its screen-manifest works fine anyway). Tested live:
`agent explain` showed `screen_detection_skip_reason: full_lifecycle_hook_authority` and `agent_status`
correctly followed `idle → working → idle` in sync with the pane's actual content. One nuance: a brief lag
(<1s) between submitting the prompt and the hook first reporting `working` — doesn't affect blocking calls
(`agent prompt --wait`/`agent wait`), only a standalone `agent get` without waiting right after submitting.

Codex note — `~/.codex/config.toml` has `model_reasoning_effort = "xhigh"` as its global default — different
from the `high` fixed here. The invocation passes **both** flags explicitly, `-m gpt-5.6-luna` and
`-c model_reasoning_effort="high"` — previously only the reasoning effort was set and the base model was
left to the mercy of the global config (bug found by blind review with Codex itself, see changelog v0.7).
Check the banner when launching it (`gpt-5.6-luna high`) to confirm both were applied.

If Herdr isn't active (see Step 0): use Claude Code's native subagents (Sonnet 5) for the Step 3 patterns
that don't depend on crossing providers (1, 2, 3, 5 work the same with native subagents). Pattern 4 (model
tiering) **has no cheap minion without Herdr** — it doesn't stay "unresolved": it explicitly degrades to
Delegated direct (Step 1, route 2) with a native subagent, without the cost savings but without blocking
the task.

**For any read-only/review role with a native subagent (judges, 4R lenses, model-tiering minion), use the
custom `safe-reviewer` agent** (`.claude/agents/safe-reviewer.md`) instead of `general-purpose` — it blocks
`Bash`/`Edit`/`Write`/`NotebookEdit` and Engram's write tools via `disallowedTools`, the only real way to
restrict a native subagent (it doesn't inherit `permissions.deny` from `settings.json` — a known Claude
Code limitation, `anthropics/claude-code#25000`). It excludes `Bash` on purpose, not just MCP tools: with
Bash available, the real `engram` binary could be invoked directly. If the role needs to run commands
(tests, linters), don't use `safe-reviewer` — use `general-purpose` with the text guardrail as partial
mitigation. Verified live: `mem_save`/`Bash` don't appear in `safe-reviewer`'s tool list, not even deferred
(`mem_search` does, selective blocking). Scope: only covers native subagents (Task tool); external CLIs via
Herdr don't read `.claude/agents/`, they're different surfaces.

## Step 5 — expected output

When applying this skill, state explicitly:
1. Result of the Herdr check (Step 0) and which options remain enabled.
2. Route chosen in Step 1 (Direct inline / Delegated direct / Multi-agent with pattern) and why.
3. If route 3: which Step 2 question triggered it, which pattern, and how many roles.
4. **Only if the route is 2 or 3 and an external CLI is involved**: which CLI/model for each role, with the
   reason. If it's route 1 (Direct inline, nothing to launch) or route 2 with a native subagent (no
   external CLI, no fixed model to report), omit this point instead of forcing it — there's no "for each
   role" to fill in when there aren't multiple roles.
5. **If there was degradation** (see Step 6): state it explicitly — which role was missing and which
   guarantee of the pattern was lost. Never report a multi-agent pattern as complete if it isn't.

## Step 6 — failures partway through

General rule: **never wait indefinitely, never leave orphaned tabs, never pretend a pattern completed if it
didn't.**

1. **Explicit timeout per role.** Every `agent prompt --wait`/`agent wait` carries a `--timeout` (60-120s
   for a normal prompt, more for long reasoning tasks) — never an unbounded wait. If the timeout is hit,
   don't keep waiting on that pane blindly: go straight to point 3 (degradation).
   **Exception confirmed in earpi (v0.33)**: if the task asks the executor to run commands as part of
   verifying its own work (`bun test`, build, lint), 180s wasn't enough with Codex nearly done — that time
   adds to the reasoning time, it doesn't replace it. For this kind of task use 300-600s, or prefer `herdr
   pane wait-output <target> --match <expected-end-text>` instead of a single long wait on `agent_status` —
   it allows checking real progress instead of waiting blindly until the limit.
2. **Herdr can go down partway through a pattern** (between `tab create` and `agent start`/`agent prompt`,
   or with agents already running). There's no proactive signal for this — it's detected because the next
   Herdr command fails or doesn't respond. Whenever a Herdr command fails partway through a pattern, rerun
   `herdr status` to confirm whether the server is still alive before assuming anything else.
3. **Degradation with partial results**: if some roles responded and others didn't (timeout, Herdr down, a
   pane that never started), continue with what was obtained — but state it explicitly in Step 5: which
   role is missing and which guarantee of the pattern was lost (blind dual-judge with 1 judge is a single
   opinion, not consensus; 4R lenses with 3 of 4 lenses covers 3 dimensions, not all 4). Never report the
   pattern as complete if it isn't.
4. **Tab cleanup**: if a pattern is aborted or degrades partway through, close with `herdr tab close
   <tab_id>` the tabs that were created, including ones that never started properly — don't leave orphaned
   tabs running in the user's workspace.
5. **The single correction attempt (Surgical single-attempt correction) can also fail**: if the applied fix
   doesn't resolve the problem, don't retry in a loop — report it to the user as an unresolved finding, not
   as an error to force until it works.

## Status and adjustments

Full version history in `CHANGELOG.md`, in this same skill directory — not loaded here so as not to bloat
the file that's read on every invocation. `TODO.md` stands as the record of method and findings per round —
check there for the state of active pending items instead of assuming it from here.
