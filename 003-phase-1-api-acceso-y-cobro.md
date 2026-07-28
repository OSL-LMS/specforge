# PRD-003 (fase 1): `apps/api` en NestJS — el dominio de acceso y cobro

**Status**: Draft
**Date**: 2026-07-28
**Author**: AI-assisted
**Priority**: P1
**Depends on**: ADR-001
**Supersedes**: None
**Issue**: —

## Impacted Projects

| Project | Impact |
|---------|--------|
| **platform** | `pnpm-workspace.yaml` declara `packages: ["apps/*"]` y aprueba en `allowBuilds` los scripts de build nuevos. Paquete nuevo `apps/api` (NestJS): módulos `session`, `access`, `billing`, `analytics`, endpoint `/health`, más `apps/api/tsconfig.json` y `apps/api/nest-cli.json` con la configuración de build de § Design Decisions. Tests en **Vitest** (`apps/api/vitest.config.ts`, `*.spec.ts` junto al código, `test/*.e2e-spec.ts`) con `vitest`, `unplugin-swc` y `@swc/core`. En la raíz: `src/lib/api-client.ts` (nuevo), `src/app/chat/error.tsx` (nuevo), `scripts/check-access-bridge.ts` (nuevo), y `package.json` gana los scripts `test` y `check:access`. Se modifican `src/app/chat/page.tsx:26`, `src/app/api/chat/route.ts:39`, `src/lib/db.ts:26` (añade `max` explícito) y `src/app/api/t/route.ts:42` (deja de usar `AUTH_SECRET` como sal). `src/lib/access.ts` queda reducido a su declaración de tipos y `src/app/api/paddle/webhook/route.ts` se retira al final de la migración. `.env.example` añade `API_BASE_URL`, `ACCESS_VIA_API`, `AUTH_COOKIE_NAME`, `ACCESS_TIMEOUT_MS` y `ANALYTICS_SALT`. `drizzle.config.ts`, `src/lib/schema.ts`, `curriculum/` y los nueve `scripts/check-*.ts` existentes **no se tocan**. Segundo servicio en Railway. |

---

## 1. Problem Statement

[ADR-001](ADR-001-backend-nestjs-front-nextjs.md) decidió partir `platform` en `apps/web` (Next), `apps/api` (NestJS) y `packages/shared`. Esta es la primera fase, y su trabajo no es entregar capacidad de producto: es **descubrir si el puente de sesión funciona antes de que nada importante dependa de él**.

El ADR deja dos zonas marcadas como caras o irreversibles, y esta fase no entra en ninguna. La primera es `packages/shared`: `drizzle.config.ts:9` y seis checks de `scripts/` importan `src/lib/schema.ts` **con extensión**, un arreglo deliberado que depende de `allowImportingTsExtensions` (PRD-002 § 9); moverlo es el punto de no retorno y va en la última fase (ADR-001 § 7). La segunda es el stream del tutor: `src/app/api/chat/route.ts` construye un `ReadableStream` sobre `client.messages.stream` y es la ruta más sensible a latencia del producto (ADR-001 § 6).

Queda un dominio que sirve exactamente para esto. **Acceso y cobro** es pequeño y está cerrado sobre sí mismo:

- `src/lib/access.ts` son 123 LOC con tres funciones exportadas y exactamente tres puntos de llamada: `src/app/chat/page.tsx:26` (`getAccess`), `src/app/api/chat/route.ts:39` (`ensureTrial`) y `src/app/api/paddle/webhook/route.ts:52,72` (`setSubscriptionStatus`).
- Es dueño de una sola tabla, `subscriptions` (`src/lib/schema.ts:107`), y ninguna otra parte del código la escribe.
- El webhook de Paddle **no lleva sesión**: se autentica por firma sobre el body crudo. Puede mudarse sin depender del puente de auth, lo que da un camino de despliegue que se valida solo.
- No toca el stream: `ensureTrial` es una llamada JSON que ocurre **antes** de abrir el stream (`route.ts:39`, el stream se construye en la línea 102).

La pregunta que esta fase responde es concreta: hoy `auth()` es una llamada de función en el mismo proceso (`src/auth.ts:38`) y la sesión es un JWT en cookie (`src/auth.ts:53`). ¿Puede otro servicio verificar esa misma identidad por su cuenta, sin un segundo sistema de identidad y sin que `apps/web` tenga que ser creído? Si la respuesta es no, se descubre aquí, con 211 LOC en juego y un `git revert` de vuelta.

### 1.1 El servicio tiene que ser seguro estando expuesto

El repositorio es AGPL-3.0 y público, y la adoptabilidad es un problema declarado del proyecto (PRD-002 § 1, punto 4). Eso fija una restricción sobre esta fase: **ninguna propiedad de seguridad de `apps/api` puede depender de la topología de red del despliegue.**

Restringir `/v1/access*` a una red privada es endurecimiento legítimo donde la plataforma lo ofrezca, pero no puede ser lo que hace correcto al sistema: quien despliegue esto en un VPS, en una sola máquina o en una plataforma sin red interna se llevaría un sistema inseguro sin haber hecho nada mal, y sin que nada falle de forma visible. Por eso § 5 y § 8 especifican `apps/api` como si todos sus endpoints fueran alcanzables desde internet, que es el caso por defecto.

## 2. Goals

1. Desplegar `apps/api` como servicio NestJS, con `/health` respondiendo 200 sin tocar Postgres y su propio pool contra la misma base.
2. Cuando `apps/api` reciba una petición con un JWT de sesión de Auth.js válido, el sistema deberá resolver `userId` y `email` **sin consultar a `apps/web` ni a Postgres**.
3. Si el JWT falta, no verifica, está caducado, o su payload no trae `id` y `email` no vacíos, el sistema deberá responder **401 antes de abrir conexión a Postgres** — nunca 500.
4. La identidad deberá resolverse **exclusivamente** desde el token: si una petición suministra un `email` o un `userId` por cuerpo, query o ruta, el sistema deberá rechazarla o ignorar ese campo, nunca usarlo.
5. Si falta la configuración que el puente necesita para funcionar, el sistema deberá **fallar al arrancar**, no en la primera petición de un estudiante.
6. Mover las tres funciones de `src/lib/access.ts` a `apps/api` sin cambio observable en la frontera gratis/pago mientras `apps/api` responde: mismos estados, mismos días de trial, mismo momento de arranque de la prueba, mismo `email` sin transformar.
7. Mover el webhook de Paddle a `apps/api` conservando la verificación de firma sobre el **body crudo** y sus dos ramas de 400.
8. Si `apps/api` no responde dentro del timeout, el sistema deberá renderizar `/chat` igual y denegar el turno del tutor con 503, sin llamar a Anthropic (§ 5.3).
9. Dejar `drizzle.config.ts`, `src/lib/schema.ts`, `curriculum/` y los nueve `scripts/check-*.ts` sin modificar (ADR-001 § 7).
10. Si el puente falla en producción, el sistema deberá poder volver al camino anterior cambiando una variable de entorno, sin desplegar código.

