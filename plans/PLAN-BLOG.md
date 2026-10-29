# ULM - Plan del blog multiusuario

Ultima actualizacion: 2026-10-13 (iteracion 1 commiteada como "Blog structure"; **fases 1, 2 y 3 hechas, sin commitear**; falta la verificacion manual en el navegador de la 3 y la fase 4 (endurecimiento))
Documento de planeacion del modulo blog. Lo pendiente va primero; las decisiones se registran en la seccion 12.

---

## 1. Estado actual (verificado en el codigo)

El blog ya es **multiusuario en la URL**, pero el contenido sigue **quemado en el front** y es solo del dueño del ejemplo:

| Pieza | Donde | Estado |
|---|---|---|
| Username | `users.username` ([user.py](../../ulm-core/app/db/models/user.py)), mig `e4b19c7d2a60` | Nulo hasta que el usuario lo crea; unico sin distinguir mayusculas (`uq_users_username` sobre `lower(username)`); **inmutable** una vez creado. Reglas en [username.py](../../ulm-core/app/services/username.py). |
| API de cuenta | `PUT /me/username` ([me.py](../../ulm-core/app/api/routes/me.py)) | Crea el username (422 formato/reservado, 409 en uso o ya creado). `GET /me` devuelve `username`. No exige permiso de dominio. |
| API publica | `GET /public/blogs/{username}[/{seccion}/items[/{elemento}]]` ([public_blogs.py](../../ulm-core/app/api/routes/public_blogs.py), [blog_service.py](../../ulm-core/app/services/blog_service.py)) | ✅ Fase 2: blog + secciones habilitadas con su contenido en un request; colecciones (subsecciones + publicados, sin cuerpo); elemento con su cuerpo sin frontmatter. **404 uniforme** para todo lo que no se puede ver. `Cache-Control` + `ETag` (304). |
| Permiso | [permissions.py](../../ulm-core/app/services/permissions.py) | `blog` (sin `.ai`). Visible en Settings -> Users. |
| Rutas | [public.routes.ts](../../ulm-web/src/router/routes/public.routes.ts) | `/blog` = landing. `/blog/:username` (`blog-home`, redirige a la primera seccion), `/:section` (`blog-section`), `/:section/:item` (`blog-item`): **dinamicas por slug**. |
| Layout | [PublicLayout.vue](../../ulm-web/src/layouts/PublicLayout.vue) | Resuelve el blog contra la API y lo provee (`useBlogOwner`: username, display_name, secciones). Menu desde las secciones habilitadas, colapsable en movil; "Este blog no existe" en 404; URL canonica sin pedir el blog dos veces. |
| Contenido | `blg_entities`, cargado con [seed_blog.py](../../ulm-core/scripts/seed_blog.py) | ✅ Sin contenido quemado: `posts.ts`, `blogDemo.ts` y las 5 paginas fijas se borraron. Vistas por tipo en `ulm-web/src/views/public/blog/` y registro en `config/blogSectionTypes.ts`. Markdown con `lib/markdown.ts` (markdown-it + DOMPurify) via `MarkdownContent.vue`. |
| Settings | [UsernameCard.vue](../../ulm-web/src/components/account/UsernameCard.vue) en Settings -> General | Crear username con validacion de formato en vivo. |
| Admin | `/admin/blog` ([BlogAdminPage.vue](../../ulm-web/src/views/admin/BlogAdminPage.vue)), item "Blog" al final del sidebar | Exige permiso `blog`. Sin username: pide crearlo. Con username: link al blog y modulo vacio. |
| Secciones, plantillas, `.md` (backend) | `blg_entities` (mig `4c07b2a15e70`), [blog_admin_service.py](../../ulm-core/app/services/blog_admin_service.py), rutas `/blog/*` ([blog.py](../../ulm-core/app/api/routes/blog.py)) | ✅ Fase 1: API privada completa (config, secciones, contenido de plantillas, subsecciones, subida/descarga de `.md`). Sin datos todavia. |
| UI de gestion | `/admin/blog` ([BlogAdminPage.vue](../../ulm-web/src/views/admin/blog/BlogAdminPage.vue)) y `/admin/blog/sections/:id` ([BlogSectionEditPage.vue](../../ulm-web/src/views/admin/blog/BlogSectionEditPage.vue)) | ✅ Fase 3: ajustes del blog, secciones (crear, habilitar con tope, ordenar, borrar), formulario por plantilla con vista previa, subsecciones y `.md` (subir, reemplazar, publicar, mover, descargar, borrar). |

**Datos reales (base de dev, 2026-10-09):**

| user_id | email | username | permiso `blog` |
|---|---|---|---|
| 1 | ruben.d21pc@gmail.com (admin) | `rubendorado` | si (admin, calculado) |
| 2 | rddc0321@gmail.com | — (sin crear) | si |

- ⚠️ El username real del usuario 1 es `rubendorado`, pero `DEMO_BLOG_USERNAME = 'ruben'`: **hoy su blog se ve vacio**. Se arregla cambiando la constante (1 linea) o desaparece en la fase 2, cuando el contenido salga de la base.
- El usuario 2 no tiene URL publica hasta crear su username (P1, seccion 12).

---

## 2. Vision

- Blog por usuario en `/blog/<username>`; `/blog` sin usuario describe la app.
- **Menu horizontal superior** con las secciones que el usuario habilita, **maximo N a la vez** (N configurable, 6 inicialmente).
- **Tipos de seccion** (catalogo en codigo, extensible con nuevos desarrollos):

| Tipo | Instancias | Contenido | Plantilla |
|---|---|---|---|
| **Inicio** (`home`) | 1 | Introduccion general del blog y su proposito; puede presentar proyectos y productos que ofrece | Hecha en el front, datos por **formulario** |
| **Sobre mi** (`about`) | 1 | Introduccion personal | Hecha en el front, datos por **formulario** |
| **Perfil profesional** (`professional`) | 1 | Perfil profesional del usuario | Hecha en el front, datos por **formulario** |
| **Coleccion** (`collection`) | **N** (Proyectos, Libros, How-to, Hobbies...) | Plantilla de front + **subsecciones** creadas por el usuario + en cada una un **listado de `.md`** que al abrirse se renderizan | Hecha en el front; la intro por formulario, los elementos por `.md` |

