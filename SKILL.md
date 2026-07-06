---
name: btp-react-deploy
description: Procedimiento para construir y desplegar apps React (Vite + Tailwind v4) en SAP BTP Cloud Foundry vía MTA, sirviendo desde HTML5 Application Repository y embebibles en Work Zone. Úsalo SIEMPRE que el usuario quiera hacer el deploy de una app React a BTP, prepare una nueva app React para BTP, mencione mta.yaml, xs-app.json, approuter standalone, mbt, cf deploy, HTML5 Apps Repo, "CODE 1001" del upload, destinations XSUAA, o Work Zone embed. Aplica estos pasos y validaciones desde el inicio del desarrollo, no solo al final — varios bloqueos previos del deploy se evitan con configuración correcta desde el primer commit.
---

# Deploy de React en SAP BTP (Cloud Foundry + MTA)

> Patrón para apps React servidas desde el **HTML5 Application Repository** y
> embebibles en **SAP Work Zone** (managed approuter). Reemplaza el patrón
> "wz-stub" (UI5 Component.js + iframe al bundle React): **NO usar** — el
> HTML5 Apps Repo acepta React directo si el manifest es minimal y el zip está
> bien armado.

## Parámetros del proyecto (completar por app)

Este skill es genérico. Reemplazá estos placeholders por los valores de tu
subaccount/proyecto (ver un ejemplo real al final):

| Placeholder | Qué es | Cómo obtenerlo |
|---|---|---|
| `<CF_API>` | endpoint de la API de Cloud Foundry (según región) | cockpit BTP → subaccount → "API Endpoint" |
| `<CF_ORG>` | organización de CF | `cf orgs` |
| `<CF_SPACE>` | space de CF (ej. DEV / QA / PRD) | `cf spaces` |
| `<REGION_HOST>` | host de apps de la región (`cfapps.<region>.hana.ondemand.com`) | del dominio del approuter |
| `<APP_DIR>` | carpeta de la app React (**lowercase**) | tu repo |
| `<MTA_ID>` | id del MTA (= `xsappname`, = `sap.app.id`) | vos lo definís |
| `<SAP_CLOUD_SERVICE>` | valor de `sap.cloud.service` del manifest | vos lo definís |
| `<WORKZONE_HOST>` | host de Work Zone del subaccount (`*.workzone.cfapps.<region>...`) | cockpit / URL de Work Zone |

## Cuándo aplicar este skill

Al crear una app React nueva para BTP **y** al desplegar una existente. Las
configs que rompen el deploy deben quedar bien desde el primer commit, no
parcheadas al final.

## Stack y supuestos

- Framework UI: **React 19** + `react-dom` + `react-router-dom 7` (HashRouter)
- Build tool: **Vite** con `@vitejs/plugin-react`
- Styling: **Tailwind CSS 4** vía `@tailwindcss/vite`
- Iconos: `lucide-react`
- TypeScript con `tsc -b` antes del `vite build`
- Carpeta de salida del build Vite: **`build/`** (NO `dist/`)
- Modo de despliegue: HTML5 Application Repository + approuter standalone
  (embebible en Work Zone vía managed approuter)
- Node CF: definido por buildpack `nodejs_buildpack`

## Checklist previo al deploy (verificar SIEMPRE)

1. **`vite.config.ts`** tiene `base: './'` y `build.outDir: 'build'`.
   `base: './'` es CRÍTICO: con `/` absoluto los assets dan 404 en Work Zone.
2. **`package.json` del app**: el script `build` es
   `"tsc -b && vite build && node scripts/create-zip.cjs"`.
   `archiver` está en `devDependencies`. `ci-lint` apunta a `eslint .`.
3. **`scripts/create-zip.cjs`** zipea `build/` **flat al root del zip** (NO en
   subcarpeta `<id>-<version>/`) y escribe en `../resources/<zip-name>.zip`.
4. **`public/manifest.json` (bundle)** — minimal, SIN `sap.ui` / `sap.ui5` /
   `_version` / `crossNavigation`. Solo:
   ```json
   {
     "sap.app": { "id": "<MTA_ID>", "type": "application",
       "title": "...", "applicationVersion": { "version": "X.Y.Z" } },
     "sap.cloud": { "public": true, "service": "<SAP_CLOUD_SERVICE>" }
   }
   ```
   `applicationVersion.version` **debe coincidir** con `version:` de `mta.yaml`.
