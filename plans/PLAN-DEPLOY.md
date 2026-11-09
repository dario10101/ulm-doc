# ULM - Plan de deploy

Estado: app en produccion en https://getulm.app; fases 1-6 hechas: **el back se despliega solo con cada push a `main`** (primer deploy automatico 2026-11-09, `339fb74`). Siguiente: fase 7 (limpieza de Artifact Registry) y 8 (backups).
Objetivo: beta personal en produccion y, despues, abrirla a los 5-10 usuarios invitados ([PLAN-AUTH.md](./done/PLAN-AUTH.md)), con el deploy del back automatico al hacer push a `main`.

## Arquitectura en produccion

```
navegador ──► getulm.app (Cloudflare Worker ulm-web)
                ├─ /*      -> archivos de dist/ (SPA)
                └─ /api/*  -> proxy ──► Cloud Run ulm-core (us-east1) ──► Neon Postgres 16 (AWS us-east-1)
```

| Pieza | Donde | Notas |
|---|---|---|
| Dominio | Cloudflare Registrar: `getulm.app` | DNSSEC activo, WHOIS privado, renovacion automatica |
| Front | Cloudflare Worker con static assets (`ulm-web/wrangler.jsonc`) | Workers Builds: build y deploy en cada push a `main` |
| Proxy `/api` | `ulm-web/worker/index.ts` | Front y API en el mismo origen: la cookie `SameSite=Lax` funciona sin CORS con credenciales |
| API | Cloud Run `ulm-core`, SA `ulm-core-runtime`, min 0 / max 2 instancias, 512Mi | `--allow-unauthenticated`: el login lo hace la app |
| Migraciones | Cloud Run Job `ulm-migrate` (`alembic upgrade head`), misma imagen | Siempre antes de desplegar la API |
| Imagenes | Artifact Registry `us-east1/ulm/ulm-core` | |
| Secretos | Secret Manager: `database-url`, `session-secret`, `google-client-secret`, `openai-api-key` | Solo `ulm-core-runtime` los lee |
| Config no secreta | `ulm-infra/prod/core.env.yaml` | Se aplica a mano desde `ulm-infra` (ver su README) |
| Base | Neon plan Free, branch `production`, conexion directa (sin pooler) | Scale-to-zero a los 5 min; `pool_pre_ping` en el engine |
| Login | Cliente OAuth de prod, consent screen en modo Testing | Authorized domain `getulm.app`; cada invitado va tambien en Test users |

## Decisiones

- **Mismo origen via proxy en el Worker** en vez de `api.<dominio>`: no cambia CORS, cookie ni `VITE_API_BASE_URL` (`/api/v1`).
- **Worker con static assets, no Pages**: es lo que Cloudflare ofrece hoy para proyectos nuevos; mismo plan gratis. `functions/` no aplica.
- **Neon Free con scale-to-zero**: siempre encendida costaria ~20 USD/mes; la latencia de despertar (~1 s) es aceptable para la beta. Pasar a Launch solo si se acerca a las 100 CU-h/mes o se quiere historial de 7 dias.
- **Datos reales migrados con `pg_dump`/`pg_restore`** de la base local (2026-10-31), no con los scripts de import: conserva ids y todo el contenido. El dump queda en `ulm-data/backups/`.
- **A produccion viaja esquema, nunca datos**: las migraciones de Alembic son lo unico que se comparte entre local y prod.
- **`/docs`, `/redoc` y `/openapi.json` apagados** con `ENVIRONMENT=production`.
- **Deploy automatico solo cambia la imagen** (`--image`): Cloud Run conserva variables y secretos de la revision anterior. La config sigue siendo un cambio manual desde `ulm-infra`, asi un push de codigo no altera la config de prod y el CI de `ulm-core` no depende de otro repo.
- **`main` = prod** en los dos repos. Otras ramas: solo checks.
- **Auth del CI con Workload Identity Federation**: sin claves JSON en GitHub. Solo se aceptan tokens de `dario10101/ulm-core` en `refs/heads/main`.
- **Backups (fase 8): `pg_dump` diario a un bucket privado de Cloud Storage**, independiente del deploy. Nunca como artefacto de GitHub Actions (visible para quien lea el repo). Snapshot de Neon antes de cada migracion: opcional, despues.

