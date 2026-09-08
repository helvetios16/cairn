# Changelog — sdd-propose

- **v0.2** — adds a grounding step (new Step 2, subsequent steps renumbered) inspired by
  comparing this skill's flow against Gentle (Gentleman Programming's OpenCode orchestrator, see
  the Phyume note): for brownfield features, delegates a read-only exploration of the real repo
  (via the `Explore` agent type) to an agent separate from the one drafting `proposal.md`, so
  `archivos_afectados` and `riesgos` are grounded in verified paths instead of guesses. Skipped
  explicitly for greenfield features with nothing existing to explore (Constitution Principle II —
  don't force a step without a concrete need). Findings are treated as verified structure, not
  verified behavior (a static read doesn't confirm a code path works end-to-end).
- **v0.1** — first version. Adds the framing before the Spec Kit flow: receives a feature
  description, delegates directory creation and numbering to
  `.specify/scripts/bash/create-new-feature.sh`, and writes `proposal.md` with problema,
  alcance_incluye, alcance_excluye, archivos_afectados, riesgos, and rollback. Makes the value of
  `SPECIFY_FEATURE_DIRECTORY` explicit so `/speckit-specify` reuses the same folder, and requires
  reviewing the `agent-selection` risk list before indicating the next step. Keeps the proposal
  focused on the concrete case and limits clarifications to three, in line with the Spec Kit
  criteria and the repo constitution.
