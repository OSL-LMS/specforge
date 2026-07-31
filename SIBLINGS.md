# Sibling projects

| Name | Path | Status | Read first |
|---|---|---|---|
| apps-web | `../platform/apps/web` | active | `CLAUDE.md`, `docs/SYSTEM_ARTIFACT.md` |
| apps-api | `../platform/apps/api` | active | `CLAUDE.md`, `docs/SYSTEM_ARTIFACT.md` |
| shared | `../platform/packages/shared` | active | `CLAUDE.md`, `docs/SYSTEM_ARTIFACT.md` |
| platform | `../platform` | retired | `AGENTS.md`, `docs/SYSTEM_ARTIFACT.md` |

## Rules

El registro es **append-only**. Una fila no se borra ni se reescribe: se marca `retired` y se añade la que la reemplaza. Los PRD `Implemented` y `Superseded` son instantáneas congeladas y siguen citando el nombre que era correcto cuando se escribieron — `platform` aparece en la tabla `Impacted Projects` de PRD-002 a PRD-006, y esas filas tienen que seguir resolviendo.

Consecuencia práctica (hard rule 11): un PRD `Draft` sólo puede citar filas `active`; uno histórico puede citar filas `retired`.

**Un sibling es una unidad con convenciones propias, no un repositorio.** Los tres primeros comparten el repositorio `platform` —un historial, un merge, un despliegue— y aun así son siblings separados, porque no comparten nada de lo que un implementador o un revisor necesita saber: stack, runner de tests, sistema de módulos, artefacto de despliegue ni reglas de frontera. Cada uno declara su `CLAUDE.md` y su `SYSTEM_ARTIFACT.md`.

**Un PRD que toque dos paquetes cita dos filas**, y su `system_artifact_diff` lista los dos documentos. Es lo normal aquí, no la excepción: los dominios cruzan paquetes —el `tutor` vive en el proxy de `apps/web` y en el servicio de `apps/api`— y desde 2026-07-31 cada mitad se documenta con el código que la implementa y referencia a la otra por nombre.

**`commit_hash` sigue siendo uno solo** aunque se citen varias filas: los tres comparten repositorio, así que un cambio que los toque a la vez aterriza en un commit. `gate-block.md` ya lo contempla para PRDs multi-sibling.

## Cuándo añadir una fila

Cuando aparezca una unidad con convenciones propias que un PRD vaya a citar: un paquete nuevo del workspace, o un repositorio nuevo. No cuando aparezca un directorio.

`platform` pasó a `retired` el 2026-07-31 al partirse en los tres de arriba. No desapareció nada: el repositorio existe y sigue siendo donde aterrizan los commits. Lo que dejó de tener sentido es citarlo como **una** unidad de convenciones, que es lo que esta tabla declara. Su `docs/SYSTEM_ARTIFACT.md` se retiró en el mismo movimiento; los PRD que lo citan lo hacen por ruta **y commit**, así que resuelven en el historial de git.
