# De Vuelta — Guía para Claude Code

**Lee `CONTEXT.md` antes de cualquier tarea.** Es la fuente de verdad del proyecto
(fases, decisiones, esquema). Este archivo cubre lo operativo: cómo está armado el
código, cómo se trabaja en él y qué reglas no se rompen.

## Qué es

PWA hiperlocal para reunir mascotas perdidas con sus dueños en la Alcaldía
Benito Juárez, CDMX. Next.js 15 (App Router) + React 19 + Supabase
(Auth/Postgres+PostGIS/Storage) + Mapbox GL + OneSignal + Claude vision +
Serwist. Tailwind v4 y shadcn/ui para la UI.
Repo: `unicostudios-mx/de-vuelta` (renombrado; antes `vecino-peludo`), rama
de trabajo: `main` (push directo, NO crear ramas `claude/` sin permiso
explícito). Producción: `de-vuelta.vercel.app`.

## Estado actual

- **Fase 0**: docs listos; pendientes manuales (entrevistas, dominio).
- **Fase 1**: ✅ completa y desplegada en Vercel.
- **Fase 2**: ✅ completa. Auth email+contraseña, CRUD de mascotas con fotos
  en Storage, RLS real en `users`/`pets` (migración 0003), verificada
  end-to-end.
- **Fase 3**: ✅ completa. Reportes de pérdida con mapa Mapbox
  (`LocationPicker` reutilizable), invariante BJ validado client- y
  server-side, RLS de `lost_reports` (migración 0004), verificada
  end-to-end incluyendo intentos de bypass.
- **Fase 4**: ✅ completa. Lado público (`/perdidos`, view
  `active_reports_public` con ubicación aproximada ~300 m) + flujo de
  vecinos con cuenta (`?next=` en auth), RLS de `sightings`
  (migración 0005), verificada end-to-end incluyendo negativos vía API.
- **Fase 5**: ✅ completa y verificada end-to-end. Push OneSignal:
  broadcast a BJ al crear reporte + aviso dirigido al dueño al llegar un
  avistamiento. Notificación real entregada en producción. Antes de tocar
  service workers o llamar `optOut()`, leer
  `.claude/skills/de-vuelta/references/troubleshooting.md` → "Push: la
  configuración correcta (y cómo NO romperla)".
- **Fase 6**: ✅ completa. Matching con Claude vision: cada avistamiento con
  foto recibe un score de coincidencia contra la mascota, y el dueño lo
  confirma o descarta (`lib/matching.ts`, migraciones 0006 y 0007).
  Verificado en producción: 0.97 para la misma perra, 0.02 para un perro
  distinto. Costo real ~$0.01 por avistamiento. El push, la fila de match y
  el análisis corren en `after()` de `next/server` — fuera del camino
  crítico: si algo vuelve a bloquear la respuesta del vecino, la espera se
  dispara a ~27 s. Rate limiting en `lib/rate-limit.ts`: 20 avisos/hora por
  vecino rechazan el envío; 8 análisis/hora por vecino y 25/hora por reporte
  solo saltan el score (el aviso se guarda y el dueño revisa a mano).
- **Siguiente**: Fase 7 (capa comunitaria + adopción curada) — la fase más
  grande del plan; leer el detalle en `CONTEXT.md` sección 7 antes de
  empezar, tiene varias decisiones de producto abiertas.

## Comandos

```bash
npm run dev              # dev server (Turbopack)
npm run build            # build producción (webpack — Serwist lo requiere; NO usar --turbopack)
npm run lint             # ESLint
npm run validate-polygon # prueba el polígono BJ (3 casos)
supabase db push         # aplica migraciones pendientes (CLI ya linkeado al proyecto)
supabase gen types typescript --linked > types/database.ts  # regenera tipos DB
```

No hay suite de tests automatizados: la verificación es `build` + `lint` +
recorrido manual end-to-end en local o producción (incluyendo los casos
negativos: bypass de RLS vía API, coordenadas fuera de BJ, sesión ajena).

## Mapa del repo

