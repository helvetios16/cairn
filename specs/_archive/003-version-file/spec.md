# Feature Specification: Archivo VERSION en la raíz del repo

**Feature Branch**: `003-version-file`

**Created**: 2026-08-07

**Status**: Draft

**Input**: User description: "Archivo VERSION en la raiz del repo con un string fijo de version,
feature de prueba descartable para validar el ciclo sdd-propose/verify/archive"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Consultar la versión del repo desde un archivo (Priority: P1)

Como mantenedor del repo, quiero un archivo `VERSION` legible en la raíz que indique la versión
actual, sin depender de correr `git tag`/`git describe`.

**Why this priority**: única historia de esta feature — es un caso de prueba chico y descartable
para el ciclo completo `sdd-propose` → `sdd-verify` → `sdd-archive` (ver `proposal.md`).

**Independent Test**: leer el archivo `VERSION` en la raíz del repo y confirmar que contiene un
string de versión no vacío.

**Acceptance Scenarios**:

1. **Given** el repo `cairn`, **When** se lee `VERSION` en la raíz, **Then** el archivo existe y
   contiene una única línea con un string de versión no vacío.

### Edge Cases

- N/A — feature de un solo archivo estático, sin lógica ni casos límite reales.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El repo MUST tener un archivo `VERSION` en su raíz.
- **FR-002**: `VERSION` MUST contener una única línea de texto con un string de versión no vacío.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: `test -f VERSION` en la raíz del repo devuelve éxito.
- **SC-002**: El contenido de `VERSION` no está vacío.

## Assumptions

- No hace falta automatizar la generación del valor desde `git tag`/`git describe` (ver
  `proposal.md`, alcance_excluye) — un string fijo alcanza.