- Cada usuario **renombra el boton** de cada seccion (nombre a visualizar).
- Dentro de los `.md` (y en los campos de las plantillas) se ven **imagenes por URL**.
- Los elementos de colecciones se **suben y descargan como `.md`**; nada de edicion en linea del `.md` en v1.
- El blog es el dominio de permisos `blog`.

---

## 3. Donde viven los .md — decision: **base de datos** (B0, confirmada)

| Opcion | Veredicto |
|---|---|
| **GitHub** | Solo tenia sentido como "push = publicar" para un unico autor. Con varios usuarios, el backend tendria que gestionar una GitHub App u OAuth por usuario, webhooks, repos ajenos, rate limits y revocacion de tokens: mucha complejidad para 5-10 usuarios, y obliga a cada uno a saber git. **Fuera de v1.** |
| **Archivos en el servidor** | **Incompatible con el deploy planeado.** Cloud Run es *stateless*: el disco es efimero y se pierde en cada redeploy o al escalar a cero. |
| **Base de datos** ✅ | Un `.md` pesa unos KB; 1000 posts son pocos MB, muy por debajo del free tier de Neon. El backup va con el resto, el borrado es transaccional (D4) y no anade infraestructura. |
| **Object storage (R2/GCS)** | Solo se justifica para **imagenes/binarios** alojados por nosotros (no en v1: imagenes por URL externa, B14). |

**Concepto: desacoplar contenido de origen.** Con el `.md` en la base, de donde viene (subida manual, GitHub, editor) es un detalle de ingesta.

**Consecuencia:** `ulm-repository` deja de ser la fuente del blog y queda como repo de documentacion (`how-to/`).

---

## 4. Tipos de seccion: catalogo extensible (B9)

**Patron: registro de tipos** (*plugin registry*; **concepto: principio abierto/cerrado** — se extiende agregando, sin modificar lo existente). Cada tipo se declara una vez en cada lado:

**Backend** (`app/services/blog_section_types.py`):

```python
SECTION_TYPES = {
    "home":         SectionType(singleton=True,  has_items=False, content=HomeContent,         default_label="Inicio",             default_slug="inicio"),
    "about":        SectionType(singleton=True,  has_items=False, content=AboutContent,        default_label="Sobre mí",           default_slug="sobre-mi"),
    "professional": SectionType(singleton=True,  has_items=False, content=ProfessionalContent, default_label="Perfil profesional", default_slug="perfil"),
    "collection":   SectionType(singleton=False, has_items=True,  content=CollectionContent,   default_label="Proyectos",          default_slug="proyectos"),
}
```

- `content` es un modelo Pydantic: valida al escribir y al leer. Cada contenido lleva `schema_version`; un cambio de esquema se resuelve **al leer** (se actualiza el documento viejo en memoria y se guarda en la siguiente escritura), sin migracion de BD — el mismo patron que se usa en DynamoDB.
- Un tipo desconocido en la base (por ejemplo, uno que se retiro) se ignora en la lectura publica en vez de romper el blog (igual que `known_permissions`).

**Frontend** (`src/config/blogSectionTypes.ts`): tipo -> `{ view, editor, icon }` (componentes con carga diferida).

**Agregar un tipo nuevo** (ej. `contact`, `gallery`): 1 modelo Pydantic + 1 entrada en `SECTION_TYPES` + 2 componentes Vue (vista y formulario) + 1 entrada en el registro del front. **Sin migracion, sin rutas nuevas, sin tocar los demas tipos.** Tests: un test parametrizado por tipo (crear, validar, leer publico) cubre cada tipo nuevo al agregarlo al registro.

**Contenido de cada plantilla (propuesta; los formularios lo editan, B10):**

| Tipo | Campos |
|---|---|
| `home` | `headline`, `intro` (markdown corto), `highlights[]` (≤12: `{kind: project\|product\|link, title, description, image_url?, url?}`), `cta?` `{label, url}` |
| `about` | `photo_url?`, `bio` (markdown), `interests[]`, `links[]` `{label, url}` |
| `professional` | `headline`, `summary` (markdown), `experience[]` `{company, role, start, end?, description}`, `education[]`, `skills[]`, `certifications[]`, `links[]` (LinkedIn, GitHub, CV) |
| `collection` | `intro` (markdown corto), `layout` (`list`\|`cards`), `sort` (`date_desc`\|`manual`\|`title`) |

Los campos markdown de las plantillas se renderizan con el mismo pipeline seguro que los `.md` (seccion 9.1). Limites de tamaño por campo en el modelo Pydantic.

**Colecciones: subsecciones y elementos.**
- Una coleccion tiene **subsecciones** (ej. Libros -> "Ficcion", "Tecnicos") creadas por el usuario, con nombre, descripcion opcional y orden. En la vista publica son encabezados/filtros dentro de la seccion.
- Cada **elemento** es un `.md` con frontmatter (`title`, `date`, `slug`, `excerpt`, `tags`, `cover_url`), asignado a una subseccion o a ninguna.
- La URL del elemento **no depende de la subseccion** (B15): `/blog/u/libros/dune`, no `/blog/u/libros/ficcion/dune`. Mover un elemento de subseccion no rompe sus links.

---

## 5. Limite de secciones habilitadas (B13)

- `BLOG_MAX_ENABLED_SECTIONS=6` en `app/core/config.py` (variable de entorno). Se valida en el service al habilitar o crear: 422 con el limite en el mensaje. El front lo recibe en `GET /blog/settings` para deshabilitar el interruptor al llegar al tope.
- Cuentan **solo las habilitadas**; las colecciones cuentan igual que las plantillas.
- **Topes contra abuso** (configurables igual): secciones creadas por usuario (20), subsecciones por coleccion (20), elementos por coleccion (200), `.md` ≤ 200 KB.
- **Carrera** (dos requests habilitando a la vez la 6ª y 7ª): la escritura incrementa la `version` del registro del blog con condicion (`WHERE version = :leida`), lo que serializa los cambios del mismo usuario. Es el equivalente de una *conditional write* de DynamoDB, asi que migra tal cual.
- **UI:** con 6 botones el menu horizontal no cabe en movil -> colapsa a menu desplegable bajo `sm`.

