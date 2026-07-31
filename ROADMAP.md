# Roadmap

**Last rotated**: 2026-07-31
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

### ROADMAP-003: Que un pago perdido deje de costar el acceso

**Status**: Shipped
**Last reviewed**: 2026-07-28
**PRD**: PRD-004

**Problem / outcome**: El estado de cobro llegaba por una sola vía —el webhook de Paddle— y nada detectaba ni reparaba un evento perdido, así que un estudiante que pagaba podía quedarse sin tutor hasta el siguiente evento de su suscripción, que puede tardar un mes. Un barrido periódico repara esa dirección sin intervención. La contraria —seguir sirviendo a quien canceló— se detecta y se cuenta, pero se deja a decisión humana a propósito: una revocación automática por correo es irreversible y el correo lo elige quien inicia el checkout.
**User**: estudiante que paga (recupera el acceso sin escribir a soporte); mantenedores del repositorio (la deriva pasa a ser un número observable en cada pasada en vez de una incógnita).
**Siblings likely impacted**: platform

**Evidence**:
- PRD-004 <!-- categoría 7: item retroactivo creado por el flujo de auto-update (PRD-001 § 4.2); la justificación vive en PRD-004 § 1 -->

### ROADMAP-004: El tutor, fuera del framework web

**Status**: Shipped
**Last reviewed**: 2026-07-30
**PRD**: PRD-005

**Problem / outcome**: El tutor —lo único de la plataforma por lo que alguien paga— seguía viviendo dentro del servidor que sirve páginas, así que la frontera que ADR-001 compró cubría la periferia y no el centro. Y lo que se le enseñaba al modelo salía del array que mandaba el navegador, no de la conversación guardada, de modo que un cliente podía fabricar lo que el tutor supuestamente había dicho antes. Pasa a servirse desde `apps/api`, con Next de proxy, y el hilo pasa a salir de la base.
**User**: estudiante (sin cambio observable salvo que el turno abandonado deja de facturarse); mantenedores del repositorio (la frontera y el coste de sostenerla).
**Siblings likely impacted**: platform

**Evidence**:
- PRD-005 <!-- categoría 7: item retroactivo creado por el flujo de auto-update (PRD-001 § 4.2); la justificación vive en PRD-005 § 1 -->

### ROADMAP-005: La frontera entre dominio y presentación, terminada

**Status**: Shipped
**Last reviewed**: 2026-07-31
**PRD**: PRD-006

**Problem / outcome**: ADR-001 decidió tres paquetes y sólo se habían construido dos, así que `apps/api` alcanzaba siete módulos de la raíz —cinco por ruta relativa de cuatro niveles, dos por copia— y la raíz del repositorio era a la vez raíz del workspace y servicio desplegado. La deriva que esa forma permite había ocurrido ya: el union de eventos de telemetría tenía seis miembros en un lado y siete en el otro. Los tres paquetes existen ahora y los duplicados se cerraron por construcción. Cierra ADR-001; el equipo deja de sostener una migración a medias, que su § 6 nombra como el peor de los estados posibles.
**User**: mantenedores del repositorio (la frontera y su coste de mantenimiento); estudiante (sin cambio observable — mismas rutas, misma sesión, mismo dominio).
**Siblings likely impacted**: platform

**Evidence**:
- PRD-006 <!-- categoría 7: item retroactivo creado por el flujo de auto-update (PRD-001 § 4.2); la justificación vive en PRD-006 § 1 -->

**Caveats**: el repositorio gana su primera CI en el mismo corte, y es **advisoria**: no bloquea el merge, porque la protección de rama quedó declinada por decisión del propietario. Compra visibilidad, no puerta.

### ROADMAP-006: La evidencia del estudiante, recogida y comprobada

**Status**: Shipped
**Last reviewed**: 2026-07-31
**PRD**: PRD-007

**Problem / outcome**: La plataforma era un chat con un muro de pago que no registraba nada de lo que hacía un estudiante: ni quién avanzaba, ni quién se atascaba en qué lección, ni de dónde saldría el portafolio verificable que la escuela vende. El dato que lo hacía barato ya estaba escrito y sin usar — el `outcome` de cada lección no es un objetivo de aprendizaje, es un artefacto que existe en internet. Pasa a recogerse por estudiante y por lección, distinguiendo **declarado** de **verificado** en el esquema y no en la interpretación, porque un portafolio que acepta autodeclaraciones no es verificable. El progreso no se mide por consumo: si la métrica se puede subir sin aprender nada, es la métrica equivocada.
**User**: estudiante (entrega y ve su evidencia); evaluador de la defensa H1 (deja de abrir a mano el repositorio de cada estudiante antes de cada sesión); mantenedores (el abandono por lección pasa a ser consultable).
**Siblings likely impacted**: apps-api, apps-web, shared

**Evidence**:
- CON-6 <!-- categoría 2: issue del tracker del equipo, con el problema y sus restricciones pedagógicas -->
- PRD-007 <!-- categoría 7: item retroactivo creado por el flujo de auto-update (PRD-001 § 4.2); la justificación vive en PRD-007 § 1 -->

**Caveats**: la verificación es de **alcanzabilidad**, no de contenido — "una página con su nombre" y "un repositorio con más de un commit" quedan para un PRD posterior. Y de las siete lecciones de E1 solo tres producen un artefacto propio: el `verified` de L2, L3, L4 y L6 apunta a la misma dirección que el de L1, así que sirve como señal de abandono y no como evidencia de piezas distintas.