```
app/
  layout.tsx              # header con nav según sesión, registra SW y push
  page.tsx                # landing
  globals.css             # Tailwind v4 + tokens de color de marca
  sw.ts                   # worker Serwist (compilado a public/sw.js en build)
  (auth)/                 # login + signup (+ actions.ts: signIn/signUp)
  auth/confirm/route.ts   # canje del token de confirmación por sesión
  logout/route.ts
  mascotas/               # CRUD de mascotas del dueño (+ actions.ts)
  reportes/               # lado del DUEÑO: sus reportes, detalle, revisión de matches
  perdidos/               # lado PÚBLICO: mapa y lista de reportes activos,
                          # + /[id]/avistamiento (formulario del vecino)
components/               # componentes de app (client donde hace falta)
  ui/                     # shadcn/ui generado (button, card, input, …)
lib/
  env.ts                  # única puerta a variables de entorno
  utils.ts                # cn() de shadcn
  geo/validate-bj.ts      # isInBenitoJuarez() + polígono para Mapbox
  supabase/{client,server,middleware}.ts
  notifications.ts        # push OneSignal (server-only)
  matching.ts             # score de coincidencia con Claude vision (server-only)
  rate-limit.ts           # techos horarios de avisos y de análisis (server-only)
middleware.ts             # refresh de sesión en cada request
supabase/migrations/      # 0001..0007, inmutables
types/database.ts         # generado por el CLI de Supabase
scripts/                  # prepare-geo (postinstall), filter-bj, validate-polygon
data/benito-juarez.geojson# polígono oficial CONABIO (canónico, no tocar)
docs/                     # validación, aliados, marca, checklist de cuentas
.claude/skills/de-vuelta/ # skill del proyecto: troubleshooting + bitácora
```

Rutas por audiencia — la distinción importa para RLS y privacidad:
`/perdidos*` es público (ubicación aproximada), `/reportes*` y `/mascotas*`
son del dueño autenticado (ubicación exacta), `/perdidos/[id]/avistamiento`
exige cuenta y vuelve con `?next=`.

## Arquitectura (lo no obvio)

- `lib/geo/bj-polygon.json` es **generado** por `scripts/prepare-geo.mjs` en
  `postinstall` desde `data/benito-juarez.geojson` (canónico, no tocar). Si un
  import falla ahí: `npm install`.
- `lib/env.ts` es la única puerta a variables de entorno (`publicEnv` /
  `serverEnv`), con getters lazy para no romper el build. No usar
  `process.env.X!` directo en código nuevo. Las `NEXT_PUBLIC_*` deben
  referenciarse con su nombre literal completo (Next las sustituye en build).
- Clientes Supabase: `lib/supabase/client.ts` (browser), `server.ts` (SSR +
  `createAdminClient()` con service role), `middleware.ts` (refresh de sesión,
  consumido por `middleware.ts` raíz). No meter lógica entre
  `createServerClient` y `getUser()` en el middleware.
- `createAdminClient()` **se salta RLS**: úsalo solo cuando el usuario
  legítimamente no puede leer lo que se necesita (p. ej. el vecino no puede
  leer `lost_reports`, pero hay que avisarle al dueño). Nada de lo leído así
  puede volver al cliente.
- Módulos con secretos o llamadas de servidor importan `"server-only"`
  (`notifications.ts`, `matching.ts`, `rate-limit.ts`) para que un import
  accidental desde un componente cliente rompa el build, no la producción.
- `types/database.ts` se **genera** con el CLI (`supabase gen types`).
  Regenerarlo tras cada migración. OJO: el CLI escupe avisos a stdout que se
  cuelan en la primera/última línea del archivo — revisarlas.
- RLS: `users`/`pets` (0003), `lost_reports` (0004), `sightings` + view
  pública (0005) y `matches` (0006) ya tienen políticas; 0007 son solo
  índices para el rate limiting. Siguen **deny-all** y sin usar:
  `notifications`, `partners`, `adoptable_pets`, `animal_stories` — les toca
  en Fase 7.