**Excepción declarada al goal 6.** El estado degradado del goal 8 *sí* es un cambio observable: con `apps/api` caído, un estudiante de pago entra a `/chat` pero recibe 503 al escribir, donde hoy se le serviría. Es la consecuencia aceptada de no conceder acceso al tutor sin poder verificarlo, y se declara aquí en vez de esconderse detrás de "sin cambio observable".

## 3. Non-Goals

- **`packages/shared`.** Va en la última fase. En esta, `apps/api` importa el esquema desde la raíz por ruta relativa (§ Design Decisions).
- **Mover el stream del tutor.** `src/app/api/chat/route.ts` sigue en Next y sigue hablando con Anthropic desde ahí.
- **Navegador → `apps/api` directo.** Esta fase es solo servidor-a-servidor: no se llama `enableCors()` (§ 8). El navegador sigue hablando únicamente con Next. El estado final del ADR-001 sí contempla el camino directo; llega en una fase posterior.
- **Mover el repositorio Next a `apps/web`.** Deliberadamente aplazado (§ Design Decisions).
- **Rate limiting.** Riesgo aceptado, argumentado en § 8.
- **Worker, colas, trabajo diferido.** El ADR los contempla; no son esta fase.
- **Corregir que `customData.email` lo controle el cliente.** Defecto preexistente, nombrado en § 8 como riesgo heredado.
- **Currículo, conversaciones, Auth.js.** Auth.js sigue emitiendo la sesión desde Next; `apps/api` solo la verifica.

## 4. User Flows

Mientras `apps/api` responde, ningún flujo cambia para el estudiante. Lo que cambia es quién ejecuta cada paso — y aparece un cuarto flujo que hoy no existe.

```mermaid
sequenceDiagram
    participant B as Navegador
    participant W as apps/web (Next)
    participant A as apps/api (NestJS)
    participant P as Postgres
    participant PD as Paddle

    Note over B,PD: 1 — Entrar a /chat (no gasta la prueba)
    B->>W: GET /chat (cookie de sesión)
    W->>W: auth() + lee la cookie de sesión
    W->>A: GET /v1/access (Authorization: Bearer <jwt>)
    A->>A: getToken() → userId, email
    A->>P: SELECT subscriptions WHERE email
    A-->>W: 200 {allowed, status, trialDaysLeft}
    W-->>B: ChatClient o Paywall

    Note over B,PD: 2 — Primer mensaje al tutor (arranca el trial)
    B->>W: POST /api/chat
    W->>A: POST /v1/access/trial (Bearer)
    A->>P: INSERT ... ON CONFLICT DO NOTHING
    A-->>W: 200 {allowed:true, status:"trial", trialDaysLeft:7}
    W->>W: abre el stream a Anthropic (sin cambios)
    W-->>B: text/plain en streaming

    Note over B,PD: 3 — Paddle confirma el pago (sin sesión)
    PD->>A: POST /v1/webhooks/paddle (firma sobre body crudo)
    A->>A: unmarshal(rawBody, secret, firma)
    A->>P: UPSERT subscriptions
    A-->>PD: 200 "ok"

    Note over B,PD: 4 — apps/api no responde (§ 5.3)
    B->>W: GET /chat
    W-xA: GET /v1/access (timeout o 5xx)
    W-->>B: ChatClient igual (página abierta)
    B->>W: POST /api/chat
    W-xA: POST /v1/access/trial (timeout o 5xx)
    W-->>B: 503 — sin llamar a Anthropic, sin emitir telemetría
```

## 5. API

Todos los endpoints son **nuevos**, servidos por `apps/api`.

### 5.1 Autenticación

`apps/web` lee la cookie de sesión con `cookies()` de `next/headers` y la reenvía como `Authorization: Bearer <token>`. Resuelve el nombre probando `__Secure-authjs.session-token` y luego `authjs.session-token`; si encuentra un trozo `<nombre>.0`, lanza en vez de enviar un token truncado (§ 9, fila 35).

`apps/api` verifica con **`getToken()`** de `@auth/core/jwt`, no con `decode()` a pelo:

```ts
getToken({ req, secret: process.env.AUTH_SECRET, salt: process.env.AUTH_COOKIE_NAME })
```

**Por qué `getToken()` y no `decode()`**: `decode()` devuelve `null` únicamente si el token es falsy; ante secreto equivocado, salt equivocado, token corrupto o `exp` vencido, `jwtDecrypt` **lanza** y `decode` no lo captura. Especificarlo pelado convertiría en 500 los casos que el goal 3 exige que sean 401 — incluido el desajuste de salt, que es el fallo más probable de esta fase, y que dejaría de aparecer como la señal que § 10 paso 3 vigila. `getToken()` envuelve la decodificación en `try/catch`, devuelve `null` al fallar, lee la cabecera `Bearer` por sí mismo y reensambla cookies troceadas.

**`AUTH_COOKIE_NAME` es obligatoria y no tiene valor por defecto.** `apps/api` no arranca sin ella (§ 9, fila 3). El `salt` de Auth.js es el nombre de la cookie, y Auth.js elige el prefijo `__Secure-` según el **protocolo de la URL de la petición**, no según `NODE_ENV` — con `trustHost: true` (`src/auth.config.ts:15`) eso sale del `X-Forwarded-Proto` del proxy. `apps/api` recibe un Bearer y estructuralmente no puede ver ese protocolo, así que no puede replicar el criterio: cualquier valor por defecto sería una conjetura que acierta en los dos entornos de hoy y falla en `pnpm build && pnpm start` local o en un despliegue sin TLS. Es el mismo patrón que el repositorio ya aplica a `CURRICULUM_SLUG` (`.env.example`): obligatoria y sin defecto, para que un entorno mal configurado falle en vez de elegir en silencio.

**Validación de forma del payload.** `getToken()` valida `exp`, `nbf` y el tag AEAD; no valida la forma de los claims. El guard devuelve 401 salvo que el payload traiga un `id` string no vacío **y** un `email` string no vacío. Esto preserva una invariante que hoy imponen los call sites (`src/app/api/chat/route.ts:33`, `src/app/chat/page.tsx:20`) y que `docs/SYSTEM_ARTIFACT.md` registra en el dominio `acceso`; sin ella, un `email` ausente llegaría a `eq(subscriptions.email, undefined)`.