---

## 6. URLs dinamicas (B12)

**Si, pueden ser dinamicas sin mucho problema, y con colecciones multiples son necesarias:** con N colecciones no hay un nombre de plantilla fijo para cada una.

- Cada seccion tiene un `slug` unico por usuario. **Por defecto es el nombre base de la plantilla** (`inicio`, `sobre-mi`, `perfil`) o, en colecciones, derivado del nombre (`libros`, `how-to`).
- Front: rutas genericas `/blog/:username/:section` y `/blog/:username/:section/:item`. El layout ya tiene la lista de secciones (viene en `GET /public/blogs/{username}`), resuelve el slug -> tipo y monta el componente del registro.
- `/blog/:username` (sin seccion) -> la primera seccion habilitada por `position` (redireccion en el front). Sin secciones habilitadas: estado vacio.
- **Renombrar el boton no cambia el slug** (los links compartidos no se rompen). El slug se edita aparte, con aviso; en v1 sin redirecciones del slug viejo.
- Reglas del slug: mismo formato que el username (`^[a-z0-9]+(?:-[a-z0-9]+)*$`, 2-40), unico por usuario, reservados minimos para rutas futuras dentro del blog (`rss`, `feed`, `sitemap`). No chocan con rutas de la app porque viven debajo de `/blog/:username`.
- Slug de elemento: unico **por seccion** (B5: subir uno existente -> 409 salvo `replace=true`).

---

## 7. Modelo de datos: tabla generica pk/sk (B11)

### 7.1 Decision y costo

Decidido (2026-10-09) contra la recomendacion: **una tabla generica estilo DynamoDB** (*single-table design*) en Postgres, para que una migracion futura sea casi mecanica.

Lo que se pierde frente a tablas relacionales, y como se compensa:

| Se pierde | Mitigacion en este diseño |
|---|---|
| FKs entre entidades (seccion -> subseccion -> elemento) | Integridad en el service: borrar una seccion borra sus subsecciones, elementos y cuerpos en la misma transaccion; borrar una subseccion deja sus elementos sin subseccion. Tests de cada caso. |
| FK compuesta que impide mezclar datos de dos usuarios | `user_id` sigue siendo **columna real con FK a `users`** y `CHECK (pk = 'USER#' \|\| user_id)`: la base rechaza una fila cuyo pk apunte a otro usuario. El repository arma el `pk` siempre desde el `user_id` de la sesion, nunca desde el cliente. Tests A/B. |
| `CHECK` sobre columnas tipadas (estado, tipo) | `CHECK` sobre `entity_type`; el resto lo validan modelos Pydantic **al escribir y al leer**. |
| `alembic check` sobre la forma de los datos | Solo cubre las columnas fijas; la forma del `data` la cubren los tests de los modelos Pydantic. |
| Consultas ad-hoc / analytics con SQL comodo | Aceptado: el blog no tiene analytics. Si hiciera falta, Postgres consulta JSONB igual (`data->>'slug'`). |

Lo que se gana: un esquema fijo (agregar tipos y campos no requiere migracion) y una correspondencia 1:1 con DynamoDB.

### 7.2 Tabla

```
blg_entities
  pk            TEXT        NOT NULL          -- 'USER#<user_id>' (particion: todo el blog de un usuario)
  sk            TEXT        NOT NULL          -- tipo + ids, ver 7.3
  user_id       INT         NOT NULL FK users -- redundante con pk a proposito (aislamiento, ver 7.1)
  entity_type   TEXT        NOT NULL          -- CHECK IN ('BLOG','SECTION','GROUP','ITEM','BODY')
  gsi1pk        TEXT        NULL              -- indice secundario: unicidad de slugs
  gsi1sk        TEXT        NULL
  data          JSONB       NOT NULL          -- documento validado por Pydantic segun entity_type / tipo de seccion
  version       INT         NOT NULL DEFAULT 1 -- bloqueo optimista (conditional write)
  created_at    TIMESTAMPTZ NOT NULL
  updated_at    TIMESTAMPTZ NOT NULL
  PRIMARY KEY (pk, sk)
  UNIQUE (gsi1pk, gsi1sk)                     -- slugs unicos; NULL no participa
  CHECK (pk = 'USER#' || user_id)
  CHECK (entity_type IN (...))
```

- Ids de entidades: **ULID generado por la app** (no secuencias: DynamoDB no tiene autoincremento, y los ULID se ordenan por fecha de creacion).
- **Secciones singleton usan el tipo como id** (`SEC#home`): la clave primaria garantiza que no haya dos "Inicio" del mismo usuario, sin indice extra.
- Nombre de la tabla con el prefijo `blg_` y constraints segun la `naming_convention` (D18).

### 7.3 Entidades y claves

| Entidad | `sk` | `gsi1pk` / `gsi1sk` | `data` |
|---|---|---|---|
| Config del blog | `BLOG` | — | `{enabled, display_name?}` |
| Seccion | `SEC#<sectionId>` (`home`/`about`/`professional` o ULID) | `USER#<uid>#SECSLUG` / `<slug>` | `{type, label, slug, position, enabled, schema_version, content{...}}` |
| Subseccion | `GRP#<sectionId>#<groupId>` | — | `{label, description?, position}` |
| Elemento (metadatos) | `ITM#<sectionId>#<itemId>` | `USER#<uid>#SEC#<sectionId>#ITMSLUG` / `<slug>` | `{slug, title, date, excerpt, tags[], cover_url?, group_id?, status, position?, size_bytes, sha256, original_filename}` |
| Cuerpo del `.md` | `BODY#<itemId>` | — | `{content_md}` (original, tal cual se subio) |

El cuerpo va **aparte** de los metadatos: los listados no lo leen (seccion 9.2) y en DynamoDB un registro tiene tope de 400 KB.

### 7.4 Patrones de acceso (todos por clave, ninguno con join)

