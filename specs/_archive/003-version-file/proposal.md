# Proposal: Archivo VERSION en la raíz del repo

## problema

No hay un único lugar legible por script (sin depender de `git tag`) que indique la versión
actual del repo `cairn`. Esta feature es, además, un caso de prueba deliberadamente chico y
descartable para correr el ciclo completo `sdd-propose` → `/speckit-specify` → implementación →
`sdd-verify` → `sdd-archive` de punta a punta.

Supuestos: no hace falta automatizar la generación del valor desde `git describe`/tags — un string
fijo simple (`0.1.0-dev`) alcanza para esta versión. No hay actores externos ni flujo de usuario
real detrás de esta feature; es infraestructura de prueba interna.

## alcance_incluye

- Un archivo `VERSION` en la raíz del repo (`/Users/sebastian/Documents/Variety/cairn/VERSION`) con
  una única línea de texto (el string de versión).

## alcance_excluye

- Automatizar la generación del valor desde `git tag`/`git describe`.
- Cualquier mecanismo de bump automático de versión (ya existe el patrón manual de
  `metadata.version` + `CHANGELOG.md` por skill, definido en la constitución — esta feature no lo
  reemplaza ni lo toca).
- Cualquier consumo del archivo `VERSION` por otro script o proceso — queda fuera de alcance, es
  solo el archivo.

## archivos_afectados

- `VERSION` (nuevo, raíz del repo).

## riesgos

Riesgo mínimo: un solo archivo de texto plano en la raíz del repo, sin lógica ni dependencias.

Lista de riesgo de agent-selection: no detectada. Revisado contra los patrones del Paso 2 de
`agent-selection` (`.env*`, SSH/credenciales, CI/CD, infraestructura, migraciones,
producción/deploy, auth/pagos/borrado masivo) — `VERSION` no coincide con ninguno.

## rollback

Borrar el archivo `VERSION` de la raíz del repo. Sin estado adicional que revertir (no hay
migración, no hay configuración externa que dependa de su existencia todavía).