## Fases

| Fase | Contenido | Estado |
|---|---|---|
| 1 | Neon: proyecto en AWS us-east-1, PG 16, restore del dump local | ✅ |
| 2 | GCP: Artifact Registry, SA `ulm-core-runtime`, secretos, Job `ulm-migrate`, servicio `ulm-core`; `ulm-infra/prod/core.env.yaml` | ✅ |
| 3 | Dominio `getulm.app`, Worker `ulm-web` con proxy `/api` y `_headers`, OAuth de prod | ✅ |
| 4 | Prueba completa en prod (login, modulos, logout); CI de `ulm-web` en verde | ✅ |
| 5 | WIF + SA de deploy (una sola vez, comandos a mano): pool y provider OIDC de GitHub con condicion de repo y rama; SA `ulm-deployer` con `artifactregistry.writer` (repo `ulm`), `run.developer` e `iam.serviceAccountUser` (solo sobre `ulm-core-runtime`); variables de GitHub `GCP_PROJECT`, `GCP_REGION`, `WIF_PROVIDER`, `DEPLOYER_SA`. Detalle abajo | ✅ |
| 6 | Job `deploy` en `ulm-core/.github/workflows/ci.yml`: `needs` lint/tests/postgres/docker, solo `main` + `workflow_dispatch`, `concurrency` sin cancelar. Pasos: build y push con tag = SHA -> `jobs update` + `execute --wait` de `ulm-migrate` -> `run deploy --image` -> smoke test `https://getulm.app/api/v1/health`. Actualizar actions a versiones Node 24. Primer deploy con un cambio inocuo. Reemplaza a `wif-check.yml`. Detalle abajo | ✅ |
| **7** | **Limpieza y doc:** politica de Artifact Registry (ultimas 5 imagenes), rollback ya documentado en `ulm-infra/README.md` (fase 6), reglas de migraciones en `technical-design.md` | ⏳ |
| 8 | Backups: `pg_dump` diario de Neon a bucket privado de Cloud Storage (`us-east1`, capa gratis) con retencion; probar un restore | ⏳ |

## Fase 5 en detalle: WIF + SA de deploy

Objetivo: que un workflow de `dario10101/ulm-core` corriendo en `main` pueda actuar en GCP como `ulm-deployer`, sin ninguna clave guardada en GitHub. Al terminar, GitHub solo tiene 4 variables no secretas.

### Como funciona

```
GitHub Actions (job en main)
  │ 1. pide un token OIDC firmado por GitHub (claims: repository, ref, sha...)
  ▼
Security Token Service de GCP
  │ 2. el provider "ulm-core" valida firma + condicion (repo y rama)
  │ 3. lo cambia por un token federado del pool "github"
  ▼
IAM Credentials
  │ 4. el pool tiene workloadIdentityUser sobre ulm-deployer -> token de ulm-deployer (1 h)
  ▼
gcloud / docker en el runner actuan como ulm-deployer
```

- **Pool**: contenedor de identidades externas (aca, "lo que venga de GitHub").
- **Provider**: a quien se le cree (issuer `token.actions.githubusercontent.com`), como se mapean los claims del token a atributos y **que condicion tienen que cumplir**. La condicion es la frontera de seguridad: un token de otro repo o de otra rama se rechaza aca.
- **`principalSet`**: el conjunto de identidades del pool con un atributo dado (aca, `repository = dario10101/ulm-core`); es a quien se le permite hacerse pasar por `ulm-deployer`.

### Datos fijos

| Dato | Valor |
|---|---|
| Proyecto (id / numero) | `unified-life-manager` / `802907629918` |
| Repo | `dario10101/ulm-core`, rama `main` |
| Pool / provider | `github` / `ulm-core` |
| SA de deploy | `ulm-deployer@unified-life-manager.iam.gserviceaccount.com` |
| SA de runtime (ya existe) | `ulm-core-runtime@unified-life-manager.iam.gserviceaccount.com` |

Comandos en PowerShell, una linea cada uno (si la politica de scripts bloquea `gcloud`, usar `gcloud.cmd`).

### 5.1 Activar APIs

