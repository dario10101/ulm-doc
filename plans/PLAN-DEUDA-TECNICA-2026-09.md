# ULM - Plan de deuda tecnica (sep 2026)

Estado: **cerrado** (fases 1-6 ejecutadas al 2026-09-27). Historico de referencia: que se hizo y que trampas aparecieron.
Lo que quedo abierto (D10, D15, D17, D19, D20) se movio a [technical-pending.md](../technical/technical-pending.md) y [capabilities_wish-list.md](../functional/capabilities_wish-list.md); las reglas permanentes a [technical-design.md](../technical/technical-design.md).

---

### Fase 6 (parcial) — Renombre de tablas y D11

**Renombre.** Las tablas de un usuario ahora lo dicen en el nombre, como ya hacia finanzas: `cl_categories` → `cl_user_categories`, `cl_template_tasks` → `cl_user_template_tasks`, `cl_week` → `cl_user_weeks`, `cld_tasks` → `cld_user_tasks`, `meals` → `user_meals`, `weights` → `user_weights`; y `cl_tasks` → `cl_week_tasks` (cuelga de la semana, no del usuario). Las rutas de la API y las clases ORM no cambiaron.

Cuatro migraciones escritas a mano, cada una con un solo proposito:

| Revision | Que hace |
|---|---|
| `2f90f7eeef16` | Borra 16 indices `ix_<tabla>_id` que duplicaban el de la PK (venian de `primary_key=True, index=True`, patron del tutorial de FastAPI). Va primero para no renombrar indices condenados. |
| `4d0cb87197ac` | Renombra las 6 tablas con su secuencia, PK, FK e indices explicitos, incluidas las FK de tablas **no** renombradas que apuntan a ellas. |
| `a8e4ccee81be` | `cl_tasks` → `cl_week_tasks`, e indexa `cl_week_id`: la tabla que mas crece no tenia indice en la columna por la que siempre se consulta. |
| `27dd878d152b` | D11 (abajo). |

**`scripts/check_db_names.py`**, nuevo y en CI: compara los nombres de PK, FK, UNIQUE, indices y secuencias de la base contra los que generan los modelos. Cubre el punto ciego de `alembic check`, que ya habia aparecido con D18. Se verifico que detecta el drift corriendolo contra la base **antes** de migrar.

**D11 — constraints en BD.**
- **Nombre de categoria unico por usuario**, sin distinguir mayusculas y **contando las DISABLED**. Asi analytics no parte una misma categoria en dos. Es un indice unico sobre `(user_id, lower(name))`: una UNIQUE constraint de Postgres no admite expresiones.
- **Una sola semana abierta por usuario**, con un indice unico *parcial* (`WHERE NOT closed`).

El service valida lo mismo antes de escribir y responde 422 (nombre repetido en el guardado) o 409 (nombre de una deshabilitada, con la sugerencia de reactivarla). Un handler nuevo de `IntegrityError` convierte en 409 lo que se le escape al service, en vez de un 500. Los nombres se recortan en el schema: `"Salud "` es `"Salud"`. El front replica la regla (`src/lib/categoryNames.ts`) para avisar sin ir al servidor.

Una trampa que aparecio: el indice se valida fila por fila, asi que **intercambiar dos nombres** (A→"B", B→"A") chocaba en el primer UPDATE. `replace_categories` ahora renombra en dos pasos, pasando antes por un nombre temporal.

Verificado:
- Postgres rechaza cada caso por SQL directo.
- Los 6 tests nuevos fallan con el bug inyectado (sin renombre en dos pasos, sin validacion previa, sin el indice parcial).
- El ciclo de CI completo pasa sobre una base vacia.
- Downgrade y upgrade sobre la base real conservan los datos.

**Encontrado de paso:** la migracion de finanzas (`31bdd71b624a`) **no corria sobre una base vacia**. Sembraba tags para `user_id=1`, que ahi no existe todavia, asi que el job `postgres` de CI estaba en rojo desde ese commit. Se corrigio salteando la siembra si el usuario no existe. Editar una migracion ya aplicada es seguro solo porque, en las bases donde ya corrio, el resultado es identico. Lo que queda abierto es **D20**.

