# Diseño tecnico

Ultima revision: 2026-10-29. Detalle por repo en `ulm-core/CLAUDE.md` y `ulm-web/CLAUDE.md` (no se duplica aqui).

## Arquitectura

Monolito modular: un backend, un frontend SPA, una base Postgres. Cada dominio (planning, finanzas, salud, blog, english, ia) es un modulo con la misma cadena de capas.

```
ulm-web (Vue SPA) --/api--> ulm-core (FastAPI) --> PostgreSQL
                                   |--> Google OIDC, OpenAI, ARASAAC/Pexels/Openverse
```

| Repo | Rol | Versionado |
|---|---|---|
| `ulm-core` | Backend FastAPI | git + GitHub, CI |
| `ulm-web` | Frontend Vue | git + GitHub, CI |
| `ulm-repository` | Guias `how-to/` y contenido `.md` | git local, sin remoto |
| `ulm-infra` | IaC / CI-CD futuro | vacio |
| `ulm-data` | Archivos fuente de imports (xlsx) | **sin git** (datos personales) |
| `ulm-doc` | Esta documentacion | git + GitHub |

## Stack

- **Backend**: Python 3.12, FastAPI, Pydantic v2, SQLAlchemy 2.0, Alembic, psycopg2, pytest, ruff.
- **Frontend**: Vue 3 (`<script setup>`) + TypeScript, Vite 5 (fijado: Vite 8/rolldown rompe en Windows), Vue Router, Tailwind (colores por variables CSS), Vitest, eslint + prettier.
- **Base**: PostgreSQL 16 en Docker local.
- **IA**: OpenAI SDK (Responses API, `store=False`), `whisper-1` para voz. Catalogo cerrado de modelos en `app/services/ai_models.py`. Sin LangChain.
- **CI**: GitHub Actions en core (lint, tests SQLite, job Postgres con migraciones ida y vuelta + `check_db_names`) y web (lint, typecheck, vitest).

## Capas del backend

`route -> service -> repository (Protocol) -> SQLAlchemy`. Las rutas nunca tocan el ORM; los services devuelven schemas (`mappers.py`); errores de dominio (`services/errors.py`) sin HTTP; integraciones externas detras de un `Protocol` en `app/integrations`.

- **Transaccion**: una por request, commit solo en `get_db`. Los repositories hacen `flush()`, nunca `commit()`.
- **Multiusuario**: todo repository recibe `*, user_id`; un recurso ajeno responde 404 (no 403). Tablas hijas: FK compuesta o validacion en service + test A/B.
- **Auth**: Google OIDC + PKCE, sesion en cookie httpOnly (token hasheado en `user_sessions`). Routers en `PRIVATE_ROUTERS`/`PUBLIC_ROUTERS`; tests fallan si una ruta queda sin sesion o sin permiso. Decisiones en [PLAN-AUTH.md](../plans/done/PLAN-AUTH.md).
- **Blog**: excepcion deliberada al modelo relacional: tabla generica pk/sk JSONB `blg_entities` (estilo DynamoDB). Ver [PLAN-BLOG.md](../plans/PLAN-BLOG.md) seccion 7.
- **Consumo IA**: toda llamada paga registra en `ai_usage_events` (`AiUsageRecorder`).

## Reglas permanentes

- **Un cambio de esquema = una migracion.** Revisar siempre el `--autogenerate` (renombres salen como drop + add, con perdida de datos).
- **Nombres de tabla**: `<modulo>_user_<entidad>` si es de un usuario; `<modulo>_default_<entidad>` si es plantilla para usuarios nuevos (se copia al registrarse, sin `status`, borrar es borrar); sin `user` si es catalogo global o cuelga de otra tabla. Renombrar arrastra a mano secuencia, PK, FK e indices; `scripts/check_db_names.py` lo verifica.
- **Postgres no indexa FKs solo**: `index=True` en las que se consultan; nunca en `primary_key`.
- **Zona horaria**: BD en UTC, dominio en la zona del usuario, conversion solo al entrar/salir del service. Nunca `.date()` ni `isoweekday()` sobre un instante. Ver `ulm-repository/how-to/timezones.md`.
- **Tests independientes**: si pasa en suite y falla aislado, el test esta mal.
- **LLM solo para lo que no resuelve el codigo**: el LLM narra, no calcula; `user_id` siempre de la sesion.
- **Markdown**: render en el front y sanitizado siempre (`MarkdownContent.vue`, unico `v-html`).
- **Commits chicos y tematicos**, mensajes cortos.

## Deploy

En produccion en `getulm.app`: Cloudflare Worker con static assets (web + proxy `/api` al backend, mismo origen) + Cloud Run us-east1 (core) + Neon (Postgres) + dominio en Cloudflare. `main` = prod. Arquitectura, fases y reglas de migraciones con datos en prod en [PLAN-DEPLOY.md](../plans/PLAN-DEPLOY.md).

### Contenedores

- **Solo `ulm-core` va en contenedor** (Cloud Run solo ejecuta imagenes). `ulm-web` son estaticos en un Worker de Cloudflare: un nginx seria un servidor extra sin ganancia. Postgres es Neon: en contenedor obligaria a manejar disco y backups a mano. Postgres en Docker solo en local.
- **`Dockerfile` en `ulm-core`** (lo construye su CI); en `ulm-infra` va lo que cruza repos. `ulm-infra/local/` = solo desarrollo (compose: `db` + `migrate` + `api`).
- **Migraciones fuera del arranque**: la misma imagen corre `alembic upgrade head` como paso aparte (servicio `migrate` en local, Cloud Run Job antes de cada revision en prod). En el entrypoint, dos instancias que arrancan a la vez migrarian en paralelo, y un fallo se veria como "la API no arranca" en vez de "el deploy fallo".
- **Config solo por variables de entorno**; nunca un `.env` dentro de la imagen (`.dockerignore`). Imagen sin dependencias de dev, usuario no root, puerto de `$PORT`, logs por stdout.

### Migraciones con datos en prod

El CI de `ulm-core` migra la base de prod en cada push a `main`, antes de desplegar la API: durante un rato la API vieja corre contra la base nueva, y no hay rollback de la base.

- **Compatibles con el codigo anterior (expand/contract)**: cambios destructivos (renombrar, cambiar tipo, borrar) en dos deploys: agregar y copiar -> la app usa lo nuevo -> borrar lo viejo en un deploy posterior.
- **Columna nueva `NOT NULL` en tabla con filas**: `server_default` o backfill en la misma migracion (en local las tablas suelen estar vacias y no se nota).
- **En prod no se hace `downgrade`**: se arregla con una migracion nueva hacia adelante (el downgrade recrea la estructura, no los datos).
- **Migracion delicada**: probarla antes en un branch de Neon creado desde `production`; borrar el branch al terminar.
- **Datos de prod**: se cambian por la app o por scripts con `--dry-run`, nunca a mano en un cliente SQL.
- **Las migraciones solo siembran catalogo global**, nunca datos de un usuario (eso va en su registro).