5. **`public/xs-app.json` (bundle)** — incluye `/user-api/currentUser`, todas
   las destinations `/api/*` y `/odata/*`, y una **catch-all** al
   `html5-apps-repo-rt` con `authenticationType: xsuaa`. El catch-all debe ser
   la ÚLTIMA regla.
6. **`approuter/xs-app.json` (standalone)** — `welcomeFile` apunta al
   `index.html` del bundle (`/<sap.app.id>/index.html`); rutas a destinations
   antes del catch-all `^/<sap.app.id>/(.*)$ → html5-apps-repo-rt`.
7. **`mta.yaml`**:
   - Módulo `html5` con `path: <APP_DIR>`, `builder: custom`,
     `commands: [npm install, npm run build]`, `build-result: build`.
   - Módulo `com.sap.application.content` con `path: .`,
     `build-result: resources` y SIN `requires/artifacts` (el zip se busca solo
     en `./resources/`).
   - `xsuaa`, `html5-apps-repo` (plan `app-host` + `app-runtime`), `destination`.
8. **`xs-security.json`** — `xsappname` igual al MTA ID (con underscores OK
   para registros nuevos). `redirect-uris` incluye SIEMPRE tanto el host del
   approuter standalone como el de Work Zone (`<WORKZONE_HOST>`).
9. **`tsconfig.app.json`** — `allowJs: true` si vas a importar componentes
   `.jsx`. Sin esto, `tsc -b` falla con "could not find module".
10. **`.gitignore`** incluye `build`, `dist`, `node_modules`, `resources/`
    (no commitear el zip).
11. **Lint pasa**: el pipeline de SAP CI/CD corre `npm run ci-lint` y bloquea
    el deploy ante CUALQUIER error de ESLint (los warnings sí pasan).
12. **Path del módulo html5 en `mta.yaml` es lowercase** y exactamente igual a
    la carpeta real. El pipeline corre en Linux y es case-sensitive.

## Errores que ya nos pasaron y cómo evitarlos

- **Síntoma:** `HTTP 500 — Upload application content failed { CODE: '1001' }
  validation error: Could not find applications in the request.`
  **Causa raíz:** se usaba un patrón "wz-stub" (UI5 Component.js + iframe al
  bundle React) creyendo que el HTML5 Apps Repo solo aceptaba UI5. Acepta React
  directo si el manifest es minimal y el zip está bien armado.
  **Fix preventivo:** manifest sin `sap.ui5.dependencies`, sin `_version`, sin
  Component.js, sin subcarpeta `<id>-<version>/` dentro del zip. Archivos del
  bundle **flat al root** del zip.

- **Síntoma:** Assets `/assets/index-XXXX.js` dan 404 cuando la app corre
  embebida en Work Zone.
  **Causa:** `base: '/'` (absoluto) en `vite.config.ts`. Work Zone monta la app
  bajo un path que no es la raíz.
  **Fix preventivo:** `base: './'` en `vite.config.ts`.

- **Síntoma:** Fetches al backend dan 404 desde Work Zone pero funcionan desde
  el approuter standalone.
  **Causa:** `fetch('/api/...')` con `/` inicial sale a la raíz del origin de
  Work Zone (`<WORKZONE_HOST>`), no al namespace de la app.
  **Fix preventivo:** **fetches SIEMPRE relativos sin `/` inicial**:
  `fetch('api/...')`, `fetch('agent/...')`. Si tenés una constante `BASE_URL`,
  dejala vacía: `` `${BASE_URL}api/...` ``.

- **Síntoma:** `tsc -b` falla con `Could not find module './foo.jsx'` al
  importar un componente JSX desde un TSX.
  **Causa:** `allowJs: false` (default) en `tsconfig.app.json`.
  **Fix preventivo:** `"allowJs": true` en compilerOptions. Si TypeScript
  infiere props demasiado estrictos por defaults `= null`, crear un `.d.ts`
  paralelo con la interface real de props.

- **Síntoma:** `cf deploy` falla durante el upload con mensaje genérico, y al
  inspeccionar el droplet/mtar el `manifest.json` tiene una
  `applicationVersion` distinta de la de `mta.yaml`.
  **Causa:** desync entre las dos versiones; el HTML5 Apps Repo valida que
  coincidan.
  **Fix preventivo:** bumpear AMBOS valores en el mismo commit, siempre.