### Zona horaria

La BD guarda UTC; cada usuario tiene su zona IANA en `users.timezone`; toda cuenta de calendario se hace convirtiendo primero a esa zona. La API habla hora de pared en ambos sentidos y rechaza con 422 cualquier fecha con offset. Verificado end-to-end contra Postgres, con 5 tests que fallan si se rompe la conversion.

Contrato: [ulm-core/README.md](../../ulm-core/README.md#zona-horaria) · implementacion: `app/services/cld_task_sync.py` · fundamentos: [how-to/timezones.md](../../ulm-repository/how-to/timezones.md).
Cola abierta: **D19** (cambiar de zona).

### Fase 1 — Alembic y aislamiento de tests

**D1 — Alembic.** Linea base `cf2ce11c7a8b` generada contra una BD vacia, BD real sellada con `stamp head`, `create_all()` fuera del arranque. Verificado: `alembic check` sin drift, y una BD creada desde cero produce un esquema **identico** al de produccion. Flujo de uso: [ulm-core/README.md](../../ulm-core/README.md#migraciones-de-base-de-datos) · fundamentos: [how-to/alembic.md](../../ulm-repository/how-to/alembic.md).
Cola abierta: **D18** (convencion de nombres).

El diff de la linea base revelo drift real, ya corregido: tres columnas (`cl_categories.status`, `cld_tasks.duration_minutes`, `users.timezone`; nombres de entonces) tenian `DEFAULT` en la BD que los modelos no declaraban, residuo de los `ALTER TABLE` manuales. Los modelos ahora declaran ese `server_default`.

### Fase 3 — Unit of Work y CRUD de peso (D4, D16)

**D4 — una transaccion por request.** El commit vive ahora solo en `get_db` (`app/db/session.py`). Los repositories ya no exponen `commit()`/`refresh()`: exponen `flush()`, que los services usan donde necesitan un id generado. O se escribe la operacion completa, o no se escribe nada.

Un test lo demuestra en vez de asumirlo (`test_failed_week_creation_leaves_no_week_behind`): fuerza un fallo justo despues de crear la semana, al copiar las tareas del template, y verifica que no quede ninguna semana — ni siquiera una abierta y vacia que bloquee crear la siguiente. Verificado que el test **atrapa el bug**: reintroduciendo un commit a mitad, falla.

El `conftest` se alineo con produccion (el override de `get_db` ahora commitea igual), asi que los tests ejercitan el mismo camino que la app real. Y se verifico end-to-end contra Postgres: una escritura normal persiste, y un 404 de dominio no deja filas.

Dos cosas que hubo que resolver:

- **`expire_on_commit=False`** en el `sessionmaker`. Al commitear, SQLAlchemy expira los atributos de todos los objetos de la sesion; como el commit ahora ocurre al final del request, esa recarga caeria fuera de la transaccion.
- **Un `delete()` sin flush no llega a la base.** Con `autoflush=False`, borrar y volver a consultar en el mismo request devolvia el objeto desde el identity map. El repository de peso necesitaba un `flush()` explicito.

**Limitacion conocida:** FastAPI ejecuta el codigo posterior al `yield` de una dependencia **despues de enviar la respuesta**. Un fallo del commit en si (no de las escrituras, que ya se validaron en el flush) se registra en el log pero no alcanza a cambiar la respuesta. Queda documentado en `get_db`.

**D16 — Edit/Delete de peso.** `updateWeight`/`deleteWeight` en el servicio, un `WeightFormDialog` para la edicion y confirmacion antes de borrar. Detalle que se cuido: si se borra el ultimo registro de una pagina, se retrocede una pagina para no dejar la vista vacia. Dos tests nuevos, y verificado contra el backend real (PUT 200, DELETE 204, segundo DELETE 404).

### Fase 2 — CI, linters y convencion de nombres (D3, D18)

**CI.** `.github/workflows/ci.yml` en ambos repos, en cada push y pull request.

- `ulm-core`: tres jobs — lint (`ruff check` + `ruff format --check`), tests (`pytest` sobre SQLite, rapido), y **postgres** (levanta un Postgres 16 de servicio, corre `alembic upgrade head` desde cero, `alembic check`, `scripts/check_db_names.py`, `alembic downgrade base` y otra vez `upgrade head`).
- `ulm-web`: `npm ci` + eslint + prettier + `vue-tsc` + vitest.

El job de Postgres es el que justifica el gasto: es lo unico que atrapa las divergencias con SQLite, que en este proyecto ya mordieron dos veces. Y `alembic check` impide subir un modelo cambiado sin su migracion.

Cada paso se simulo localmente antes de darlo por bueno: los tres jobs de core pasan, y las 3 migraciones suben y bajan limpias (`downgrade base` deja solo `alembic_version`).

**Linters.** `ruff` en core (`pyproject.toml`, `line-length = 100` para no reformatear medio proyecto) y `eslint` + `prettier` en web. Tres reglas se apagaron con razon documentada, no por comodidad:

- **B008** (`function-call-in-default-argument`): son los `Depends()` de FastAPI, que son LA forma de declarar dependencias, no el bug de argumento mutable que la regla busca.
- **UP042** (`replace-str-enum`): migrar `class X(str, Enum)` a `StrEnum` cambia el resultado de `str()` y afectaria la serializacion de Pydantic.
- **no-undef** en TS/Vue: TypeScript ya detecta identificadores inexistentes y conoce los tipos del DOM; la regla solo producia falsos positivos sobre `document`, `HTMLElement`, etc.

De paso se separo `requirements-dev.txt` de `requirements.txt`, para que la imagen de produccion (D17) no arrastre pytest ni ruff.

**D18 — convencion de nombres.** `naming_convention` en la `MetaData` y una migracion (`4abfa186c2c2`) que renombra los 23 constraints que Postgres habia bautizado a su manera.

Lo que hizo falta entender: **`alembic check` daba "sin cambios"** con la convencion ya puesta y la base todavia con los nombres viejos, porque autogenerate compara estructura y no nombres. El drift existia pero era invisible, y habria aparecido recien el dia que una migracion intentara borrar un constraint por un nombre inexistente. La migracion se escribio a mano con `ALTER TABLE ... RENAME CONSTRAINT` (solo metadata: no reescribe datos), y es reversible — probado bajando y volviendo a subir.

Verificado: 24 constraints en convencion, 0 fuera, y una base creada desde cero sigue siendo identica a produccion.

### Fase 5 — Aislamiento, errores, mapeo y observabilidad

**D9 — errores de dominio unificados.** `app/services/errors.py` con una jerarquia (`DomainError` → `NotFoundError` / `ConflictError` / `ValidationError`). Desaparecieron las 3 definiciones de `CategoryNotFoundError` y, con ellas, los alias en los imports de las rutas (`CategoryNotFoundError as UnknownCategoryIdsError`).

**D6 — aislamiento por usuario en peso.** `update`/`delete` filtran por `user_id` en el repository, asi que un registro ajeno responde **404 y no 403**: distinguirlos le confirmaria a un usuario que ese id existe. Tres tests nuevos lo demuestran (editar, borrar y listar registros de otro usuario).

**D7 — `cl_template_tasks.user_id`.** Primera migracion real con Alembic (`56b5e7da78de`). El `--autogenerate` proponia dos cosas que habrian fallado y hubo que reescribirlas a mano:

- `add_column(nullable=False)` sin relleno: revienta con la primera fila existente (habia 10). Se aplico expandir / rellenar / contraer, con el backfill tomando el dueño de la categoria, que es de donde se derivaba hasta ahora.
- un foreign key **sin nombre**, cuyo `drop_constraint(None)` del downgrade falla. Alembic incluso lo avisa. Es exactamente **D18**: sin convencion de nombres hay que acordarse de esto en cada migracion.

Verificado: 10 filas rellenadas, 0 discrepancias contra la categoria, `alembic check` sin drift. El service ya no deriva la pertenencia por join (un query menos por operacion).

**D8 — el ORM deja de llegar a las rutas.** `app/services/mappers.py` concentra la traduccion entidad → esquema, y los services devuelven esquemas. Ninguna ruta importa ya `app.db.models` (salvo `dummy`, que es el endpoint de prueba deliberado). Ahora si se cumple lo que prometian los `Protocol`: cambiar de persistencia no obliga a tocar la capa de API.

**D12 — logging y errores no controlados.** `app/core/logging.py` (formato y nivel unicos, `LOG_LEVEL` por configuracion) reemplaza el `print()` del arranque, y un `@app.exception_handler(Exception)` registra el traceback con metodo y ruta pero **devuelve solo un 500 generico**: el detalle de un fallo interno no le sirve al cliente y puede filtrar informacion del sistema. Verificado con una ruta que lanza a proposito.

### Fase 4 — CalendarPage (D5, D14, D13)

`CalendarPage.vue` paso de **2.177 a 462 lineas**, como orquestador. Se hizo en el orden del plan, de lo mas seguro a lo mas riesgoso, verificando typecheck + tests + la pagina real en cada paso.

Logica pura extraida, **con tests** (de 21 a 61 tests de frontend):

| Modulo | Que contiene |
|---|---|
| `lib/timeline.ts` | posicionamiento y reparto en carriles de tareas que se solapan |
| `lib/monthGrid.ts` | grilla mensual, colores de eventos, celdas |
| `lib/taskForm.ts` | validacion del formulario de tarea (lo unico realmente duplicado entre alta y edicion) |
| `lib/time.ts` | reloj de 12 horas y armado de fechas (D14) |
| `lib/eventTypes.ts`, `lib/dom.ts` | tipos de evento del filtro; scroll dentro de un contenedor |

Componentes: `DailyView`, `WeeklyView`, `MonthlyView`, `YearlyView`, `EventTypeFilter`, y los dialogos `TaskDetailDialog`, `TaskFormDialog`, `TaskEditDialog`, `DeleteTaskDialog` (cada dialogo es autocontenido: tiene su estado, su submit y su error).

Composables: `useCalendarSection` (unifica los 4 cargadores, que eran el mismo codigo con otro rango) y `useCategories` (**D13**: cache compartida, elimina 4-5 fetches redundantes).

Tres cosas que aparecieron al hacerlo:

- **`to12Hour` no estaba duplicada: eran dos funciones distintas con el mismo nombre.** En `CalendarPage` recibia un `Date`; en `QuickAddPage`, minutos. Ahora son `clockFromDate` y `clockFromMinutes`.
- **No se unificaron los formularios de alta y edicion.** Tienen 11 campos comunes pero 7 y 4 exclusivos: unificar habria dado el componente lleno de `v-if="mode === 'edit'"` que el propio plan desaconsejaba. Lo realmente compartido (la validacion) se extrajo a `lib/taskForm.ts`.
- **Una regresion evitada:** al mover `selectedDay` dentro de `MonthlyView`, pulsar "Today" estando ya en el mes actual dejaba de reseleccionar el dia (el watcher del cursor no se disparaba). Se resolvio dejandolo como `v-model` del padre.

Queda pendiente de verificacion **manual en el navegador** el checklist de la Fase 4 (las 4 vistas, navegacion, alta/edicion/borrado): typecheck, tests y compilacion estan verdes, pero las vistas siguen sin tests de componente.

**D2 — `conftest.py`.** Un solo engine, una transaccion por test revertida al final, `dependency_overrides` limpiado por fixture, y eliminados los workarounds que existian para esquivar la fuga.

Dos trampas que aparecieron y quedan documentadas donde viven:

- **pysqlite rompe SAVEPOINT** por su manejo implicito de transacciones: sin el workaround de SQLAlchemy los `commit()` de la app se aplican de verdad y el rollback no los deshace. El aislamiento *parece* funcionar. Ver el comentario en `tests/conftest.py`.
- **7 tests dependian del estado que dejaba el anterior**. Ya son autocontenidos, via `_open_week_with_template_tasks` y `_create_and_close_week`.