| # | Necesidad | Consulta |
|---|---|---|
| A1 | Blog publico por username | `users` por `lower(username)` (en Dynamo: GSI `USERNAME#<username>`) |
| A2 | Menu: config + secciones | `pk = USER#<uid> AND (sk = 'BLOG' OR sk LIKE 'SEC#%')` (≤ ~20 filas) |
| A3 | Seccion por slug | Sale de A2 (ya cargado), sin consulta extra |
| A4 | Listado de una coleccion (sin cuerpos) | `pk = USER#<uid> AND sk LIKE 'ITM#<sectionId>#%'` |
| A5 | Subsecciones de una coleccion | `pk = USER#<uid> AND sk LIKE 'GRP#<sectionId>#%'` |
| A6 | Elemento por slug | `gsi1pk = USER#<uid>#SEC#<sectionId>#ITMSLUG AND gsi1sk = <slug>` |
| A7 | Cuerpo del elemento | `pk = USER#<uid> AND sk = BODY#<itemId>` |
| A8 | Contar secciones habilitadas (limite N) | Sale de A2 |

`LIKE 'prefijo%'` sobre la clave primaria usa el indice (equivale a `begins_with` en DynamoDB). Estos patrones son el contrato: **una consulta nueva que no sea uno de ellos se agrega a esta tabla antes de implementarla**, para no perder la propiedad de migrable.

### 7.5 Migracion futura a DynamoDB

- Misma tabla, mismas claves: `pk`/`sk` -> partition/sort key, `gsi1pk`/`gsi1sk` -> GSI1, `data` -> atributos, `version` -> `ConditionExpression`.
- Unicidad de slugs: en Dynamo un GSI **no** es unico; se reemplaza por un registro "candado" por slug escrito en la misma `TransactWriteItems`. Es el unico punto que no migra 1:1.
- `user_id` e `entity_type` quedan como atributos normales.
- El cambio de persistencia se hace en una nueva implementacion del repository (`Protocol`), sin tocar services ni API.

---

## 8. Imagenes por URL (B14)

- Solo `https://` en los `.md` y en los campos de las plantillas (validado con Pydantic `HttpUrl` + esquema https; en el `.md` lo filtra el render).
- Render: `<img loading="lazy" referrerpolicy="no-referrer">`, DOMPurify, CSP `img-src 'self' https:`.
- Riesgos aceptados: una imagen externa puede rastrear a quien visita (*tracking pixel*) y el link se rompe si el sitio de origen la borra. Alojar imagenes propias (R2) queda para despues.

---

## 9. Seguridad y rendimiento

### 9.1 XSS almacenado (*stored XSS*)

El markdown admite HTML crudo. Un usuario podria subir un `<script>` (o `<img onerror=...>`, o un link `javascript:`), y cualquier persona *con sesion iniciada* que abra ese blog lo ejecutaria en el **mismo origen** que la app privada (*same-origin*), con acceso a su sesion y a la API.
- Mitigacion obligatoria: `markdown-it` con `html: false`, salida pasada por **DOMPurify**, links limitados a `http(s)`/`mailto` y CSP sin `unsafe-inline` en scripts. Aplica a los `.md` y a los campos markdown de las plantillas.
- A futuro: servir los blogs publicos desde otro subdominio para separar origen.

### 9.2 Endpoints publicos

- ✅ **404** uniforme (mismo cuerpo) para username inexistente, usuario deshabilitado o sin permiso; se extiende a seccion deshabilitada, slug inexistente y borrador.
- ✅ Esquemas publicos propios ([app/schemas/blog.py](../../ulm-core/app/schemas/blog.py)); nunca id, email, nombre de Google ni avatar. Tampoco ids internos de entidades: el publico navega por slugs.
- ⏳ *Rate limit* por IP (fase 4).

### 9.3 Validacion de la subida

Solo `.md`, UTF-8 valido, ≤ 200 KB, frontmatter YAML con loader seguro (`python-frontmatter` usa `yaml.safe_load` por defecto; no cambiarlo), slug unico por seccion (B5).

### 9.4 Rendimiento

Leer los `.md` de la base **no** es el cuello de botella. Estimaciones (no medidas):

| Paso | Costo tipico |
|---|---|
| Consulta por clave (A2-A7) | ~1-5 ms |
| Red Neon <-> Cloud Run, misma region | ~1-3 ms por consulta |
| Red usuario <-> backend (ej. Bogota -> us-east1) | ~80-150 ms por request |
| Render en el navegador (markdown-it + DOMPurify) | ~1-10 ms por post normal |
| **Cold start de Cloud Run** (scale-to-zero) | **~1-3 s** |
| **Neon suspendido** (tras ~5 min sin uso) | **~0.5-1 s** al despertar |

Con trafico bajo, la primera visita tras un rato inactivo puede tardar 2-4 s, casi todo por arranques en frio. Medidas, en orden de impacto:
- **Cache HTTP en el edge de Cloudflare**: `Cache-Control: public, max-age=60, stale-while-revalidate=86400` + `ETag`. **Concepto: *stale-while-revalidate*** — se entrega la copia guardada al instante y se revalida en segundo plano. Trade-off: un cambio tarda hasta `max-age` en verse (o se purga la cache al publicar).
- **Sin cascada de requests** (*request waterfall*): `GET /public/blogs/{username}` devuelve config **y** secciones habilitadas (con el contenido de las plantillas) en una sola respuesta; una seccion de plantilla no necesita otro request.
- **Listados sin cuerpo**: A4 no lee `BODY#`.
- **`min-instances=1`** en Cloud Run solo si, con la cache, sigue molestando.
- **Medir, no suponer:** header `Server-Timing`.

### 9.5 SEO

Una SPA en Vue se indexa mal. Aceptable en v1; si importa, prerender o SSR.

---

## 10. Username (iteracion 1)

**Donde vive:** en `users`, no en el blog. Es identidad de la **cuenta**: hoy es la URL del blog y a futuro el usuario del login con contraseña. Por eso `PUT /me/username` no exige el permiso `blog`.

**Reglas** ([username.py](../../ulm-core/app/services/username.py), espejo de formato en [lib/username.ts](../../ulm-web/src/lib/username.ts)):
- 3 a 30 caracteres; `^[a-z0-9]+(?:-[a-z0-9]+)*$`.
- Solo ASCII: evita **homoglifos** ("аdmin" con "а" cirilica).
- Se normaliza a minusculas antes de validar y guardar.
- Reservados: rutas de la app actuales o futuras. La lista vive solo en el backend.
- Unico sin distinguir mayusculas. **Concepto: TOCTOU.** La consulta previa da el mensaje claro (409); la garantia real es el indice unico.
- **Inmutable** (B6, B8).