```powershell
gcloud services enable iam.googleapis.com iamcredentials.googleapis.com sts.googleapis.com
```

`sts` cambia el token de GitHub por uno de Google; `iamcredentials` emite el token de la SA.

### 5.2 Pool y provider

```powershell
gcloud iam workload-identity-pools create github --location=global --display-name="GitHub Actions"
gcloud iam workload-identity-pools providers create-oidc ulm-core --location=global --workload-identity-pool=github --display-name="ulm-core main" --issuer-uri="https://token.actions.githubusercontent.com" --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.ref=assertion.ref" --attribute-condition="assertion.repository=='dario10101/ulm-core' && assertion.ref=='refs/heads/main'"
```

- La condicion mira `ref` y no `sub`: si mas adelante el job usa un *environment* de GitHub, `sub` cambia de forma y `ref` no.
- `push` y `workflow_dispatch` lanzados sobre `main` cumplen la condicion; un PR, otra rama o un fork, no.

### 5.3 SA de deploy y sus permisos

```powershell
gcloud iam service-accounts create ulm-deployer --display-name="ULM deploy desde GitHub Actions"
gcloud artifacts repositories add-iam-policy-binding ulm --location=us-east1 --member="serviceAccount:ulm-deployer@unified-life-manager.iam.gserviceaccount.com" --role="roles/artifactregistry.writer"
gcloud projects add-iam-policy-binding unified-life-manager --member="serviceAccount:ulm-deployer@unified-life-manager.iam.gserviceaccount.com" --role="roles/run.developer" --condition=None
gcloud iam service-accounts add-iam-policy-binding ulm-core-runtime@unified-life-manager.iam.gserviceaccount.com --member="serviceAccount:ulm-deployer@unified-life-manager.iam.gserviceaccount.com" --role="roles/iam.serviceAccountUser"
```

| Rol | Alcance | Para que |
|---|---|---|
| `artifactregistry.writer` | Solo el repo `ulm` | Push de la imagen |
| `run.developer` | Proyecto | `jobs update` / `jobs execute --wait` de `ulm-migrate` y `run deploy` de `ulm-core` |
| `iam.serviceAccountUser` | Solo `ulm-core-runtime` | Desplegar revisiones (y el Job) que corren con esa identidad |

Lo que **no** tiene: lectura de secretos, acceso a Neon, IAM del proyecto. Las migraciones las corre el Job como `ulm-core-runtime`, que es quien lee `database-url`.

### 5.4 Permitir que GitHub use `ulm-deployer`

```powershell
gcloud iam service-accounts add-iam-policy-binding ulm-deployer@unified-life-manager.iam.gserviceaccount.com --role="roles/iam.workloadIdentityUser" --member="principalSet://iam.googleapis.com/projects/802907629918/locations/global/workloadIdentityPools/github/attribute.repository/dario10101/ulm-core"
```

Doble filtro: el provider solo deja pasar `main` de `ulm-core` (5.2), y este binding solo le da `ulm-deployer` a ese repo.

### 5.5 Variables en GitHub

Nombre completo del provider:

```powershell
gcloud iam workload-identity-pools providers describe ulm-core --location=global --workload-identity-pool=github --format="value(name)"
```

Deberia dar `projects/802907629918/locations/global/workloadIdentityPools/github/providers/ulm-core`.

En `ulm-core` -> *Settings -> Secrets and variables -> Actions -> pestaña **Variables*** (no Secrets: nada de esto es sensible):

| Variable | Valor |
|---|---|
| `GCP_PROJECT` | `unified-life-manager` |
| `GCP_REGION` | `us-east1` |
| `WIF_PROVIDER` | La salida del comando anterior |
| `DEPLOYER_SA` | `ulm-deployer@unified-life-manager.iam.gserviceaccount.com` |

### 5.6 Verificacion (workflow temporal)

Prueba WIF aislada, antes de meter el deploy en el CI: si algo falla en la fase 6, ya se sabe que no es la autenticacion.

`ulm-core/.github/workflows/wif-check.yml`, solo manual:

