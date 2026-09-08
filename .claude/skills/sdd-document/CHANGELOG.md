# Changelog — sdd-document

- **v0.1** — first version. New skill that maintains `ARCHITECTURE.md`, a portable architecture
  map (stack, module map, conventions, entry points) meant as the base any AI reads before
  exploring this repo — deliberately not `CLAUDE.md`, so it isn't tied to one specific tool
  (Constitution Principle VI). Two modes: bootstrap (first run, no existing file, full-repo pass
  via a dedicated read-only `Explore` agent) and incremental (the file already exists — grounds the
  update in what a given feature's `sdd-implement` run actually touched, editing only the
  overlapping section(s) instead of re-scanning or regenerating the whole file). Motivated directly
  by the user's requirement that this not be Claude-Code-specific (unlike the native `init` skill)
  and that it plug into the SDD flow's own data (what `sdd-implement` touched) rather than
  re-deriving repo state independently each time. Optional, invoked explicitly — not wired as a
  precondition into `sdd-implement`, `sdd-judge`, `sdd-verify`, or `sdd-archive`. Closes a narrower
  gap than Gentle's "Cognitive Doc Design" (which documents deployment for humans, not architecture
  for AI exploration) — see the sdd-judge v0.2 and sdd-propose v0.2 decisions from the same Gentle
  comparison for the related prior gaps closed.