**Para el futuro login con contraseña** (decidido, no implementado): identidad `password` en `user_identities` con `provider_subject = str(users.id)`; login por username **o** email; *rate limit*, bloqueo temporal y argon2/bcrypt obligatorios.

---

## 11. Permisos

- ✅ `Permission.BLOG = "blog"`, sin `blog.ai`.
- `blog` controla quien **gestiona** su blog y si es **visible**. **Ver** un blog no requiere permiso ni sesion.
- ✅ Quitar el permiso oculta el blog (404) sin borrar datos (B2).
- Ademas del permiso, el interruptor `enabled` del registro `BLOG` del propio usuario.

---

## 12. API

**De la cuenta** (solo sesion):

| Metodo | Ruta | Estado |
|---|---|---|
| `GET` | `/me` | ✅ incluye `username` |
| `PUT` | `/me/username` | ✅ crea el username una vez |

**Privada del blog** (requiere `blog`; el repository arma el `pk` desde el usuario de la sesion). ✅ Implementada en la fase 1:

| Metodo | Ruta | Nota |
|---|---|---|
| `GET/PUT` | `/blog/settings` | `enabled`, `display_name`; el GET devuelve tambien `max_enabled_sections`, `max_sections` y el catalogo de tipos. El registro `BLOG` se crea en el primer acceso |
| `GET` | `/blog/sections` | todas (habilitadas o no), en el orden del menu, con su contenido |
| `POST` | `/blog/sections` | `{type, label?, slug?, enabled?}` (deshabilitada por defecto); tipo desconocido -> 422; singleton repetido -> 409; slug explicito tomado -> 409 (uno automatico se resuelve con `-2`, `-3`); topes -> 422 |
| `PUT` | `/blog/sections/order` | `{ids}` con **todas** las secciones exactamente una vez (si no, 422) |
| `PATCH` | `/blog/sections/{id}` | `label`, `slug`, `enabled` (tope N -> 422). El orden va por `/order` |
| `PUT` | `/blog/sections/{id}/content` | contenido de la plantilla (formulario); 422 con la ruta de cada campo invalido |
| `DELETE` | `/blog/sections/{id}` | borra subsecciones, elementos y cuerpos |
| `GET/POST` | `/blog/sections/{id}/groups` | subsecciones (solo colecciones; en plantillas -> 422) |
| `PUT` | `/blog/sections/{id}/groups/order` | `{ids}` con todas las subsecciones exactamente una vez (si no, 422). Fase 3 |
| `PATCH/DELETE` | `/blog/sections/{id}/groups/{groupId}` | `label`, `description`, `position`; borrar deja sus elementos sin subseccion |
| `GET` | `/blog/sections/{id}/items` | listado propio, incluye borradores, sin cuerpo |
| `PUT` | `/blog/sections/{id}/items/order` | orden manual: `{ids}` con todos los elementos exactamente una vez (si no, 422). Fase 3 |
| `POST` | `/blog/sections/{id}/items` | multipart: `file` + `group_id?` (form) + `?replace=true`. Nuevo -> 201 en borrador; slug existente -> **409**, salvo `replace` -> 200 (conserva id, estado y posicion) |
| `GET` | `/blog/sections/{id}/items/{itemId}/download` | el `.md` original, `Content-Disposition: attachment` |
| `PATCH` | `/blog/sections/{id}/items/{itemId}` | `status`, `group_id` (`null` lo saca de la subseccion), `position` |
| `DELETE` | `/blog/sections/{id}/items/{itemId}` | borra tambien el cuerpo |

Todo recurso de otro usuario, o de otra seccion, responde **404**.

**Publica** (sin auth, solo lectura, solo habilitado y `published`; todas con `Cache-Control` + `ETag`):

| Metodo | Ruta | Estado |
|---|---|---|
| `GET` | `/public/blogs/{username}` — `{username, display_name, sections: [{slug, label, type, content}]}`, solo habilitadas y en orden | ✅ |
| `GET` | `/public/blogs/{username}/{sectionSlug}/items` — `{section, groups: [{label, description, items}], ungrouped}`: elementos publicados, sin cuerpo, anidados en su subseccion (sin exponer ids), ordenados segun `content.sort` | ✅ |
| `GET` | `/public/blogs/{username}/{sectionSlug}/items/{itemSlug}` — metadatos + `body_md` (sin frontmatter) + `{section}` | ✅ |

---

## 13. Plan por fases

| Fase | Contenido | Hecho cuando? |
|---|---|---|
| **Iteracion 1** ✅ | Username, permiso `blog`, `GET /public/blogs/{username}`, landing de `/blog`, rutas `/blog/:username/...`, contenido quemado, `/admin/blog` vacio | ✅ commiteada ("Blog structure") |
| **1. Backend** ✅ | `blg_entities` (migracion), repository (`Protocol` + SQLAlchemy sobre pk/sk), registro `SECTION_TYPES` con modelos Pydantic, service (limites, singleton, slugs, borrado en cascada por service, version optimista), API privada completa | `pytest` verde: tests A/B (B no lee, edita ni borra nada de A; `CHECK pk/user_id`), limite N, singleton, slugs unicos, cascada, test parametrizado por tipo; `alembic check` y `check_db_names.py` sin drift. ✅ 2026-10-09: `pytest` 456 verde (55 nuevos), `ruff` limpio, migracion ida y vuelta, sin drift; verificado contra Postgres real (JSONB, prefijos, `CHECK`, version) en transaccion revertida |
| **2. Datos de prueba + lectura publica** ✅ | `scripts/seed_blog.py` (seccion 14), API publica, rutas dinamicas y registro de tipos en el front, 4 vistas, render seguro, menu colapsable en movil, borrar `posts.ts` y `blogDemo.ts` | Payloads XSS neutralizados; 404 uniforme; listado sin cuerpo; cabeceras de cache presentes; los dos blogs de prueba recorren todos los tipos. ✅ 2026-10-13: `pytest` 480 verde (24 nuevos), `ruff` limpio, sin migracion nueva; `vitest` 279 verde (29 nuevos), `vue-tsc`, `eslint` y `vite build` OK; seed y endpoints verificados contra la base de dev y por el proxy de Vite |
| **3. UI de gestion** ✅ | En `/admin/blog`: secciones (crear, habilitar con tope, renombrar, slug, ordenar, borrar), **formulario por plantilla** (B10), subsecciones, subir / descargar / publicar / mover / borrar `.md` | `vitest` + verificacion manual contra backend real. ✅ 2026-10-13: `pytest` 485 verde (5 nuevos), `ruff` limpio; `vitest` 302 verde (23 nuevos), `vue-tsc`, `eslint` y `vite build` OK. ⏳ Falta la verificacion manual en el navegador |
| **4. Endurecimiento** | CSP, *rate limit*, cache de Cloudflare (y purga al publicar), `Server-Timing` | Cabeceras verificadas; tiempo de carga medido en frio y en caliente |
| **Despues** | Imagenes en R2, editor de `.md` en linea, sync con GitHub, RSS, sitemap, SSR/prerender, redirecciones de slugs viejos, login con contraseña, nuevos tipos de seccion | — |

