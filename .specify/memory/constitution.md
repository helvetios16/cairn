<!--
Sync Impact Report
- Version change: TEMPLATE → 1.0.0 (initial ratification)
- Modified principles: none (first formal adoption; principles below codify practices already
  applied in this repo's history, not new rules)
- Added principles:
  I. Prueba adversarial real, no especulación
  II. No generalizar sin necesidad concreta
  III. Confirmación humana explícita ante acciones irreversibles
  IV. Trazabilidad de versiones
  V. Iteración dirigida por el usuario
  VI. Concepto por encima de marca
- Added sections: Governance
- Removed sections: [SECTION_2_NAME] / [SECTION_3_NAME] placeholders from the template — no
  additional constraints or workflow content beyond the six principles applies to this repo today
- Templates requiring updates: none flagged — `.specify/templates/plan-template.md`'s
  "Constitution Check" section already reads this file generically, no template edit needed
- Follow-up TODOs: none
-->

# Cairn Constitution

Repo de skills de Claude Code. Colección de skills independientes (`agent-selection`, y las que se
agreguen — incluyendo la de SDD/Spec Kit en construcción), cada una versionada y documentada por
separado. Esta constitución rige cómo se diseñan, prueban y evolucionan esas skills, no el
contenido técnico de ninguna en particular.

## Core Principles

### I. Prueba adversarial real, no especulación

Toda afirmación sobre seguridad o comportamiento de una skill se verifica ejecutándola de verdad
—CLIs reales vía Herdr, subagentes reales, comandos reales— antes de darla por cierta. Nunca se
documenta como resuelto un hallazgo que solo fue razonado en teoría; si algo no se pudo probar en
vivo, se deja anotado explícitamente como pendiente de verificación, no como cerrado.

**Rationale**: el historial de este repo (rondas de seguridad de `agent-selection`) mostró
repetidas veces que el razonamiento teórico sobre sandboxes y permisos no coincidía con el
comportamiento real de los CLIs — solo la ejecución en vivo destapó los gaps reales.

### II. No generalizar sin necesidad concreta

No se crean abstracciones, wrappers o mecanismos genéricos anticipando casos futuros hipotéticos.
Un mecanismo nuevo solo se construye cuando aparece un caso real y angosto que lo amerita, probado
con la misma metodología adversarial del Principio I.

**Rationale**: un wrapper o guardrail mal generalizado da falsa sensación de seguridad —es peor
que no tener nada, porque hace parecer resuelto algo que no lo está.

### III. Confirmación humana explícita ante acciones irreversibles

Ninguna skill de este repo automatiza sin gate humano una acción irreversible o de alto impacto:
borrado, migración, `git push`/force, o lanzar un agente con capacidad destructiva. El guardrail es
la confirmación explícita del usuario antes de ejecutar, no una validación posterior.

**Rationale**: el costo de pausar a confirmar es bajo; el costo de una acción destructiva no
pedida (trabajo perdido, un push no deseado) puede ser alto y no siempre reversible.

### IV. Trazabilidad de versiones

Todo cambio a una skill bumpea su versión en el frontmatter (`metadata.version`) y agrega una
entrada en el `CHANGELOG.md` de esa skill explicando qué cambió y por qué. El historial completo
vive en `CHANGELOG.md`; el `SKILL.md` principal —lo que se carga en cada invocación— nunca se
infla con historial, solo con estado y criterio vigente.

**Rationale**: mantiene cada invocación de la skill liviana sin perder trazabilidad de cómo llegó
a su forma actual.

### V. Iteración dirigida por el usuario

El flujo de refinamiento de una skill es: caso real → ejecutar el framework en vivo → surgen
hallazgos → el usuario prioriza explícitamente qué aplicar → se aplica solo ese lote → bump de
versión. No se aplican baterías completas de cambios de una vez sin que el usuario elija el orden.

**Rationale**: da al usuario control explícito sobre el ritmo y alcance de cada cambio, en vez de
que una ronda de hallazgos se traduzca automáticamente en una reescritura grande.

### VI. Concepto por encima de marca

El catálogo de patrones y conceptos de ingeniería de agentes que informa las skills de este repo se
separa de la marca o herramienta puntual que los popularizó. Se adopta el principio transferible;
la herramienta concreta (Spec Kit, un curso, un framework con nombre propio) se evalúa aparte y se
puede reemplazar sin invalidar el principio.

**Rationale**: evita que una skill quede atada a las decisiones de producto de una herramienta de
terceros cuando lo que realmente importa es el patrón subyacente.

## Governance

Esta constitución tiene precedencia sobre las decisiones de diseño de cualquier skill individual
de este repo. Ante un conflicto entre lo que documenta el `SKILL.md` de una skill puntual y un
principio de acá, gana esta constitución hasta que se resuelva explícitamente (enmendándola o
documentando una excepción justificada en la skill).

**Enmiendas**: cualquier cambio a esta constitución requiere el mismo criterio del Principio V
—propuesta explícita, decisión del usuario, no aplicación silenciosa— y se versiona según semver:
MAJOR para remoción o redefinición incompatible de un principio, MINOR para agregar un principio o
ampliar sustancialmente su guía, PATCH para aclaraciones de redacción sin cambio de fondo.

**Revisión de cumplimiento**: al crear o modificar sustancialmente una skill de este repo, revisar
que no contradiga estos seis principios antes de darla por lista. No hace falta un checklist
formal por PR (no hay flujo de PR en este repo local) — la revisión ocurre en la misma sesión que
introduce el cambio.

**Version**: 1.0.0 | **Ratified**: 2026-08-07 | **Last Amended**: 2026-08-07