```yaml
name: WIF check
on: workflow_dispatch
permissions:
  contents: read
  id-token: write   # sin esto GitHub no emite el token OIDC
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: google-github-actions/auth@v3
        with:
          workload_identity_provider: ${{ vars.WIF_PROVIDER }}
          service_account: ${{ vars.DEPLOYER_SA }}
      - uses: google-github-actions/setup-gcloud@v3
      - run: gcloud run services describe ulm-core --region=${{ vars.GCP_REGION }} --format="value(status.url)"
      - run: gcloud artifacts docker images list ${{ vars.GCP_REGION }}-docker.pkg.dev/${{ vars.GCP_PROJECT }}/ulm --limit=3
```

Push a `main` y lanzarlo desde *Actions -> WIF check -> Run workflow*:

1. **Desde `main`** -> verde, imprime la URL de `ulm-core` y las imagenes.
2. **Desde otra rama** (crear `wif-test`, push, lanzar sobre esa rama) -> tiene que **fallar** en `auth` con un error de condicion/permiso. Esto prueba que la condicion de rama funciona.

Despues borrar `wif-check.yml` y la rama `wif-test`. Las versiones `@v3` de las actions de Google son las vigentes al escribir esto: verificar en su repo al usarlas.

### Problemas conocidos

- `Permission 'iam.serviceAccounts.getAccessToken' denied`: falta el binding 5.4 o el `principalSet` tiene mal el numero de proyecto o el repo. Los cambios de IAM pueden tardar unos minutos en propagarse.
- `The given credential is rejected by the attribute condition`: el token no cumple 5.2 (otra rama, o el nombre del repo no coincide exacto, mayusculas incluidas).
- `Unable to retrieve ... ACTIONS_ID_TOKEN_REQUEST_URL`: falta `id-token: write` en `permissions`.

### Terminada cuando

- [x] Pool `github` y provider `ulm-core` creados con la condicion de repo y rama
- [x] `ulm-deployer` con sus 3 roles y el binding `workloadIdentityUser`
- [x] 4 variables en GitHub
- [x] `wif-check` verde desde `main` y rechazado desde otra rama
- [x] Rama de prueba borrada
- [x] `wif-check.yml`: borrado en la fase 6

## Fase 6 en detalle: deploy automatico del back

Objetivo: cada push a `main` de `ulm-core` que pasa los checks termina, sin intervencion, con la base migrada y la nueva revision sirviendo en `getulm.app`. Todo en `ulm-core`; GCP no se toca (lo dejo la fase 5).

### Flujo

```
push a main ─► lint ─┐
               tests ─┼─► deploy: build + push (tag SHA) ─► Job ulm-migrate ─► run deploy --image ─► smoke test
            postgres ─┤                                     (si falla, corta)          (getulm.app devuelve el SHA)
              docker ─┘
```

Cada flecha es un paso del mismo job: si uno falla, los siguientes no corren y el run queda en rojo.

| Si falla... | Estado de prod | Que hacer |
|---|---|---|
| Un check (lint, tests, postgres, docker) | Intacto: `deploy` ni arranca | Arreglar y push |
| Build / push de la imagen | Intacto | Ver el log del paso |
| `ulm-migrate` | API vieja sirviendo; la base puede quedar a mitad **solo** si la migracion no es transaccional (Postgres hace DDL en transaccion: lo normal es que quede como estaba) | Logs del Job en Cloud Run; arreglar con migracion nueva |
| `run deploy` | Base ya migrada, API vieja sirviendo (Cloud Run no pasa trafico a una revision que no arranca) | Por expand/contract la API vieja funciona con la base nueva; arreglar y push |
| Smoke test | Revision nueva sirviendo pero algo entre Cloudflare y la API falla | Revisar; rollback de la API si hace falta (abajo) |

### 6.1 Version visible en `/health` (cambio de codigo)

Para que el smoke test pruebe que responde **la revision nueva** y no solo "algo".

- `Dockerfile`: `ARG GIT_SHA=dev` + `ENV APP_VERSION=$GIT_SHA`. La version queda dentro de la imagen, no en la config de Cloud Run (coherente con "el CI solo cambia la imagen").
- `app/core/config.py`: `app_version: str = "dev"`.
- `/health` devuelve tambien `"version"`. Test que lo cubra.
- El build del CI pasa `--build-arg GIT_SHA=${{ github.sha }}`; el build local sigue diciendo `dev`.