### Fase 3 — detalle (2026-10-13, sin commitear)

Decisiones del usuario: **resumen + editor por seccion** (cada uno con su URL), **flechas subir/bajar** (sin drag & drop), **vista previa** en los campos markdown.

Archivos:
- `ulm-core`: `PUT /blog/sections/{id}/groups/order` y `/items/order` (`app/api/routes/blog.py`, `BlogAdminService.reorder_groups/reorder_items`, schema `OrderWrite`); `InvalidSectionOrderError` pasa a `InvalidOrderError(noun)`, generico. Tests en `test_blog_admin.py`.
- `ulm-web`:
  - Datos: `lib/http.ts` (`postForm`; `request` no fija `Content-Type` con `FormData`), `services/blogAdminApi.ts`, `types/blogAdmin.ts`, `lib/reorder.ts`.
  - Rutas: `admin-blog` -> `views/admin/blog/BlogAdminPage.vue` (resumen; reemplaza la pagina vacia) y `admin-blog-section` -> `views/admin/blog/BlogSectionEditPage.vue` (`navActive: admin-blog`).
  - Componentes (`components/blog-admin/`): `UsernameRequired`, `MoveButtons`, `MarkdownField` (Edit/Preview con `MarkdownContent`), `RepeatableList` (generico: agregar/quitar/ordenar con tope), `LinksField`, `TagsField`, `FormFooter`, `GroupsPanel`, `ItemsPanel`; formularios en `forms/` (`HomeForm`, `AboutForm`, `ProfessionalForm`, `CollectionForm`); `components/ui/ConfirmDialog.vue` (confirmacion generica).
  - Registro `config/blogSectionEditors.ts` (tipo -> formulario + `hasItems`), espejo de `config/blogSectionTypes.ts` del publico.
  - `composables/useSectionContentForm.ts`: copia editable, cambios sin guardar (`onBeforeRouteLeave` + `confirm`), guardado y `nullIfBlank`.
  - Tests: `blogAdmin.spec.ts`, `blogSectionEdit.spec.ts`, `reorder.spec.ts` (incluye `postForm`), helper `tests/helpers/blogAdmin.ts`.

Decisiones tomadas al implementar:
- **Orden atomico** de subsecciones y elementos (`PUT .../order` con la lista completa), igual que el de secciones: subir uno con `PATCH position` serian dos requests.
- **Lista de elementos plana**, en el orden manual, con la subseccion como selector por fila (el plan decia agruparla visualmente): con flechas, agrupar haria confuso mover entre grupos. Las flechas solo aparecen si la coleccion ordena `manual`.
- **Subida en serie** de varios `.md`, con resultado por archivo; un 409 (slug repetido) ofrece "Replace it" (reintenta con `?replace=true`).
- **Descarga sin fetch**: `<a href download>` a la API (mismo origen, la cookie viaja sola).
- La cabecera del editor (nombre, URL, visibilidad) manda **solo lo que cambio**: habilitar puede chocar con el tope y no debe bloquear un cambio de nombre. Cambiar la URL muestra el aviso de links rotos.
- Los formularios limpian antes de mandar (`nullIfBlank`): una URL opcional vacia va como `null` (el backend rechaza `""`). Los errores 422 del backend se muestran tal cual (traen la ruta del campo).
- Las secciones nuevas se crean **deshabilitadas** y se abre su editor.

Limitaciones conocidas:
- Sin verificacion manual en el navegador todavia (cubierto por tests de componentes).
- Los cambios tardan hasta 60 s en verse en el blog publico (cache); el admin lo avisa.
- No hay "descartar cambios" en los formularios: se recarga la pagina (pide confirmacion).

### Fase 2 — detalle (2026-10-13, sin commitear)

Archivos:
- `ulm-core`: `app/services/blog_reads.py` (nuevo: `load_sections` y `sort_items`, compartidos por el admin y el publico), `blog_service.py` (lectura publica), `blog_admin_service.py` (usa `blog_reads`), `app/schemas/blog.py` (schemas publicos), `app/api/http_cache.py` (nuevo), `app/api/routes/public_blogs.py`, `app/api/deps.py`, `scripts/seed_blog.py` (nuevo); tests: `test_blog_public.py` y `test_seed_blog.py` (nuevos), `test_blog.py`.
- `ulm-web`: `lib/markdown.ts`, `components/blog/{MarkdownContent,BlogItemCard,BlogLinks,BlogNoContent}.vue`, `types/blog.ts`, `services/publicBlogApi.ts`, `composables/useBlogOwner.ts`, `config/blogSectionTypes.ts`, `router/routes/public.routes.ts`, `layouts/PublicLayout.vue`, `views/public/blog/*` (6 vistas nuevas); borrados `data/posts.ts`, `data/blogDemo.ts`, `views/public/{HomePage,AboutPage,ProjectsPage,BlogListPage,BlogPostPage}.vue` y `tests/BlogPostPage.spec.ts`; tests: `markdown.spec.ts` (nuevo), `publicBlog.spec.ts` (reescrito). Dependencias nuevas: `markdown-it`, `dompurify`, `@types/markdown-it`.