**La identidad sale del token y de ningún otro sitio.** `email` y `userId` se leen **exclusivamente** de `request.user`, que puebla `SessionGuard`. Ni `/v1/access` ni `/v1/access/trial` aceptan cuerpo, parámetros de query ni parámetros de ruta; la app se configura con `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true })`. Esto no es una precaución genérica: las funciones que se portan reciben el correo como argumento (`src/lib/access.ts:47`, `:86`), así que cablear `@Query('email')` hacia ellas es un error de una línea y del todo natural — y daría a cualquier estudiante con sesión válida lectura y escritura sobre la fila de suscripción de cualquier otro, siendo `subscriptions.email` la llave única (`src/lib/schema.ts:109`). La fila 19 de § 9 existe para eso.

`apps/api` **no acepta identidad por ninguna otra vía**: ni cabeceras `X-User-Id`, ni un secreto compartido que declare al usuario.

### 5.2 Endpoints

| Método | Ruta | Auth | Respuesta |
|---|---|---|---|
| `GET` | `/health` | — | `200 {"status":"ok"}` |
| `GET` | `/v1/access` | Bearer | `200 Access` · `401` |
| `POST` | `/v1/access/trial` | Bearer | `200 Access` · `401` |
| `POST` | `/v1/webhooks/paddle` | firma Paddle | `200 "ok"` · `400` · `413` |

`Access` es el tipo que hoy vive en `src/lib/access.ts:16`, sin cambios de forma:

```ts
type Access = {
  allowed: boolean;
  status: "none" | "trial" | "active" | "canceled";
  trialDaysLeft: number | null;
};
```

- `GET /v1/access` — solo lee (equivale a `getAccess`). Nunca crea trial.
- `POST /v1/access/trial` — crea el trial si no existe y devuelve el acceso (equivale a `ensureTrial`). **Idempotente en dos sentidos**: una segunda llamada del usuario no reinserta ni reemite `trial_started`, y un reintento del cliente tras un timeout tampoco — el árbitro es el `UNIQUE(email)` de `src/lib/schema.ts:109` y el `returning()` vacío del `onConflictDoNothing`, no el proceso que atiende.
- `POST /v1/webhooks/paddle` — cuerpo crudo, con **dos** ramas de 400: `"firma inválida"` si `paddle.webhooks.unmarshal` lanza, y `"sin evento"` si resuelve sin lanzar pero sin evento (`src/app/api/paddle/webhook/route.ts:39-41`). Las dos están declaradas como invariante viva en `docs/SYSTEM_ARTIFACT.md`; portar solo una sería deriva contra el documento vivo. Un error **posterior** a la verificación devuelve `200` para que Paddle no reintente en bucle, y se registra bajo las reglas de § 8. `413` si el cuerpo excede el límite declarado abajo.
- `/health` **no consulta Postgres**. Es el único endpoint sin token ni firma, y hacerle verificar la base entregaría a un llamante anónimo un viaje a Postgres, vaciando el goal 3.

**Límite de tamaño de cuerpo**: `64kb` explícito en la ruta del webhook. `rawBody: true` almacena el cuerpo entero **antes** de verificar la firma, en un endpoint que cualquiera alcanza; hoy estaría acotado en `100kb` por el defecto heredado de body-parser, o sea por accidente y no por decisión, y cualquier `useBodyParser` posterior lo ampliaría en silencio.

### 5.3 Contrato del cliente (`src/lib/api-client.ts`)

Nuevo módulo en la raíz. Es donde vive la política de degradación, y sin él la política no existe: **`fetch` en Node no tiene timeout por defecto**, así que un `apps/api` que acepta la conexión y no responde (reinicio de despliegue, pool agotado) colgaría el render de `/chat` indefinidamente en vez de degradar.

```ts
type ApiResult = Access | { error: true };
```

- **Timeout**: `AbortSignal.timeout(ACCESS_TIMEOUT_MS)`, por defecto `2000`.
- **Nunca lanza.** Timeout, conexión rechazada, 5xx, 401, JSON no parseable y 200 con cuerpo que no encaja en `Access` se mapean todos a `{ error: true }`.
- **Nunca coerciona a permitido.** Ninguna respuesta anómala puede producir `allowed: true`.

| Call site | Con `{ error: true }` |
|---|---|
| `src/app/chat/page.tsx:26` | Renderiza `ChatClient`, **no** `Paywall`, **no** redirección a `/signin`. Es la forma que el archivo ya tiene en su repliegue defensivo (`chat/page.tsx:20-22`), y lo que se ve son datos del propio usuario (`loadConversation`), sin fuga entre estudiantes. |
| `src/app/api/chat/route.ts:39` | Devuelve `503` antes de abrir el stream. **No** llama a Anthropic y **no** emite `tutor_message_sent` (`chat/route.ts:64`) — es el evento con el que § 10 paso 3 lee el embudo, y emitirlo en un turno denegado lo corrompería. |

**Un 401 de `apps/api` no es "el estudiante no tiene sesión".** Un desajuste de salt produce 401 para todo el mundo con sesión perfectamente válida; `src/middleware.ts:15-19` no lo detecta porque valida el JWT por su cuenta y nunca llama a `apps/api`. Redirigir a `/signin` en ese caso produciría un bucle: iniciar sesión, volver a `/chat`, mismo 401. Por eso el 401 cae en `{ error: true }` como cualquier otro fallo y nunca redirige.

**Sin reintento automático** en esta fase. El timeout ya acota la espera, y el reintento solo tendría sentido con backoff, que es complejidad que aún no se ha ganado.

## 6. Data Model

**No hay migración.** `subscriptions` (`src/lib/schema.ts:107-115`) queda exactamente como está: `id`, `email` (único), `status`, `trial_ends_at`, `paddle_subscription_id`, `created_at`, `updated_at`. Ningún `drizzle-kit generate` en esta fase, y `drizzle.config.ts` no se toca.

**El `email` no se transforma.** Hoy la llave llega sin normalizar por el camino del tutor (`src/app/api/chat/route.ts:32` toma `session.user.email` tal cual) mientras el webhook sí normaliza (`webhook/route.ts:25`, `.toLowerCase()`) y el registro también (`src/auth.ts:70`). Esa asimetría existe en producción hoy. `apps/api` leerá el `email` del token, que es el mismo valor (`src/auth.ts:93-98` copia el token a la sesión), y **lo usará sin transformar**: añadir un `.toLowerCase()` al portar es exactamente lo que haría un implementador razonable al ver la asimetría, y sería un cambio de comportamiento observable que merece su propio PRD. La fila 18 de § 9 fija la paridad.

**Conexiones.** Aparece un segundo pool de `pg` contra la misma base. `src/lib/db.ts:26` abre hoy `new Pool({ connectionString })` **sin `max`**, o sea el default de `pg` (10). Esta fase fija `max` explícito en los dos servicios y reserva margen para los otros consumidores, que existen y no son ninguno de esos pools: `drizzle-kit migrate` y los scripts de `scripts/` (`check-curriculum-load.ts` abre su propia conexión).

| Consumidor | `max` |
|---|---|
| Next (`src/lib/db.ts`) | 8 |
| `apps/api` | 8 |
| Margen para migraciones y scripts | 4 |

