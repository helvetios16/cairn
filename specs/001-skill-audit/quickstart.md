# Quickstart: Auditoría de consistencia de skills

## Prerrequisitos

- Estar en la raíz del repo `cairn`.
- `scripts/check-skills.sh` implementado (Fase 2/3, ver `tasks.md`).

## Escenario 1 — estado actual del repo (Historia 1 + 2)

```bash
chmod +x scripts/check-skills.sh   # solo la primera vez
./scripts/check-skills.sh
echo "exit code: $?"
```

**Esperado**: una fila por chequeo aplicable, todas en `pass` (a la fecha de este plan, las skills
propias del repo — `agent-selection`, `sdd-implement` — tienen `name` sincronizado y
`CHANGELOG.md`; las `speckit-*` no declaran `metadata.version`, así que su chequeo de changelog
aparece `skipped`, no `fail`). Exit code `0`.

## Escenario 2 — introducir un `name` desincronizado (Acceptance Scenario 2 de Historia 1)

```bash
cp .claude/skills/sdd-implement/SKILL.md /tmp/sdd-implement-SKILL.md.bak
sed -i '' 's/name: "sdd-implement"/name: "sdd-implement-renamed"/' .claude/skills/sdd-implement/SKILL.md
./scripts/check-skills.sh
echo "exit code: $?"   # esperado: 1
# revertir
cp /tmp/sdd-implement-SKILL.md.bak .claude/skills/sdd-implement/SKILL.md
rm /tmp/sdd-implement-SKILL.md.bak
```

**Esperado**: la fila `sdd-implement | name-matches-dir | fail` aparece con el detalle
`sdd-implement-renamed` (encontrado) vs. `sdd-implement` (esperado). Exit code `1`. Después de
revertir, correr de nuevo el Escenario 1 y confirmar que vuelve a `pass`/exit `0`.

## Escenario 3 — quitar un `CHANGELOG.md` (Acceptance Scenario 2 de Historia 2)

```bash
mv .claude/skills/sdd-implement/CHANGELOG.md /tmp/sdd-implement-CHANGELOG.md.bak
./scripts/check-skills.sh
echo "exit code: $?"   # esperado: 1
# revertir
mv /tmp/sdd-implement-CHANGELOG.md.bak .claude/skills/sdd-implement/CHANGELOG.md
```

**Esperado**: la fila `sdd-implement | has-changelog | fail`. Exit code `1`. Al revertir, vuelve a
`pass`/exit `0` y ninguna otra skill cambia de resultado (SC-003).