Decisiones tomadas al implementar:
- **Elementos anidados en su subseccion** en la respuesta publica (`groups[].items` + `ungrouped`): asi no se expone el id de la subseccion y el front no tiene que agrupar.
- **Cache solo en 200**: un 404 no se guarda (un blog puede aparecer). `ETag` = sha256 del cuerpo; `If-None-Match` igual -> 304 sin cuerpo.
- **Sin registro BLOG** (el usuario nunca entro al modulo) el blog se trata como habilitado y sin secciones; con `BLOG.enabled = false`, 404.
- **Seed con 6 habilitadas + 1 deshabilitada** para `rubendorado` (el plan decia 5+1, que no llega al tope). El seed pasa por `BlogAdminService`: un dato invalido falla igual que en la API, y `test_seed_blog.py` lo corre en CI. Sin `--reset`, si un usuario ya tiene blog el script no lo toca (protege datos reales).
- **Render seguro** (`lib/markdown.ts`): markdown-it sin HTML + validacion de links (http(s), mailto, `#`, rutas propias; no `//otro-sitio`) + imagenes solo https (una http se muestra como su texto alternativo) + DOMPurify. Detalle: con un `ALLOWED_URI_REGEXP` propio, DOMPurify valida contra el **todos** los atributos que no considera seguros, asi que la expresion tambien acepta valores sin `:` ni `/` (`_blank`, `lazy`).
- Menu: la seccion queda marcada tambien dentro de uno de sus elementos (se compara el parametro `section`, porque la ruta del elemento es hermana de la de la seccion).
- Un tipo de seccion que el front no conoce no sale en el menu (el backend ya omite los que no estan en su catalogo).

Limitaciones conocidas:
- La cache de 60 s tambien aplica al navegador del autor: un cambio puede tardar hasta un minuto en verse en el blog publico (no en `/admin/blog`, que usa la API privada).
- CSP y *rate limit* siguen pendientes (fase 4).
- No hubo verificacion visual en un navegador: la cubren los tests de componentes y la prueba de los endpoints por el proxy.
- Ajuste posterior (2026-10-13): el encabezado del blog publico pasa a todo el ancho (marca a la izquierda, menu al centro, Admin y usuario a la derecha; desplegable bajo `lg`) y el contenido de `max-w-3xl` a `max-w-5xl` (articulos en `max-w-3xl` por legibilidad; grillas de tarjetas a 3 columnas en `lg`).

### Fase 1 — detalle (2026-10-09, sin commitear)

Archivos (`ulm-core`):
- Modelo y migracion: `app/db/models/blog.py` (`BlgEntity`), `migrations/versions/4c07b2a15e70_agrega_blg_entities.py`; `migrations/env.py` y `tests/conftest.py` importan el modelo.
- Claves y persistencia: `app/repositories/blog_keys.py` (unico lugar que conoce el formato de `pk`/`sk`/`gsi1`), `blog_entity_repository.py` (`Protocol` generico: get, query por prefijo, get por indice, create, delete, flush) y `sqlalchemy_blog_entity_repository.py`.
- Dominio: `app/services/blog_section_types.py` (registro de tipos + modelos de contenido), `blog_documents.py` (forma de `data` de cada entidad), `blog_slugs.py`, `blog_markdown.py` (validacion del `.md` y su frontmatter), `blog_admin_service.py`; errores nuevos en `errors.py` (incluido `ConcurrentUpdateError`, generico).
- API: `app/schemas/blog_admin.py` (separado de los publicos), `app/api/routes/blog.py` (en `PRIVATE_ROUTERS` con `Permission.BLOG`), `app/api/deps.py`, `app/main.py`.
- Configuracion: `BLOG_MAX_*` en `app/core/config.py` y `.env.example`. Dependencias nuevas: `python-multipart` (subida de archivos) y `python-frontmatter`. `pyproject.toml`: `fastapi.File`/`fastapi.Form` en las llamadas permitidas de B008.
- Tests: `tests/test_blog_admin.py` (nuevo), `tests/test_permissions.py` (`/blog/sections` exige `blog`).

Decisiones tomadas al implementar:
- **Bloqueo optimista con `version_id_col` de SQLAlchemy**: todo UPDATE de `blg_entities` lleva `WHERE version = <leida>`; si otro request la cambio, `ConcurrentUpdateError` -> 409. Los cambios que afectan al conteo de secciones reescriben el registro `BLOG`, que hace de candado del tope N.
- **Repository generico (llave-valor)** en vez de un metodo por patron de acceso: es exactamente la interfaz de DynamoDB, y los patrones (7.4) viven en el service, que es el unico que conoce las claves (via `blog_keys`).
- Ids **ULID** propios (`app/core/ids.py`, sin dependencia); las secciones singleton usan el tipo como id (`SEC#home`).
- Un elemento nuevo queda en **borrador**; reemplazarlo conserva id, estado, posicion y subseccion (salvo que se mande otra).
- Un tipo de seccion que ya no esta en el catalogo se omite al listar (con log) pero sigue contando para los topes.
- Las URLs de imagen se normalizan con `HttpUrl` de Pydantic (`https://ulm.dev` -> `https://ulm.dev/`).
- El `.md` se guarda **completo** (con frontmatter): la descarga es identica a lo subido. La lectura publica (fase 2) quitara el frontmatter al renderizar.


### Iteracion 1 — detalle (2026-10-04)

Archivos:
- `ulm-core`: `app/db/models/user.py`, `migrations/versions/e4b19c7d2a60_agrega_username_a_users.py`, `app/services/{username,blog_service,user_admin_service,permissions,errors}.py`, `app/repositories/{user_repository,sqlalchemy_user_repository}.py`, `app/schemas/{user,blog}.py`, `app/api/routes/{me,public_blogs}.py`, `app/api/deps.py`, `app/main.py`; tests: `tests/test_blog.py`, `test_auth.py`, `test_permissions.py`.
- `ulm-web`: `router/routes/{public,admin}.routes.ts`, `layouts/PublicLayout.vue`, `views/public/*`, `views/admin/BlogAdminPage.vue`, `components/account/UsernameCard.vue`, `components/blog/BlogNoContent.vue`, `composables/{useAuth,useBlogOwner}.ts`, `data/blogDemo.ts`, `lib/{username,userPermissions}.ts`, `services/{authApi,publicBlogApi}.ts`, `config/{nav,permissions}.ts`, `components/layout/TopBar.vue`, `components/params/UsersPanel.vue`; tests: `username.spec.ts`, `publicBlog.spec.ts`, `BlogPostPage.spec.ts`.