- `active_reports_public` es la **única** superficie pública de datos: corre
  con permisos de su dueño (atraviesa RLS a propósito) y ancla las
  coordenadas a una cuadrícula de 0.003° (~300 m). Snap determinístico, no
  jitter — el jitter se promedia entre lecturas y revela el punto real.
- Storage: buckets `pet-photos` y `sighting-photos`, lectura pública, escritura
  solo del dueño dentro de su carpeta `{user.id}/…`. Las URLs públicas se
  guardan en las columnas `photo_urls`. Servir esas imágenes con
  `next/image` requiere el host de Supabase en `remotePatterns`
  (`next.config.ts`) — si cambia el proyecto de Supabase, cambia ese hostname.
- Service workers: se registra **uno solo**, `/OneSignalSDKWorker.js`
  (`components/sw-register.tsx`), que importa el worker de Serwist. Serwist
  está con `register: false` y desactivado en dev. Dos workers no pueden
  compartir el scope `/`; el SDK de OneSignal se importa en un solo lado o
  duplica listeners. Antes de tocar esto, leer el troubleshooting del skill.
- shadcn/ui: base configurada (`components.json`, `lib/utils.ts`, tokens en
  `app/globals.css`). Agregar componentes con `npx shadcn add <x>` (funciona
  en local; en el entorno remoto de Claude la red lo bloquea).
- Tokens de color: `primary` = teal de marca `#0F766E`, `destructive` =
  urgencia `#DC2626`, extras `urgency`/`success`. Usar clases de token
  (`text-primary`), no hex arbitrarios.

## Patrones del código (seguirlos en código nuevo)

- **Mutaciones = Server Actions**, no route handlers. Viven en el
  `actions.ts` de su carpeta de ruta, con `"use server"` arriba. Las que
  alimentan un formulario devuelven `{ error: string | null }` y se consumen
  con `useActionState`; las de un botón (`deletePet`, `resolveReport`,
  `confirmMatch`) no devuelven nada y lanzan si fallan.
- **Validación con zod** declarada a nivel de módulo. Para fechas usar
  `.refine((d) => d.getTime() <= Date.now() + 60_000, …)`, nunca
  `.max(new Date())`: el schema se evalúa al cargar el módulo y la fecha se
  congelaría; los 60 s absorben el desfase de reloj cliente/servidor.
- **RLS es la autorización real**; los checks en el servidor son defensa en
  profundidad y los del cliente son UX. Tras un `update` bajo RLS, encadenar
  `.select()` y verificar que volvieron filas: un update bloqueado responde
  "ok" con cero filas y el usuario creería que se guardó.
- **Todo lo que es amplificación va en `after()`** de `next/server` (push,
  fila de `matches`, análisis de la foto), envuelto en try/catch y con logs
  `[push]` / `[match]` / `[sighting]`. Nunca bloquear la respuesta del
  usuario con red externa. Si falla, el dato crítico ya está en la DB y hay
  camino manual.
- **Degradar el lujo, no la función**: sin score el aviso igual llega y el
  dueño revisa a mano; sin push el reporte igual existe. Los límites de
  `rate-limit.ts` se resuelven a favor del usuario cuando lo que está en
  juego es su aviso, y en contra cuando lo que está en juego es el crédito
  de la API.
- **Llamadas a Claude**: `client.messages.parse()` con `zodOutputFormat`,
  modelo `claude-opus-5`, `timeout` corto y `maxRetries: 0`; la función
  devuelve `null` si no pudo evaluar y **nunca lanza**.
- **Mensajes de error al usuario en español, humanos y sin filtrar
  internals** ("Puede que el reporte ya esté resuelto", no el error de
  Postgres). Los detalles van a `console.error`.
- **Comentarios**: explican el porqué y la consecuencia de la decisión, no
  el qué. Están en español, como el resto de la prosa del repo; los
  identificadores y los commits, en inglés.

## Reglas del proyecto

1. **Invariante geográfico**: toda coordenada se valida con
   `isInBenitoJuarez()` (`lib/geo/validate-bj.ts`) antes de escribir en DB.