El paso 1 de § 10 verifica esa suma contra el límite de conexiones del plugin de Postgres de Railway **antes** de continuar, y ajusta si el límite real es menor.

**Propiedad.** Al terminar la fase, `subscriptions` la escribe solo `apps/api`. Durante la migración la escriben los dos (§ 10); el `onConflictDoUpdate` de `setSubscriptionStatus` y el `onConflictDoNothing` de `ensureTrial` ya hacen esa convivencia segura — mismas sentencias, misma fila única por `email`.

## 7. Architecture

```mermaid
flowchart TB
    subgraph repo["repositorio platform"]
        subgraph root["raíz — Next.js (sin mover)"]
            page["src/app/chat/page.tsx"]
            chat["src/app/api/chat/route.ts"]
            client["src/lib/api-client.ts (nuevo)<br/>timeout + Access | {error:true}"]
            schema["src/lib/schema.ts"]
            dcfg["drizzle.config.ts + scripts/"]
        end
        subgraph api["apps/api — NestJS (nuevo)"]
            guard["SessionGuard<br/>getToken() de @auth/core/jwt"]
            access["AccessModule"]
            billing["BillingModule<br/>webhook Paddle"]
            analytics["AnalyticsService<br/>provider inyectable"]
            db2["DrizzleModule (pool propio)"]
        end
    end
    pg[(Postgres)]
    paddle[Paddle]
    anthropic[Anthropic]

    page --> client
    chat --> client
    client -->|"Bearer JWT"| guard
    guard --> access
    paddle -->|firma| billing
    access --> db2
    billing --> db2
    access --> analytics
    billing --> analytics
    db2 --> pg
    dcfg --> schema
    db2 -.->|"import temporal por ruta relativa"| schema
    chat --> anthropic
```

Cuatro piezas nuevas en `apps/api`:

- **`SessionGuard`** — `CanActivate` que resuelve el Bearer con `getToken()`, valida la forma del payload y pone `{ userId, email }` en el request. Es la única puerta de entrada de identidad.
- **`AccessModule`** — las tres funciones portadas desde `src/lib/access.ts`, incluida la relectura tras la carrera del `onConflictDoNothing` (`access.ts:112-119`), que se conserva porque el escenario no desaparece al mudarse.
- **`BillingModule`** — el webhook. `NestFactory.create(AppModule, { rawBody: true })`, y **`req.rawBody` es un `Buffer`**: `unmarshal` recibe hoy un `string` (`webhook/route.ts:31,35`, vía `req.text()`), así que hace falta `.toString("utf8")` explícito o la firma no verifica nunca.
- **`AnalyticsService`** — provider inyectable de NestJS envolviendo el mismo `posthog-node`. **No** se porta `src/lib/analytics.ts` tal cual: hoy construye el cliente en el ámbito del módulo y exporta una función suelta (`analytics.ts:19-23`), que no se puede inyectar ni espiar desde `Test.createTestingModule`. Sin esto, la fila 21 de § 9 solo podría afirmar "una fila" y no "un evento", que es la mitad interesante.

## 8. Security

**El puente de sesión es la superficie nueva, y § 1.1 fija que debe ser segura estando expuesta.**

1. **`apps/api` nunca confía en `apps/web`.** No existe cabecera de identidad ni secreto compartido que sustituya al token: la identidad sale de `getToken()` y de la validación de forma de § 5.1, o la petición es 401. Ni `email` ni `userId` se leen de cuerpo, query o ruta (§ 5.1).
2. **401 antes de Postgres** (goal 3). El guard corre antes que el servicio, así que tráfico sin token no genera carga de base.
3. **CORS deshabilitado explícitamente.** No se llama `enableCors()` en esta fase. NestJS ya viene así por defecto, pero se declara aquí para que sea una invariante revisable y no una omisión.
4. **`AUTH_SECRET` pasa a vivir en dos servicios**, y su radio de explosión es mayor de lo que parece: hoy es también la sal del píxel anónimo de páginas públicas (`src/app/api/t/route.ts:42`), cuyo argumento de "no requiere consentimiento" se apoya en que el hash no sea enlazable. Quien tenga el secreto lo rompe por fuerza bruta sobre (IP × UA × día). **Esta fase le da al píxel su propia `ANALYTICS_SALT`** — cambio de una línea, sin impacto de esquema — para que `apps/api` nunca necesite un secreto que hace de control de privacidad. Se registra además que con `session: { strategy: "jwt" }` (`src/auth.ts:53`) no hay revocación por token: rotar `AUTH_SECRET` es el único interruptor, y obliga a redesplegar los dos servicios a la vez.
5. **Uso a sabiendas fuera de etiqueta.** `@auth/core/jwt` advierte upstream: *"Auth.js JWTs are meant to be used by the same app that issued them. If you need JWT authentication for your third-party API, you should rely on your Identity Provider instead."* Se asume conscientemente: `apps/api` no es un tercero, es el mismo producto partido en dos procesos. Queda escrito para que el siguiente lector no lo redescubra como sorpresa.

**Reglas de registro.** Dos cosas no pueden llegar nunca a los logs:

- **El correo.** El catch del webhook registra únicamente `err.name` y `(err as { cause?: { code?: string } }).cause?.code` — **nunca** `err.message`, `err.stack`, ni el objeto de error. Esto corrige una afirmación falsa de la versión anterior de este PRD: `DrizzleQueryError` embebe los parámetros ligados dentro del mensaje (`drizzle-orm@0.45.2/errors.js`, `super(\`Failed query: ${query}\nparams: ${params}\`)`), y `params` incluye el correo. El `console.error` de hoy (`src/app/api/paddle/webhook/route.ts:84`) sí filtra el correo a los logs de Railway; la propiedad que había que "conservar" no existía. Fila 31 de § 9.
- **El token.** El Bearer reenviado **es** la credencial de sesión, con ~30 días de vida y sin revocación individual. Mover una credencial de una cookie (que casi nada registra) a una cabecera `Authorization` (que los loggers de petición capturan de rutina) es un cambio real de exposición. Ningún interceptor, filtro ni manejador de error serializa cabeceras de petición; si se añade un interceptor de logging, `authorization` se redacta. Fila 12 de § 9.

**Diagnóstico de los 401.** El paso 3 de § 10 vigila la tasa de 401 y § 5.1 nombra el desajuste de salt como el fallo más probable, pero un 401 por salt, por token caducado, por cabecera ausente y por secreto equivocado son hoy indistinguibles. `SessionGuard` **registra** (no devuelve al cliente) un código de razón: `missing_header | malformed | decode_failed | missing_claims`. Sin él, "vigilar la tasa de 401" es mirar un agregado sin poder atribuir un pico.

