# Pendientes tecnicos

Los ids `D*` vienen del plan de deuda original ([PLAN-DEUDA-TECNICA-2026-09.md](../plans/PLAN-DEUDA-TECNICA-2026-09.md)). Pendientes funcionales en [capabilities_wish-list.md](../functional/capabilities_wish-list.md).

| id | Prioridad | Pendiente |
|---|---|---|
| T1 | Alta | **`doc/` y la raiz no estan en git**: sin historial ni respaldo, y ~50 comentarios de codigo versionado citan planes que viven aqui. Opciones: repo propio para la raiz (`ulm-docs`) o mover `doc/` a `ulm-repository`. |
| T2 | Alta | `ulm-repository/how-to/` sin commitear y el repo sin remoto. |
| D17 | Alta (bloquea deploy) | Sin Dockerfile ni `docker-compose`; `ulm-infra` vacio. Al crearlo: instalar `tzdata` (slim no trae IANA) y `alembic upgrade head` antes de arrancar. |
| D20 | Media | Siembra de datos por usuario dentro de migraciones; va en el registro del usuario. Se resuelve con F1 (plantilla de usuarios nuevos). |
| T3 | Media | `/ai/ask` no registra en `ai_usage_events` (solo log). |
| T4 | Media | 3 errores no capturados en `ulm-web/tests/permissions.spec.ts` (mock de comidas). |
| T5 | Media | Vistas del calendario sin tests de componente; verificacion solo manual. |
| T6 | Baja | `POINTS_BY_IMPORTANCE` en `app/schemas/` siendo regla de negocio. |
| T7 | Baja | Router `dummy` montado en la app real. |
| T8 | Baja | `fn_user_subcategories.type` redundante (no se valida): decidir si se elimina. |
| T9 | Baja | Fase 4 del blog: CSP, rate limit, cache de Cloudflare con purga. Ver [PLAN-BLOG.md](../plans/PLAN-BLOG.md). |