`/health` es publico: exponer el SHA no filtra nada (el repo ya dice que corre). Si el repo pasa a publico y no gusta, se puede acortar o quitar.

### 6.2 Job `deploy` en `.github/workflows/ci.yml`

```yaml
on:
  push:
    branches: ['**']
  pull_request:
  workflow_dispatch: # redeploy manual de main sin commit nuevo

permissions:
  contents: read # por defecto para todos los jobs; deploy agrega id-token

jobs:
  # ...lint, tests, postgres, docker sin cambios (salvo versiones de actions)...

  deploy:
    needs: [lint, tests, postgres, docker]
    if: github.ref == 'refs/heads/main' && github.event_name != 'pull_request'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    concurrency:
      group: deploy-prod
      cancel-in-progress: false # nunca cortar un deploy a mitad de una migracion
    env:
      REGION: ${{ vars.GCP_REGION }}
      PROJECT: ${{ vars.GCP_PROJECT }}
      IMAGE: ${{ vars.GCP_REGION }}-docker.pkg.dev/${{ vars.GCP_PROJECT }}/ulm/ulm-core:${{ github.sha }}
    steps:
      - uses: actions/checkout@v5
      - uses: google-github-actions/auth@v3
        with:
          workload_identity_provider: ${{ vars.WIF_PROVIDER }}
          service_account: ${{ vars.DEPLOYER_SA }}
      - uses: google-github-actions/setup-gcloud@v3
      - name: Build y push
        run: |
          gcloud auth configure-docker "$REGION-docker.pkg.dev" --quiet
          docker build --build-arg GIT_SHA=${{ github.sha }} -t "$IMAGE" .
          docker push "$IMAGE"
      - name: Migraciones
        run: |
          gcloud run jobs update ulm-migrate --image="$IMAGE" --region="$REGION" --project="$PROJECT" --quiet
          gcloud run jobs execute ulm-migrate --wait --region="$REGION" --project="$PROJECT"
      - name: Deploy de la API
        run: gcloud run deploy ulm-core --image="$IMAGE" --region="$REGION" --project="$PROJECT" --quiet
      - name: Smoke test
        run: |
          for i in $(seq 1 10); do
            body=$(curl -fsS https://getulm.app/api/v1/health) && echo "$body" | grep -q '"${{ github.sha }}"' && { echo "$body"; exit 0; }
            sleep 6
          done
          echo "getulm.app no devuelve la version ${{ github.sha }}: $body"; exit 1
      - name: Resumen
        if: always()
        run: echo "Imagen \`$IMAGE\`" >> "$GITHUB_STEP_SUMMARY"
```

Puntos de diseño:

- **`if` del job**: `push` y `workflow_dispatch` en `main`. Un PR hacia `main` corre los checks pero no despliega (y WIF lo rechazaria igual: su `ref` no es `refs/heads/main`).
- **`needs` los 4 checks**: el job `docker` ya prueba que la imagen construye, migra una base vacia y arranca; `deploy` vuelve a construir (~1-2 min mas). Reusar la imagen entre jobs (artefacto o push desde `docker`) ahorra ese tiempo pero acopla los jobs; no vale la pena a esta escala.
- **`concurrency` con `cancel-in-progress: false`**: dos pushes seguidos se encolan. Cancelar un deploy en curso podria cortarlo entre la migracion y el deploy de la API.
- **`run deploy` solo con `--image`**: conserva env vars, secretos, SA, escalado y acceso publico de la revision anterior. Lo mismo `jobs update --image` (conserva `DATABASE_URL`).
- **Smoke test por `getulm.app`** (no por `*.run.app`): prueba la cadena real Cloudflare -> proxy -> Cloud Run -> app, y con el SHA confirma que responde la revision nueva. Reintenta ~1 min por el arranque en frio (Cloud Run + Neon).
- **Versiones de actions a Node 24** en todo el `ci.yml`: `checkout@v5`, `setup-python@v6` (hay v7; v6 es la que se confirmo con Node 24). Saca el aviso de deprecacion de Node 20.

### 6.3 Limpieza