**Secretos de Paddle.** `apps/api` recibe **solo `PADDLE_WEBHOOK_SECRET`**. `PADDLE_API_KEY` no se copia: `unmarshal` usa únicamente el secreto del webhook, y el código de hoy ya corre con `process.env.PADDLE_API_KEY ?? ""` (`webhook/route.ts:13`), o sea que una clave vacía funciona para este handler. Es una credencial de servidor capaz de cancelar suscripciones y emitir reembolsos; replicarla para nada amplía el radio de explosión sin comprarlo. Si una fase posterior necesita la API de Paddle desde `apps/api`, se aprovisiona entonces. Los dos servicios **nunca comparten secreto de firma** — ver § 10 paso 4.

**PII.** El `email` es la llave de `subscriptions` (`schema.ts:109`) y ahora viaja por red entre servicios. Solo sobre TLS, nunca en query string, y bajo las reglas de registro de arriba.

**Riesgo aceptado: sin rate limiting.** El coste de CPU de una petición no autenticada es real y bajo — `getToken()` es HKDF-SHA256, un thumbprint SHA-512 sobre un JWK pequeño y un descifrado A256CBC-HS512: microsegundos, sin primitiva de hashing de contraseñas y sin E/S. Lo que la versión anterior de este PRD no decía es la otra mitad: con la política de "tutor cerrado" de § 5.3, **`apps/api` pasa a ser punto único de fallo del producto de pago**, alcanzable desde internet y sin cota de peticiones. Degradarlo deniega el tutor a todos los estudiantes que pagan. Se acepta igualmente para esta fase —el coste por petición no da apalancamiento a un atacante— y la cota de tamaño de cuerpo de § 5.2 cierra la mitad de memoria. Si aparece tráfico no autenticado sostenido, se añade entonces.

**Riesgo heredado: la llave de fila del webhook la suministra el cliente.** `customData.email` se origina en el navegador (`src/app/paywall.tsx:46`, componente `"use client"`), y el webhook lo cree como llave de fila para un **upsert** (`access.ts:78-81`). Un atacante que abra un checkout con el correo de una víctima, se suscriba y cancele, dispara `setSubscriptionStatus(victima, "canceled")` y la encierra tras el muro de pago; coste, un periodo de suscripción. **Este PRD no introduce el defecto y no lo arregla** — pero § 8 es el primer modelo de amenazas escrito del servicio que § 6 convierte en dueño único de `subscriptions`, y callarlo aquí sería perderlo. La remediación barata, para el PRD de seguimiento: contrastar `customData.email` contra el correo del cliente Paddle del evento antes de escribir.

**Cambio de superficie aceptado.** Hoy `ensureTrial` es privada y solo se invoca desde `POST /api/chat`; esa restricción de call site es lo que hace cierta la invariante *"el trial arranca con el primer mensaje al tutor"* (`access.ts:1-2`, y el dominio `acceso` de `docs/SYSTEM_ARTIFACT.md`). Tras esta fase, cualquiera con sesión válida puede arrancar su propio trial con un `curl` a `/v1/access/trial` sin escribirle nunca al tutor. El daño directo es nulo —quema su propia prueba— pero desacopla `trial_started` de `tutor_message_sent`, que es el embudo que § 10 paso 3 usa. La invariante pasa de "garantizada por el call site" a "garantizada porque solo `apps/web` llama al endpoint", y `SYSTEM_ARTIFACT.md` se actualiza en el mismo commit del gate.

## 9. Test Plan

`apps/api` usa **Vitest** con `unplugin-swc` (justificación en § Design Decisions). Unitarios en `*.spec.ts` junto al código; de integración en `apps/api/test/*.e2e-spec.ts` con `Test.createTestingModule` de `@nestjs/testing`.

- **`unplugin-swc` no es opcional.** El transformador por defecto de Vitest (esbuild) no emite metadatos de decorador y la DI de NestJS los necesita: sin el plugin, `Test.createTestingModule` no resuelve los providers. La fila 2 vigila esa configuración.
- **Cada fichero de test declara en su cabecera qué filas de este § 9 cubre**, siguiendo la convención que el repositorio ya usa (`scripts/check-schedule.ts:6`: *"Cubre las filas 21 y 23 de PRD-002 §9"*). Es lo que hace barata la verificación fila por fila de `workflow.md` paso 9.
- **Comandos**: `package.json` gana `"test": "pnpm --filter api test"` y `"check:access": "node scripts/check-access-bridge.ts"`. Sin esto, un check nuevo en un repositorio sin CI se ejecuta a mano una vez y nunca más, y dos filas del gate apuntarían a tests que nadie corre.
- Las filas del lado Next (34-39) se quedan en el estilo de checks de la raíz (`node scripts/check-*.ts` con `node:assert/strict`). Unificar los **nueve** `check-*.ts` existentes en Vitest es un cambio aparte y no entra en este PRD.
- Las filas 13-18 mockean el repositorio vía override de provider en `Test.createTestingModule`; no tocan Postgres. Las filas de `test/*.e2e-spec.ts` sí usan base real.

