# Roadmap

**Last rotated**: 2026-07-28
**Stale threshold**: 6 months
**Visibility**: public

<!-- See .claude/rules/roadmap.md for the cycle, evidence categories, and discipline. -->

## Stale items

<!-- Auto-computed at write time: items with last_reviewed older than Stale threshold. -->

Ninguno.

## Themes

<!-- Optional groupings. Two-level maximum (themes do not nest). Status is computed. -->

## Items

### ROADMAP-001: El currículo, consultable como dato

**Status**: Shipped
**Last reviewed**: 2026-07-28
**PRD**: PRD-002

**Problem / outcome**: El currículo vivía en tres constantes de TypeScript que no se conocían entre sí, así que la plataforma no podía responder preguntas de su propio dominio —cuántas lecciones tiene un módulo, en qué módulo va un estudiante— y corregir una errata obligaba a un principiante a editar código de producción. Pasa a ser un árbol consultable en base de datos, proyectado desde un archivo versionado.
**User**: estudiante (selector de lección y temario de la home), autor de contenido (erratas y clases nuevas), adoptante del repositorio.
**Siblings likely impacted**: platform

**Evidence**:
- PRD-002 <!-- categoría 7: item retroactivo creado por el flujo de auto-update (PRD-001 § 4.2); la justificación vive en PRD-002 § 1 -->

### ROADMAP-002: El acceso y el cobro, fuera del framework web

**Status**: Shipped
**Last reviewed**: 2026-07-28
**PRD**: PRD-003

**Problem / outcome**: La lógica de negocio vivía dentro de Next, así que la frontera entre dominio y presentación dependía de la disciplina de quien escribía el import, y no había superficie desde la que servir a un consumidor que no fuera el navegador. El dominio de acceso y cobro pasa a un servicio propio, `apps/api`, detrás de un flag apagado por defecto: la frontera deja de ser una convención y pasa a ser un proceso. Primera fase de ADR-001; el resto del sistema no se ha movido.
**User**: mantenedores del repositorio (la frontera y su coste de mantenimiento); estudiante (sin cambio observable mientras el servicio responda — el estado degradado sí es visible y está declarado en PRD-003 § 2).
**Siblings likely impacted**: platform

**Evidence**:
- PRD-003 <!-- categoría 7: item retroactivo creado por el flujo de auto-update (PRD-001 § 4.2); la justificación vive en PRD-003 § 1 -->
