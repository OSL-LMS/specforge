# AgDR-001: `divergencias` cuenta las dos direcciones, incluidas las que nunca se reparan

**Status**: Recorded
**Date**: 2026-07-28
**Agent**: implementador del alcance 2 de PRD-004 (worker y módulo `reconcile`)
**Triggering PRD**: PRD-004
**Sibling**: platform
**Commit**: 867c918

---

## 1. Decision

`divergencias` en la línea de resumen de PRD-004 § 8.2 cuenta **toda** diferencia detectada entre lo que reporta Paddle y lo que guarda `subscriptions`, **en las dos direcciones**, y por tanto **incluye** las divergencias en dirección `canceled` que `pendiente_revocacion` cuenta aparte y que el barrido nunca escribe. La relación resultante es invariante: `divergencias ≥ pendiente_revocacion`, y `reparadas ≤ divergencias − pendiente_revocacion`.

La cuenta es **por fila local** cuando el correo tiene filas (dos filas divergentes del mismo correo suman 2) y **1** cuando Paddle reporta algo para un correo sin fila. `ambiguo`, en cambio, cuenta **correos**, no filas.

## 2. Why the PRD did not cover this

§ 8.2 fija la **forma** de la línea —los nueve campos, su orden y sus nombres— y la declara contrato porque el paso 4 de § 10 depende de leerla. Lo que no fija es la **semántica de un campo cuando dos lecturas son defendibles**: § 6.5 enumera qué hace el barrido en cada caso y qué contador incrementa, pero nunca dice si `divergencias` es el total de las dos direcciones o solo el de la dirección que se repara. El goal 1 —"detectar toda divergencia […] **en las dos direcciones**"— empuja hacia el total; § 10 paso 4 —"`divergencias` distintas de 0 en pasadas sucesivas […] son deriva real"— se lee natural con cualquiera de las dos.

No se podía diferir a una revisión del PRD porque el contador tiene que existir para que la fila 15 de § 9 afirme "los nueve campos **con los valores correctos**": sin elegir, no hay valor correcto que afirmar.

## 3. Alternatives Considered

### Option A — Total de las dos direcciones *(elegida)*

`divergencias` es la suma de lo detectado en ambas direcciones. Se apoya en el texto del goal 1, que es la formulación más explícita que el PRD da sobre este contador, y produce una lectura de la línea que se sostiene sola: `divergencias=5 pendiente_revocacion=5 reparadas=0` dice "toda la deriva que queda es revocación pendiente, y no hay nada más"; `divergencias=5 pendiente_revocacion=1 reparadas=4` dice "se arregló todo lo arreglable".

### Option B — Solo la dirección que se repara *(rechazada)*

`divergencias` contaría únicamente las diferencias hacia `active`, y `pendiente_revocacion` sería un contador disjunto. Tiene la propiedad atractiva de que en régimen estacionario `divergencias` tiende a 0 y cualquier valor distinto de 0 es una alarma. Se rechaza por dos razones: contradice la letra del goal 1 —un contador llamado "divergencias" que ignora una de las dos direcciones que el goal manda detectar es un nombre que miente—, y deja al operador sin ningún número que responda "¿cuánta deriva total hay?", que es la pregunta del paso 4 de § 10 antes de decidir el paso 5.

## 4. Blast radius and reversal cost

El coste no está en el código —cambiarlo son dos líneas de `reconcile.service.ts` y sus tests— sino en **la serie temporal**. El plan de § 10 es explícitamente comparativo: una semana de observación con `RECONCILE_APPLY` ausente (paso 4), y solo entonces la primera pasada con escritura (paso 5), "solo si lo observado se explica". Cambiar la definición de `divergencias` después de esa semana rompe la comparación sin dejar rastro: el número sube o baja por un cambio de significado y se lee como un cambio de la realidad. Ése es exactamente el modo de fallo que un cambio de métrica silencioso produce.

Por la misma razón, la implementación cuenta las divergencias **también en modo sin escritura** —marcando la fila como resuelta aunque no se escriba—, para que las dos modalidades den el mismo número ante la misma realidad. Sin eso, el paso 5 compararía peras con manzanas aunque la definición no cambiara.

Nada más depende de esto: no hay esquema, ni migración, ni contrato HTTP, ni dependencia. El evento `subscription_reconciled` no lleva contadores.

## 5. Signals to Reconsider

| Signal | Action |
|--------|--------|
| El paso 4 de § 10 termina y el operador declara que `divergencias` no le sirvió para decidir el paso 5 | Reconsiderar la semántica en el PRD que automatice las revocaciones (§ 11), que es donde `pendiente_revocacion` deja de ser cola humana |
| Se implementa la escritura de revocaciones (§ 11) | La dirección `canceled` pasa a ser reparable y la distinción de este AgDR desaparece: `divergencias` vuelve a ser simplemente "lo que hay que arreglar" |

---

## Related Documents

- [PRD-004: Reconciliación con Paddle — el primer trabajo diferido de `apps/api`](004-reconciliacion-paddle-worker.md) — the triggering PRD