| # | Test | Type | Description | Path |
|---|---|---|---|---|
| 1 | `apps/api` compila y arranca | Regresión | `pnpm --filter api build` seguido de arrancar el entrypoint emitido y pedir `/health`. Cubre el import cruzado a `src/lib/schema.ts` por el camino de **build**, no de test — que es donde § Design Decisions demuestra que se rompe. | `../platform/apps/api/test/build-boot.e2e-spec.ts` |
| 2 | La DI de NestJS resuelve bajo Vitest | Regresión | `Test.createTestingModule` instancia `AccessService` con sus dependencias. Falla si se pierde `unplugin-swc` o `emitDecoratorMetadata`. | `../platform/apps/api/test/build-boot.e2e-spec.ts` |
| 3 | Sin `AUTH_COOKIE_NAME`, `apps/api` no arranca | Regresión | Goal 5: la configuración que el puente necesita falla al arrancar, no en la primera petición. | `../platform/apps/api/test/build-boot.e2e-spec.ts` |
| 4 | JWT válido resuelve identidad | Unit | `getToken()` con secreto y salt correctos devuelve `id` y `email`; el guard los expone en `request.user`. | `../platform/apps/api/src/session/session.guard.spec.ts` |
| 5 | Sin cabecera `Authorization` → 401 | Unit | Rechaza antes de instanciar el servicio de acceso. | `../platform/apps/api/src/session/session.guard.spec.ts` |
| 6 | Secreto incorrecto → **401, no 500** | Regresión | Token firmado con otro `AUTH_SECRET`. Falla si alguien sustituye `getToken()` por `decode()` pelado. | `../platform/apps/api/src/session/session.guard.spec.ts` |
| 7 | Token caducado → **401, no 500** | Regresión | `exp` en el pasado. Mismo motivo que la fila 6. | `../platform/apps/api/src/session/session.guard.spec.ts` |
| 8 | Salt equivocado → **401, no 500** | Regresión | Token emitido con salt `__Secure-authjs.session-token`, verificado con `authjs.session-token`. Es el fallo de despliegue más probable (§ 5.1) y la señal que § 10 paso 3 vigila. | `../platform/apps/api/src/session/session.guard.spec.ts` |
| 9 | Token válido sin `email` → 401 | Edge | Preserva la invariante de `chat/route.ts:33`; sin ella llegaría `eq(subscriptions.email, undefined)`. | `../platform/apps/api/src/session/session.guard.spec.ts` |
| 10 | Token válido sin `id` → 401 | Edge | Misma invariante, otra mitad. | `../platform/apps/api/src/session/session.guard.spec.ts` |
| 11 | 401 no abre conexión a Postgres | Integración | Con `DATABASE_URL` a un puerto muerto, una petición sin token sigue devolviendo 401 y no lanza. Goal 3. | `../platform/apps/api/test/access.e2e-spec.ts` |
| 12 | El log de un 401 lleva razón y no lleva el token | Regresión | La línea emitida contiene uno de los cuatro códigos de § 8 y no contiene `eyJ`. | `../platform/apps/api/src/session/session.guard.spec.ts` |
| 13 | Sin fila → `{allowed:true, status:"none"}` | Unit | Paridad con `access.ts:56`. Entrar a mirar no gasta la prueba. | `../platform/apps/api/src/access/access.service.spec.ts` |
| 14 | Trial vigente devuelve días restantes | Unit | `Math.ceil` sobre la diferencia; paridad con `access.ts:34`. | `../platform/apps/api/src/access/access.service.spec.ts` |
| 15 | Trial vencido → `allowed:false` | Edge | `trialEndsAt` en el pasado. | `../platform/apps/api/src/access/access.service.spec.ts` |
| 16 | `canceled` → `allowed:false` | Edge | Paridad con `access.ts:41`. | `../platform/apps/api/src/access/access.service.spec.ts` |
| 17 | `active` → `allowed:true`, `trialDaysLeft:null` | Unit | Paridad con `access.ts:26`. | `../platform/apps/api/src/access/access.service.spec.ts` |
| 18 | El `email` del token se usa sin transformar | Regresión | `Estudiante@X.com` no se pasa a minúsculas. Fija la paridad de § 6 contra el `.toLowerCase()` que un implementador añadiría al ver la asimetría con el webhook. | `../platform/apps/api/src/access/access.service.spec.ts` |
| 19 | Un `email` suministrado por el llamante se rechaza | Regresión | `GET /v1/access?email=<otro>` y `POST /v1/access/trial {"email":"<otro>"}` resuelven ambos contra el correo del token, o devuelven 400 por `forbidNonWhitelisted`. **Nunca leen la fila del otro.** Goal 4. | `../platform/apps/api/test/access.e2e-spec.ts` |
| 20 | `GET /v1/access` nunca crea trial | Regresión | Tras la llamada no hay fila en `subscriptions`. La frontera del producto depende de esto. | `../platform/apps/api/test/access.e2e-spec.ts` |
| 21 | `POST /v1/access/trial` concurrente: una fila, un evento | Integración | Dos llamadas a la vez. Cubre la carrera de `access.ts:104-119`. El "un evento" solo es afirmable con `AnalyticsService` inyectable (§ 7). | `../platform/apps/api/test/access.e2e-spec.ts` |
| 22 | `POST /v1/access/trial` secuencial no reinserta ni reemite | Regresión | Con fila existente, la segunda llamada se salta el bloque `access.ts:95-120` entero. Cubre también el reintento del cliente tras timeout. | `../platform/apps/api/test/access.e2e-spec.ts` |
| 23 | `GET /health` → 200 y no toca Postgres | Smoke | Healthcheck de Railway; § 5.2 prohíbe que consulte la base. | `../platform/apps/api/test/access.e2e-spec.ts` |
| 24 | Webhook con firma inválida → 400 | Error | `unmarshal` lanza; no se toca la base. | `../platform/apps/api/test/billing.e2e-spec.ts` |
| 25 | Webhook sin evento → 400 | Error | `unmarshal` resuelve sin lanzar pero sin evento (`webhook/route.ts:39-41`). Invariante viva en `SYSTEM_ARTIFACT.md`. | `../platform/apps/api/test/billing.e2e-spec.ts` |
| 26 | Body crudo preservado bajo NestJS | Regresión | Con `rawBody: true` y `.toString("utf8")`, la firma de un cuerpo real verifica. Es la regresión que introduce el cambio de framework. | `../platform/apps/api/test/billing.e2e-spec.ts` |
| 27 | `SubscriptionActivated` sin fila previa hace UPSERT | Edge | El caso que motivó el upsert (`access.ts:61-64`): checkout público, sin paso previo por el tutor. | `../platform/apps/api/test/billing.e2e-spec.ts` |
| 28 | `SubscriptionCanceled` → `canceled` sin pisar el `paddle_subscription_id` | Edge | Paridad con `access.ts:73-75`. | `../platform/apps/api/test/billing.e2e-spec.ts` |
| 29 | Error posterior a la firma devuelve 200 | Error | Paddle no debe reintentar en bucle. Paridad con `webhook/route.ts:81-87`. | `../platform/apps/api/test/billing.e2e-spec.ts` |
| 30 | Firma con `ts` 10 s en el pasado → 400 | Edge | Fija la ventana de frescura de 5 s del SDK y hace explícito el modo de fallo por desfase de reloj (§ 10 paso 4). | `../platform/apps/api/test/billing.e2e-spec.ts` |
| 31 | Un fallo de base no filtra el correo al log | Regresión | Forzando un error de Drizzle, la salida capturada no contiene `@`. Cierra la fuga descrita en § 8. | `../platform/apps/api/test/billing.e2e-spec.ts` |
| 32 | Cuerpo por encima de `64kb` → 413 | Edge | La cota de § 5.2, antes de verificar firma. | `../platform/apps/api/test/billing.e2e-spec.ts` |
| 33 | Email del webhook sí se normaliza | Regresión | `.toLowerCase()` de `webhook/route.ts:25` se conserva — la asimetría con la fila 18 es deliberada y preexistente. | `../platform/apps/api/test/billing.e2e-spec.ts` |
| 34 | `ACCESS_VIA_API=false` usa el camino antiguo | Regresión | El rollback por variable de entorno (goal 10) funciona sin desplegar. | `../platform/scripts/check-access-bridge.ts` |
| 35 | Cookie de sesión troceada falla ruidosamente | Error | Si aparece `<cookie>.0`, el cliente lanza en vez de enviar un token truncado, que daría un 401 indistinguible de un fallo de secreto. | `../platform/scripts/check-access-bridge.ts` |
| 36 | Timeout → página abierta, tutor cerrado | Regresión | `apps/api` que acepta la conexión y no responde: `/chat` renderiza `ChatClient` y `POST /api/chat` devuelve 503 sin llamar a Anthropic. **Es la fila que demuestra que la política de § 5.3 existe**; sin `AbortSignal.timeout` el render se cuelga y no llega a la rama de error. | `../platform/scripts/check-access-bridge.ts` |
| 37 | 200 con cuerpo malformado no permite el paso | Regresión | Ninguna respuesta anómala puede coercionarse a `allowed: true`. | `../platform/scripts/check-access-bridge.ts` |
| 38 | Un 503 no emite `tutor_message_sent` | Regresión | El turno denegado no puede contaminar el embudo que § 10 paso 3 lee. | `../platform/scripts/check-access-bridge.ts` |
| 39 | Un 401 de `apps/api` no redirige a `/signin` | Regresión | Evita el bucle de login descrito en § 5.3, que afectaría a todos los usuarios a la vez durante un desajuste de salt. | `../platform/scripts/check-access-bridge.ts` |

