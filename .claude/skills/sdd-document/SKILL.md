---
name: "sdd-document"
description: "Generates and incrementally updates ARCHITECTURE.md, a portable architecture map of the repo grounded in what sdd-implement actually touched — meant as the base any AI (not just Claude Code) reads before exploring, kept current feature by feature instead of going stale like a one-shot scan."
argument-hint: "Optional feature directory whose implementation should drive this update (e.g. specs/003-user-auth). If omitted, resolved the same way as sdd-verify (prerequisites script)."
compatibility: "Works with or without a Spec Kit project structure. Incremental mode benefits from a Spec Kit feature with tasks.md; bootstrap mode (first run) needs only the repo itself."
metadata:
  status: experimental
  version: "0.1"
user-invocable: true
disable-model-invocation: false
---

## What this skill does

Maintains `ARCHITECTURE.md` at the repo root: a portable architecture map — stack, module map, key
conventions, entry points — meant as the base any AI reads before exploring this repo, regardless
of which CLI or tool is doing the exploring (Claude Code, Codex, opencode via Herdr, or anything
else). Deliberately **not** `CLAUDE.md`: that filename and its conventions are specific to one
tool, and this repo's own constitution asks for the transferable concept, not a specific brand
(Principle VI) — the doc itself has to be tool-agnostic to be useful to more than one AI.

Unlike a one-shot whole-repo scan, this skill has two modes and prefers the cheaper one whenever
it can:

- **Bootstrap mode**: `ARCHITECTURE.md` doesn't exist yet. A first full pass over the repo is
  unavoidable — there's nothing to update incrementally from.
- **Incremental mode** (the common case after bootstrap): `ARCHITECTURE.md` already exists. Only
  the section(s) covering what a given feature's `sdd-implement` run actually touched get
  reviewed and updated — not a full re-scan. This is what keeps the doc from going stale without
  making every feature pay the cost of re-documenting the whole repo.

## User Input

```text
$ARGUMENTS
```

Optional feature directory (e.g. `specs/003-user-auth` or `003-user-auth`) whose implementation
should drive this update. If empty, resolve the same way `sdd-verify` resolves `$ARGUMENTS`: via
`.specify/scripts/bash/check-prerequisites.sh --json`, without requiring tasks — the most recently
implemented feature in the active `specs/` tree.

## Step 1 — decide bootstrap vs. incremental

1. Check whether `ARCHITECTURE.md` exists at the repo root.
   - **Doesn't exist** → bootstrap mode.
   - **Exists** → incremental mode.
2. Resolve the feature driving this update (from `$ARGUMENTS`, or the fallback above). In
   bootstrap mode a feature isn't required — the skill can also be run standalone before any SDD
   cycle has happened, to seed the first version. If no feature resolves in bootstrap mode, note in
   the Completion Report that this version isn't tied to a specific feature.
3. In incremental mode, a resolvable feature **is** required: if none resolves, stop and report
   "nothing to update from" rather than guessing what changed — same standard `sdd-verify` applies
   to not fabricating verdicts without a basis.

## Step 2 — gather the real touched surface

- **Incremental mode**: ground the update in what `sdd-implement` actually did for the resolved
  feature, not a fresh diff-everything pass:
  1. Prefer the per-phase file lists `sdd-implement` already produced while executing
     `tasks.md` (each phase's tasks name the files they touch — `sdd-implement` Step 2/3 already
     computes this for `agent-selection`'s routing). If this skill runs right after
     `sdd-implement` in the same session, reuse that data directly.
  2. If invoked standalone afterward (no live phase data available), fall back to
     `git diff $(git merge-base <default-branch> HEAD)` scoped to the feature's branch, and
     `tasks.md` for which phases are marked `[X]`.
  3. This surface is deliberately narrow — only what the feature touched, not the whole repo.
- **Bootstrap mode**: the surface is the whole repo. Delegate this pass to a dedicated read-only
  agent (the `Explore` agent type), separate from the one drafting `ARCHITECTURE.md` — same
  "explore, then generate" split `sdd-propose`'s grounding step uses, so the first version isn't
  contaminated by unverified assumptions about the repo's structure.

## Step 3 — update or create `ARCHITECTURE.md`

- **Bootstrap**: draft the file with these sections: `## Stack` (languages, frameworks, package
  manager), `## Module map` (top-level directories and each one's responsibility, one line each),
  `## Key conventions` (naming, testing approach, anything a newcomer AI would otherwise have to
  infer by trial and error), `## Entry points` (where execution starts, main scripts/commands),
  and a closing `## Last updated` line (see Step 4).
- **Incremental**: read the existing `ARCHITECTURE.md` in full first, locate which section(s)
  overlap the touched surface from Step 2, and edit only those — do not regenerate the whole file.
  Regenerating from scratch on every feature defeats the purpose (cost, and it flattens nuance a
  human may have hand-edited into the doc since bootstrap). If the touched surface doesn't map to
  any existing section (a genuinely new module/area), add a new section rather than force-fitting
  it into an unrelated one.
- In both modes: write what's verifiably true from Step 2's surface, not speculation about parts
  of the repo this run didn't actually look at. A section this run didn't touch is left exactly as
  it was — never "refreshed" on a guess.

## Step 4 — traceability

Update the `## Last updated` line with the feature (path) that drove this version and today's
date, e.g. `Last updated: specs/003-user-auth (implemented 2026-08-22)`, or `Last updated:
bootstrap, not tied to a specific feature (2026-08-22)` if Step 1 had no feature to resolve.
Constitution Principle IV (version traceability) applies to this doc's history the same as to any
skill.

## Step 5 — Completion Report

Report to the user:

- `Mode`: bootstrap or incremental.
- `Feature`: which feature drove this update, if any.
- `Sections changed`: the exact section headings touched (or "all, first bootstrap version").
- `ARCHITECTURE.md`: confirmation of the path written.

## Done When

- [ ] Bootstrap vs. incremental was decided from whether `ARCHITECTURE.md` already exists — never
      assumed.
- [ ] Incremental mode never re-scanned the whole repo; it grounded the update in the resolved
      feature's actual touched surface only.
- [ ] Incremental mode never regenerated the whole file — only the overlapping section(s), or a
      new section for a genuinely new area.
- [ ] `## Last updated` cites the feature (or explicitly notes there wasn't one) and the date.
- [ ] Nothing was written about parts of the repo this run didn't actually look at.

## Notes

- Not wired as a mandatory step of `sdd-implement` or `sdd-archive` — optional, meant to be
  invoked explicitly once a feature's implementation is done (natural point: right after
  `sdd-implement`'s closing, while its per-phase file data is still fresh in the session).
- Doesn't replace `README.md` (product-facing, for humans deciding whether to use the project) —
  this is codebase-facing, written for an AI that needs to explore the repo without re-deriving its
  structure from scratch every session.
- Closes a narrower gap than Gentle's own "Cognitive Doc Design": that skill documents
  *deployment* (install steps, `docker-compose`, migrations) for humans standing up the app. This
  skill documents *architecture* for AIs exploring the codebase — a different audience and a
  different kind of document, not a port of that skill.