Limitaciones conocidas:
- `DEMO_BLOG_USERNAME = 'ruben'` no coincide con el username real (`rubendorado`). Desaparece en la fase 2.
- Sin *rate limit* en el endpoint publico.
- Ajeno: `tests/permissions.spec.ts` emite 3 *Unhandled Errors* de `RecordsPage.vue` (`rawMeals.value is not iterable`); los tests pasan igual.

---

## 14. Datos de prueba (fase 2)

- **Script de desarrollo idempotente**, `ulm-core/scripts/seed_blog.py`, **no** una migracion (leccion de D20). Solo escribe en `blg_entities` y solo para usuarios con username; reejecutarlo reemplaza los datos de prueba sin duplicar.
- Debe cubrir **todos los tipos** entre los dos usuarios:

| Usuario | Secciones |
|---|---|
| 1 (`rubendorado`) | `home` (con highlights de proyectos y productos), `about` (migrando el texto de `AboutPage.vue`), `professional`, colecciones **Proyectos** (2 subsecciones, elementos de `ProjectsPage.vue`) y **Escritos** (los 3 posts de `posts.ts` como `.md`, con imagenes por URL) , **How-to** (borrador + elemento sin subseccion) = **6 habilitadas** (el tope); **Libros** creada deshabilitada (habilitarla da 422). `home` con label renombrado ("Bienvenida", slug `inicio`) |
| 2 (`rubendorado2`) | `home`, `about` deshabilitada, `professional`, colecciones **Libros** (subsecciones "Ficcion" y "Tecnicos", un borrador) y **Hobbies** (sin subsecciones) |

- Casos que el seed debe ejercitar: elemento en borrador (no visible en publico), elemento sin subseccion, seccion deshabilitada, `.md` con imagen por URL, label renombrado distinto del nombre base y slug por defecto.

---

## 15. Decisiones

**Cerradas:**

| id | Decision | Fecha |
|---|---|---|
| **B0** | Contenido en base de datos (no GitHub, no disco del servidor) | 2026-10-04 |
| **B1** | URL publica `/blog/<username>`; secciones debajo | 2026-10-04 |
| **B2** | Sin permiso `blog` (o usuario deshabilitado) el blog responde 404; no se borran datos | 2026-10-04 |
| **B3** | `/blog` sin usuario = descripcion de la app; `/` redirige ahi | 2026-10-04 |
| ~~**B4**~~ | ~~Tipos `page`/`collection` fijos por `section_key`~~ -> reemplazada por **B9** | 2026-10-04 |
| **B5** | Subir un slug existente responde 409; reemplazar exige `replace=true` | 2026-10-04 |
| **B6** | El identificador se llama `username`, vive en `users` y es inmutable en v1 | 2026-10-04 |
| **B7** | Mientras el contenido sea quemado, solo el blog de `DEMO_BLOG_USERNAME` lo muestra | 2026-10-04 |
| **B8** | Sin herramienta para cambiar username hasta que aparezca el caso; si se construye: solo admin, con redireccion y el viejo bloqueado | 2026-10-04 |
| **B9** | Catalogo extensible de tipos de seccion: `home`, `about`, `professional` (1 por usuario, plantilla del front) y `collection` (N por usuario, cada una en el menu, con subsecciones y `.md`). Registro de tipos en back y front | 2026-10-09 |
| **B10** | El contenido de las plantillas se carga con **formularios por plantilla** en `/admin/blog` (no `.md` con frontmatter) | 2026-10-09 |
| **B11** | Modelo: **tabla generica pk/sk/JSONB** (`blg_entities`) estilo DynamoDB, con `user_id` como columna real (FK + `CHECK`) para conservar el aislamiento | 2026-10-09 |
| **B12** | URLs dinamicas por `slug` de seccion (por defecto el nombre base de la plantilla); renombrar el boton no cambia el slug | 2026-10-09 |
| **B13** | Maximo de secciones habilitadas a la vez configurable (`BLOG_MAX_ENABLED_SECTIONS`, 6); topes adicionales contra abuso | 2026-10-09 |
| **B14** | Imagenes solo por URL `https`, en `.md` y campos de plantilla | 2026-10-09 |
| **B15** | La URL de un elemento no depende de su subseccion (`/blog/u/<seccion>/<elemento>`) | 2026-10-09 |
| **B16** | Datos de prueba por script idempotente (`scripts/seed_blog.py`), no por migracion | 2026-10-09 |
| **B17** | El seed deja a `rubendorado` justo en el tope (6 habilitadas) mas una 7ª deshabilitada | 2026-10-13 |
| ~~**P1**~~ | Username del usuario 2: lo creo el usuario (`rubendorado2`) | 2026-10-13 |

**Pendientes:**

| id | Pregunta | Recomendacion |
|---|---|---|
| **P2** | ¿Que pasa con un slug de seccion viejo al cambiarlo? | v1 sin redireccion (aviso en la UI); despues, guardar el historial y redirigir |

---

## 16. Recomendaciones

- **Terminar primero la fase 5 de auth (onboarding).** El blog es el primer modulo publico; conviene tener el multiusuario cerrado antes de exponerlo.
- **Los patrones de acceso (7.4) son el contrato del modelo.** Con una tabla generica, la disciplina que antes imponian las FKs pasa al codigo: ninguna consulta fuera de esa lista sin agregarla antes.
- **Validar `data` al leer, no solo al escribir.** Con JSONB, una fila escrita a mano o por una version vieja del codigo solo se detecta al leerla.
- **La descarga devuelve el original**, no una regeneracion.
- **Renderizar en el front, sanitizar siempre** (DOMPurify sobre el HTML final, tambien en los campos markdown de las plantillas).
- **Esquemas publicos separados de los privados** (ya aplicado en `app/schemas/blog.py`).