## 10. Migration Plan

Cinco pasos. Los cuatro primeros son reversibles sin desplegar código; el paso 1 tiene una excepción declarada.

1. **Levantar `apps/api` sin tráfico.** `pnpm-workspace.yaml` gana `packages: ["apps/*"]` y aprueba en `allowBuilds` los scripts de build que traen `@swc/core` y compañía (pnpm 11 los bloquea si no). Se crea `apps/api` con la configuración de build de § Design Decisions; segundo servicio en Railway con `DATABASE_URL`, `AUTH_SECRET`, `AUTH_COOKIE_NAME`, `PADDLE_WEBHOOK_SECRET` y `POSTHOG_*`. Se verifica `/health` y se comprueba la suma de `max` de § 6 contra el límite real del plugin de Postgres.

   ⚠️ **Este paso no es inerte para el servicio Next.** Declarar `packages:` cambia el grafo de instalación de la raíz, que *es* el servicio Next en producción: `pnpm install` pasa a instalar también NestJS, Vitest y `@swc/core`, con su coste de tiempo y tamaño de build. Y Railway despliega al mergear a `main`, así que sale a producción cuando se mergea, no cuando el operador lo decida. Hay que aislar el build del servicio Next (`pnpm install --filter`) o aceptar el crecimiento conscientemente.
   *Rollback: borrar el servicio **y revertir el commit que declara `packages:`**.*

2. **Cablear `apps/web` detrás de un flag.** `src/lib/api-client.ts` con el contrato de § 5.3, `src/app/chat/error.tsx`, y `ACCESS_VIA_API` por defecto `false`. Con el flag apagado, `chat/page.tsx` y `api/chat/route.ts` siguen importando `src/lib/access.ts` exactamente como hoy. Se despliega apagado: cero cambio de comportamiento. *Rollback: ya está apagado.*

3. **Encender el flag.** `ACCESS_VIA_API=true` en el servicio Next. Aquí se prueba el puente de verdad. Se vigilan: la tasa de 401 en `apps/api` **desglosada por el código de razón de § 8** (un desajuste de salt la pone al 100 % con `decode_failed`), el embudo `trial_started` / `tutor_message_sent` en PostHog, y la latencia de `/chat` y del primer token de `/api/chat`. *Rollback: `ACCESS_VIA_API=false`. No es un despliegue de código, pero sí reinicia el servicio Next — no es gratis.*

4. **Segundo destino de notificación en Paddle.** Se **crea** un destino nuevo apuntando a `apps/api`, en vez de editar la URL del existente. Paddle da a cada destino su propio secreto de firma, así que los dos servicios nunca comparten `PADDLE_WEBHOOK_SECRET` — y el rollback es deshabilitar el destino nuevo, no reeditar una URL. Durante el soak los dos endpoints reciben cada evento: la escritura a `subscriptions` es el mismo upsert idempotente contra la misma fila única, así que el estado converge.

   **Para no duplicar telemetría**, la ruta de Next deja de emitir `track()` al crear el destino nuevo (sigue escribiendo a Postgres). `track()` no deduplica, y sin esto cada evento de Paddle produciría dos `subscription_activated`, corrompiendo justo el embudo que el paso 3 vigila.

   **Señales antes de continuar**: la tasa de 400 en `/v1/webhooks/paddle` y el contador de entregas fallidas del panel de Paddle. El SDK impone una ventana de frescura de 5 segundos sobre el `ts` de la firma, así que un reloj adelantado en el contenedor nuevo hace fallar **todos** los webhooks con un 400 indistinguible de un secreto equivocado — y los suscriptores que pagan nunca reciben `status: "active"`, en silencio. *Rollback: deshabilitar el destino nuevo.*

5. **Soak y limpieza.** Tras 7 días —una vuelta completa del ciclo de trial— con las señales limpias **y confirmación de que el destino viejo lleva cero entregas**: se retira el destino viejo de Paddle, se borra `src/app/api/paddle/webhook/route.ts`, `src/lib/access.ts` queda reducido a su declaración de tipos (§ Design Decisions), se retira `PADDLE_WEBHOOK_SECRET` del servicio Next y se elimina el flag `ACCESS_VIA_API` con su rama muerta. El criterio de salida es la evidencia de cero entregas, no el paso del tiempo. *Este paso sí exige `git revert` para deshacerse.*

**Sin backfill de datos**: no hay migración de esquema y la tabla no se mueve de base.

## 11. Open Questions

- [ ] **Cookies troceadas.** Auth.js parte la cookie de sesión cuando excede el límite del navegador. Nuestro JWT lleva `id`, `email`, `name` y `exp`, así que no debería trocearse, y `getToken()` (§ 5.1) ya reensambla los trozos del lado de `apps/api`. Del lado de `apps/web`, la fila 35 de § 9 hace que el caso falle ruidosamente. **Diferido**: reensamblar en `api-client.ts` hasta que se observe uno.
- [ ] **`AUTH_SECRET` compartido.** Esta fase lo replica en los dos servicios. **Diferido hasta que haya un tercer consumidor**, pero el coste de salir no es el que decía la versión anterior de este PRD: el token de sesión de Auth.js es un **JWE** cifrado con `dir` + `A256CBC-HS512` y clave derivada por HKDF del secreto, no un JWT firmado — no hay clave pública que publicar. Pasar a JWKS obliga a reemplazar cómo `apps/web` **emite** el token (un override `jwt: { encode, decode }` sobre `src/auth.ts:38`), cambia lo que valida `src/middleware.ts:13` —dos ficheros que este PRD deliberadamente no toca— e invalida toda sesión viva en el corte. Diferir sigue siendo lo correcto; la estimación baja era como los aplazamientos se vuelven permanentes.

---

## Design Decisions