- **Síntoma:** El pipeline falla en stage Build con `ESLint: N problems` aunque
  local dé 0 errores.
  **Causa:** el pipeline corre `npm run ci-lint` (no `npm run lint`) y a veces
  ESLint es más estricto que la versión local.
  **Fix preventivo:** definir `"ci-lint": "eslint ."` y correrlo antes de
  pushear. Mantener `eslint-plugin-react-hooks` actualizado.

- **Síntoma:** Corre OK en local con `mbt build` pero falla en CI con
  `path: <APP_DIR> not found`.
  **Causa:** CI corre en Linux (case-sensitive) y `mta.yaml` tiene el path con
  mayúscula distinta a la carpeta real.
  **Fix preventivo:** path del módulo html5 en `mta.yaml` SIEMPRE lowercase y
  exactamente igual a la carpeta del filesystem.

- **Síntoma:** El approuter responde HTTP 200 con HTML de login XSUAA en lugar
  de OData; los logs dicen `query does not exist for request url /...`.
  **Causa:** ESE MENSAJE ES UNA PISTA FALSA — significa "no hay query de
  callback OAuth en este request" (no hay cookie de sesión), NO "ruta no existe
  en xs-app.json". Aparece igual para rutas que SÍ existen.
  **Fix preventivo:** no usarlo como evidencia de ruta faltante; para
  diagnosticar mirar `authTokens[0].error` del destination service o reproducir
  con sesión XSUAA válida.

- **Síntoma:** Browser dice 404 al pegar a un endpoint via approuter, pero la
  ruta existe en `xs-app.json` y el destination existe en BTP.
  **Causa típica:** la destination está como `OAuth2UserTokenExchange` y el user
  no tiene los scopes del backend, o el flujo es service-to-service y no hay
  user.
  **Fix preventivo:** para servicios consumidos por múltiples flows (interno +
  público), preferir `OAuth2ClientCredentials` o `BasicAuthentication` con user
  técnico. Más simple y robusto.

## Procedimiento de deploy

```bash
# Desde la raíz del repo del app
rm -rf <APP_DIR>/build resources mta_archives   # limpiar caches
cd <APP_DIR> && npm install && npm run build && cd ..
mbt build
cf login -a <CF_API> --sso
cf target -o "<CF_ORG>" -s <CF_SPACE>
cf deploy mta_archives/<MTA_ID>_<version>.mtar -f
```

Salida esperada: `Process finished.` sin errores. La app queda disponible en
`https://<host>-<MTA_ID>-approuter.<REGION_HOST>`.

## Validación post-deploy

1. **GET al approuter sin auth** debe devolver HTTP 200 con HTML de login XSUAA
   (no 404).
2. **Bundle se sirve**: abrir `https://<approuter>/<sap.app.id>/index.html` tras
   login XSUAA — debe cargar la SPA.
3. **Routing del SPA**: refrescar una URL profunda (`#/ruta/abc`) — debe seguir
   cargando (HashRouter es indistinto).
4. **Inspeccionar el zip generado**:
   ```bash
   unzip -l resources/<zip-name>.zip
   ```
   Los archivos deben estar **flat al root** (no en `<id>-<version>/`).
   `manifest.json` debe estar en raíz del zip.
5. **Logs del approuter** ante cualquier 5xx:
   ```bash
   cf logs <MTA_ID>-approuter --recent | grep -E "msg|error"
   ```

## Archivos clave del repo (referencia)

- `mta.yaml` — descriptor MTA multi-target
- `approuter/xs-app.json` — routing del approuter standalone
- `<APP_DIR>/public/xs-app.json` — routing embebido en el bundle (Work Zone)
- `<APP_DIR>/public/manifest.json` — manifest minimal del HTML5 Apps Repo
- `<APP_DIR>/vite.config.ts` — `base: './'`, `outDir: 'build'`
- `<APP_DIR>/scripts/create-zip.cjs` — empaquetador `build/` → `resources/<zip>.zip`
- `<APP_DIR>/tsconfig.app.json` — `allowJs: true` si hay imports `.jsx`
- `xs-security.json` — `xsappname` + `redirect-uris` (approuter + Work Zone)

## Plantillas

Plantillas estables de `mta.yaml`, `vite.config.ts`, `create-zip.cjs`,
`public/manifest.json` y `xs-app.json` se pueden guardar en `assets/` dentro de
este skill para arrancar apps nuevas desde una base validada (con los
placeholders ya marcados).