- Borrar `.github/workflows/wif-check.yml` (la verificacion queda cubierta por el paso `auth` de `deploy`).
- `ulm-infra/README.md`: el camino normal pasa a ser el CI; los comandos manuales quedan como respaldo (deploy de emergencia o cambio de config), con imagen por SHA en vez de `v1/v2`.

### 6.4 Primer deploy

El propio commit de 6.1-6.3 es el cambio inocuo:

1. Push a `main` -> en *Actions*, el run tiene que terminar con los 5 jobs en verde.
2. Log de `Migraciones`: el Job termina OK sin aplicar nada (la base ya esta en head).
3. `https://getulm.app/api/v1/health` devuelve `"version": "<SHA del commit>"`.
4. En Cloud Run, `ulm-core` tiene una revision nueva con la imagen `:<SHA>` y conserva variables y secretos (*Revisions -> la nueva -> Variables & Secrets*).
5. Login y una pantalla con datos en `getulm.app`.
6. Un push a otra rama: corren los checks y `deploy` aparece como *skipped*.

### Problemas conocidos

- `Permission 'run.jobs.run' denied` o similar en `Migraciones`: a `run.developer` le falta algo para ejecutar Jobs; se agrega `roles/run.invoker` sobre el Job `ulm-migrate`.
- `Permission 'iam.serviceaccounts.actAs' denied`: falta `serviceAccountUser` de `ulm-deployer` sobre `ulm-core-runtime` (fase 5.3).
- `denied: Permission "artifactregistry.repositories.uploadArtifacts"`: falta `configure-docker` para la region correcta, o el rol sobre el repo `ulm`.
- Smoke test falla pero `*.run.app/api/v1/health` muestra el SHA nuevo: problema en el Worker de `ulm-web` o en Cloudflare, no en el back.

### Rollback de la API

```powershell
gcloud run revisions list --service=ulm-core --region=us-east1
gcloud run services update-traffic ulm-core --region=us-east1 --to-revisions=<revision-anterior>=100
```

Sin rebuild. Ojo: mientras el trafico este fijado a una revision, un deploy nuevo no lo mueve solo; volver con `--to-latest`. La base no tiene rollback automatico: por eso cada migracion tiene que ser compatible con el codigo anterior (expand/contract).

### Terminada cuando

- [x] `/health` devuelve `version` (con test)
- [x] Job `deploy` en `ci.yml`, actions en versiones Node 24
- [x] `wif-check.yml` borrado; README de `ulm-infra` actualizado
- [x] Primer deploy automatico verificado (puntos 1-6 de 6.4)

## Reglas de migraciones con datos en prod

- Revisar siempre el `--autogenerate`: un renombre sale como drop + add y borra datos.
- Columna nueva `NOT NULL` en tabla con filas: `server_default` o backfill en la misma migracion.
- Cambios destructivos en dos deploys (expand/contract): agregar y copiar -> la app usa lo nuevo -> borrar lo viejo en un deploy posterior.
- En prod no se hace `downgrade` para arreglar: migracion nueva hacia adelante.
- Migracion delicada: probarla antes en un branch de Neon creado desde `production`, y borrar el branch al terminar.
- Datos de prod se cambian por la app o por scripts con `--dry-run`, nunca a mano en DBeaver.

## Pendientes fuera de las fases

- `blog.getulm.app`: hoy el blog vive en `getulm.app/blog`. Decidir redirect (cero codigo) o que el router detecte el subdominio.
- El front se publica aunque falle su CI (Workers Builds no espera a GitHub Actions). Si molesta: deploy del front desde el CI con `wrangler deploy` despues de los checks.
- La URL `*.run.app` sigue accesible directo (sin la cookie de `getulm.app` da 401). Opcional: header secreto que solo pone el proxy.
- CSP completa (T9): inventariar antes los origenes de imagenes del English.
- Node 20 sin soporte desde 2026-04: subir local, CI y `NODE_VERSION` del Worker a 22/24.
- `ulm-repository` sin remoto (T2). `ulm-infra` ya tiene remoto privado (incluye `ADMIN_EMAILS`); `ulm-core` es publico: no guardar ahi nada sensible ni artefactos con datos.
- Antes de los 90 dias del trial de GCP: activar la cuenta paga (si no, se apagan los recursos). Presupuesto con alerta de 5 USD.