**Cómo se construye `apps/api` — y por qué la versión anterior de este PRD no compilaba.** El import relativo a `src/lib/schema.ts` estaba justificado solo por el camino de **tests** ("Vitest resuelve TypeScript nativamente"). El camino de **build** se rompe, y se comprobó con el `typescript@5.9.3` del repositorio sobre la misma topología:

- Con `allowImportingTsExtensions` y emit: `error TS5096: Option 'allowImportingTsExtensions' can only be used when either 'noEmit' or 'emitDeclarationOnly' is set`. La raíz puede permitírselo porque `tsconfig.json:8` tiene `"noEmit": true` — y su propio comentario (`tsconfig.json:21`) documenta esa dependencia. `apps/api` **tiene** que emitir: NestJS necesita `emitDecoratorMetadata`, y `--experimental-strip-types` de Node no lo sustituye.
- Añadiendo `rewriteRelativeImportExtensions` con `rootDir: "./src"`: `error TS6059: File '…/src/lib/schema.ts' is not under 'rootDir'`.

La configuración que sí funciona, y que esta fase fija:

- `apps/api/tsconfig.json`: `allowImportingTsExtensions` **y** `rewriteRelativeImportExtensions`, **sin** `rootDir`. Compila, pero el `rootDir` inferido pasa a ser la raíz del repositorio y el árbol emitido es `dist/apps/api/src/main.js` más `dist/src/lib/schema.js`.
- `nest-cli.json` con `entryFile` apuntando a `apps/api/src/main`, y el arranque en Railway a `node dist/apps/api/src/main.js` — **no** `dist/main.js`, que es lo que asumen los defaults y lo que nadie descubriría hasta el primer arranque.
- El build es `tsc` mientras exista el import cruzado. Si se pasa a `nest build -b swc`, hay que verificar antes que SWC reescribe `.ts` → `.js` en el especificador emitido; si no lo hace, el `require(".../schema.ts")` resultante muere en runtime con `MODULE_NOT_FOUND`.
- `apps/api/package.json` fija `drizzle-orm` y `pg` a la **misma versión exacta** que la raíz (`drizzle-orm@0.45.2`, `pg@8.21.0`), vía `catalog:` de pnpm 11. Con versiones distintas se resuelven dos instancias de `drizzle-orm` y `drizzle(pool, { schema })` deja de reconocer las tablas. Igual con `@auth/core@0.41.2` **exacto**, no `^`: la derivación de clave depende de la cadena literal del HKDF, y un minor que la cambie daría 401 en todas las sesiones sin que nada más cambie.

La fila 1 de § 9 vigila todo esto por el camino de build; la fila 2 vigila el de test, que es otro transformador y no cubre lo anterior.

**Next se queda en la raíz; no se mueve a `apps/web` en esta fase.** El estado final del ADR-001 tiene tres paquetes, pero mover el repositorio Next entero toca `tsconfig.json`, `drizzle.config.ts:9`, las rutas de `scripts/`, la configuración de Railway y `next.config.mjs` — todo lo que ADR-001 § 7 marca como caro de revertir, y nada necesario para responder la pregunta de esta fase. Se aplaza a la fase que crea `packages/shared`, donde esos ficheros hay que tocar de todas formas.

**`apps/api` importa `src/lib/schema.ts` por ruta relativa, temporalmente.** Sin `packages/shared`, `apps/api` alcanza el esquema subiendo a la raíz. Es deuda declarada, marcada en el código con `// ponytail: import temporal a la raíz; lo cierra la fase de packages/shared, ver ADR-001 §7`. La alternativa —duplicar el esquema— introduce deriva entre dos definiciones de la misma tabla, que es el fallo que `packages/shared` existe para evitar.

**`src/lib/access.ts` sobrevive al paso 5, reducido a tipos.** El tipo `Access` se ancla hoy a un archivo que el paso 5 borraba, dejando a `apps/api` con una definición y a `api-client.ts` con otra — la deriva de tipos que ADR-001 § 2 encarga a `packages/shared`. Y con el union `Access | { error: true }` de § 5.3 son dos formas que mantener sincronizadas, no una. Lo barato: el archivo queda con la declaración de tipos y nada más —cero lógica, cero import de `db`— marcado con el mismo comentario `ponytail:`, y se borra en la fase de `packages/shared`.

**El puente reenvía el JWT en vez de que Next declare al usuario.** Un secreto compartido más una cabecera `X-User-Id` sería menos código, pero convertiría a `apps/api` en un servicio que confía en su cliente, y § 1.1 fija que debe ser seguro estando expuesto. Reenviar el token cuesta unas líneas y da la propiedad que el estado final necesita. Es además el mismo camino de código que usará el navegador en una fase posterior, así que se paga una vez.

**Vitest, no Jest, y no el estilo de checks de la raíz.** NestJS 11 trae Jest preconfigurado, pero el roadmap de NestJS v12 ([PR #16391](https://github.com/nestjs/nest/pull/16391), en draft, objetivo *early Q3 2026*) mueve todos los repositorios y ejemplos de Jest a Vitest, y hace que los proyectos ESM del CLI usen Vitest por defecto; Jest queda para CJS. `apps/api` nace hoy y puede nacer ESM declarando su propio `"type": "module"`.

*Corrección respecto a la versión anterior*: la raíz **no es ESM**. `package.json` no tiene campo `"type"`, así que Node la trata como CommonJS; `"module": "esnext"` y `"moduleResolution": "bundler"` (`tsconfig.json:10-11`) son ajustes para el bundler de Next, y `next.config.mjs` lleva `.mjs` precisamente porque el paquete no es ESM. La elección de Vitest se sostiene por el roadmap de NestJS y por `unplugin-swc`, no por una herencia que no existe. Que `apps/api` sea ESM es decisión suya, y añade su propio roce con la configuración de build de arriba.

**Lo que se paga por Vitest hoy**: la PR de v12 está en draft, así que `nest new` sigue generando Jest y el `vitest.config.ts` se escribe a mano. El punto delicado es `unplugin-swc`: esbuild no emite metadatos de decorador y sin ellos la DI de NestJS no resuelve nada. También significa **dos estilos de test en el repositorio** —Vitest en `apps/api`, checks de `node:assert` en la raíz— hasta que alguien unifique; se prefiere eso a migrar nueve checks que hoy funcionan dentro de un PRD que va de otra cosa.

**La red privada es endurecimiento, no diseño.** Donde la plataforma lo ofrezca, restringir `/v1/access*` a la red interna es una mejora legítima y se recomienda. Pero § 1.1 fija que no puede ser lo que hace correcto al sistema: el repositorio es público y adoptable, y una propiedad de seguridad que depende de la topología de despliegue falla en silencio para quien despliegue en otro sitio. Por eso todos los controles de § 5.1 y § 8 asumen que los tres endpoints son alcanzables desde internet.

---

## Gate: Promotion to Implemented

```yaml
commit_hash: [TBD]
tests:
  - [TBD]
system_artifact_diff:
  - [TBD]
```
