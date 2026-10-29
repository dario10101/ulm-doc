# Funcionalidades implementadas

Ultima revision: 2026-10-29. Solo lo que funciona end-to-end (backend + UI). Lo parcial va en [capabilities_in-progress.md](./capabilities_in-progress.md).

**Genericidad** (que tan facil le sirve a cualquier usuario, sin cambios):
- **Alta**: util para cualquiera tal cual.
- **Media**: util para otros, pero con supuestos mios (formato, flujo, catalogos).
- **Baja**: hecha a mi medida; para otro usuario habria que rediseñarla.

## Plataforma

| Funcionalidad | Gen. | Nota |
|---|---|---|
| Login con Google por invitacion; admin por config | Alta | Sin contraseña todavia |
| Permisos por dominio (`weight`, `meals`, `finances`, `planning`, `blog`, `english`, `ai`, `<dominio>.ai`) | Alta | |
| Administracion de usuarios (invitar, permisos, deshabilitar) en Settings -> Users | Alta | |
| Datos aislados por usuario | Alta | |
| Tema claro/oscuro por dispositivo | Alta | |
| Parametros: catalogos con borrado hibrido (borra sin registros, archiva con registros) | Alta | |
| Festivos (CO) propuestos por libreria + fechas especiales, administrados por el admin | Media | Pais fijo |

## Planificacion (`planning`)

| Funcionalidad | Gen. | Nota |
|---|---|---|
| Checklist semanal: categorias, template, semana creada desde template, puntaje al cerrar | Media | Semana sabado-viernes de 7 dias exactos, puntos HIGH=2/STANDARD=1 |
| Tareas circunstanciales, modo Track/Edit, detalle por tarea | Alta | |
| Calendario Daily/Weekly/Monthly/Yearly con tareas puntuales y recurrentes, en la zona horaria del usuario | Alta | |
| Sincronizacion tarea de calendario -> checklist semanal | Media | |
| Eventos personales (CRUD, tipo libre) | Alta | |
| Analitica: tendencia de puntaje del checklist | Media | |

## Finanzas (`finances`)

| Funcionalidad | Gen. | Nota |
|---|---|---|
| Gastos con categorias por usuario (copiadas de plantilla global), metodos de pago y tags | Alta | Metodos de pago aun globales |
| Ingresos directos e intereses mensuales por fuente/subcategoria (bolsillos) | Media | Modelo de intereses pensado en mis productos (Tyba, Lulo) |
| Analitica de gastos, ingresos e ingresos vs gastos (neto, tasa de ahorro) | Alta | |
| Formato de montos colombiano | Baja | `.` miles, `,` decimal; no configurable |

## Salud (`weight`, `meals`)

| Funcionalidad | Gen. | Nota |
|---|---|---|
| Registro de peso (CRUD) + tendencia | Alta | |
| Registro de comidas con composicion en porcentajes | Media | Formato de contenido propio |

## Contenido y aprendizaje

| Funcionalidad | Gen. | Nota |
|---|---|---|
| Blog publico por usuario en `/blog/<username>`: secciones por tipo, colecciones de `.md`, cache ETag | Alta | Ver [PLAN-BLOG.md](../plans/PLAN-BLOG.md) |
| Admin del blog: secciones, formularios por plantilla, subir/publicar/mover `.md` | Alta | |
| English: banco compartido de palabras (CEFR), practica vocab -> listening -> writing -> speaking, feedback LLM on demand, sugerencia de nivel | Alta | Ver [ENGLISH-TEACHER-PLAN.md](../plans/ENGLISH-TEACHER-PLAN.md) |
| Chat IA suelto (`/ai/ask`, sin historial ni datos del usuario) | Alta | Poco valor hoy; base para insights |

## Captura

| Funcionalidad | Gen. | Nota |
|---|---|---|
| Add record (alta rapida) y View records (listado/edicion por tipo) | Alta | Tipos `workout` y `note` visibles pero no implementados |