2. **Migraciones inmutables**: nunca editar una migración ya aplicada; crear
   la siguiente numerada. Mostrar el SQL al usuario antes de aplicarlo.
   Si un estado nuevo cabe en columnas existentes, usarlas (así se
   representan los tres estados de revisión de `matches`).
3. **Privacidad**: ubicaciones exactas nunca se exponen públicamente.
4. UI en español mexicano; código y commits en inglés (conventional commits:
   `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`).
5. Nunca commitear `.env.local` ni llaves; `SUPABASE_SERVICE_ROLE_KEY` es
   solo servidor.
6. Antes de commitear: `npm run build && npm run lint` + scan de secrets.
7. Al cerrar cada fase: actualizar `CONTEXT.md` (estado, decisiones, archivos)
   y este `CLAUDE.md`, más la bitácora del skill
   (`.claude/skills/de-vuelta/references/update-log.md`).
8. No saltarse fases: si un prompt pide algo de una fase posterior, confirmarlo
   con el usuario primero.

## Variables de entorno (.env.local — pedir al usuario, nunca inventar)

Configuradas en Vercel y en uso: `NEXT_PUBLIC_SUPABASE_URL` ·
`NEXT_PUBLIC_SUPABASE_ANON_KEY` · `SUPABASE_SERVICE_ROLE_KEY` ·
`NEXT_PUBLIC_MAPBOX_TOKEN` · `NEXT_PUBLIC_ONESIGNAL_APP_ID` ·
`ONESIGNAL_REST_API_KEY` · `ANTHROPIC_API_KEY`. Futura: MercadoPago (F8).
Plantilla completa en `.env.example`.

OJO: las marcadas **Sensitive** en Vercel (`SUPABASE_SERVICE_ROLE_KEY`,
`ONESIGNAL_REST_API_KEY`, `ANTHROPIC_API_KEY`) bajan **vacías** con
`vercel env pull` — hay que reponerlas a mano en `.env.local`. Y las
`NEXT_PUBLIC_*` NO deben marcarse Sensitive (no se inyectan en el build).

## Cuando algo se rompe

`.claude/skills/de-vuelta/` es la memoria del proyecto:
`references/troubleshooting.md` tiene los errores ya resueltos (llaves legacy
de Supabase, tipos que salen `never` por versiones de `@supabase/ssr`, el
componente `form` de shadcn que rompe el install, variables Sensitive,
la configuración de push de OneSignal) y `references/update-log.md` es la
bitácora de sesiones. Consultarlos antes de depurar desde cero.

## Pendientes conocidos

- **BLOQUEADOR del piloto — SMTP propio**: "Confirm email" ya está
  ENCENDIDO (2026-08-21), pero el proyecto usa el SMTP integrado de
  Supabase, limitado a **2 correos/hora** y con el campo no editable en
  plan Free. Es decir: hoy la app admite 2 registros por hora. Hace falta
  SMTP propio (Resend) + dominio antes de abrir a usuarios reales. El
  dashboard mismo lo advierte: "not meant to be used for production apps".
  Las cuentas de prueba NO se ven afectadas: se crean con la Admin API y
  `email_confirm: true`, que se salta el mailer.
- ~~Vistazo visual humano a los mapas~~: HECHO (2026-08-20) vía Claude in
  Chrome, que sí compone el canvas (el panel de navegador de Claude Code
  no). Ambos mapas correctos: `/perdidos` con pines rojos y popup
  "Ver reporte"; detalle del dueño con pin rojo + verdes y zoom ajustado a
  los pines. OJO: los tiles tardan ~4 s en pintar — una captura inmediata
  muestra el mapa en blanco y parece roto.
- **Deuda menor**: las rutas con mapa pesan ~620-650 kB First Load por
  `mapbox-gl`; candidato a `next/dynamic` si llega a molestar.
- **Sin implementar**: el estado `expired` de reportes (necesita un job
  programado). El flujo de callejeros (`sightings` con `report_id` null y
  `needs_help`) existe en el schema desde 0002 pero está diferido a Fase 7.
