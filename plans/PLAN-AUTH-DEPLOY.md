# ULM - Plan de auth, multiusuario y deploy

Estado: fases 0-4 hechas y commiteadas; **siguiente: fase 5 (onboarding)**; deploy decidido, no ejecutado.
Objetivo: abrir la app a 5-10 usuarios invitados con datos aislados.

## Decisiones (2026-10-03)

- **Auth**: solo Google (OIDC + PKCE, `joserfc`/`httpx`/`itsdangerous`; no authlib, su integracion httpx esta deprecada). Contraseña a futuro = otra fila en `user_identities`. Google Cloud en modo Testing con lista de test users.
- **Acceso por invitacion**: `REGISTRATION_MODE=invite_only` (`open` ya soportado). Zona horaria tomada del navegador en el primer login.
- **Admin**: `ADMIN_EMAILS` en config, no fila en base: todos los permisos, entra sin invitacion, no editable.
- **Permisos**: por dominio base (`weight`, `meals`, `finances`, `planning` = checklist + calendario) y avanzado `<dominio>.ai`. Luego se sumaron `blog`, `english`, `ai`.
- Fuera de alcance: borrar invitaciones o editar email desde la UI, log de auditoria.

## Fases

| Fase | Contenido | Estado |
|---|---|---|
| 0 | Aislamiento por usuario en todos los repositories | ✅ |
| 1 | `user_identities`, `user_sessions`, login Google, `scripts/manage_users.py` | ✅ |
| 2 | Front: `useAuth`, guard, 401 global, logout, Settings con `/me` | ✅ |
| 3 | Permisos por dominio (back + front) | ✅ |
| 4 | `/admin/users` + Settings -> System -> Users | ✅ |
| **5** | **Onboarding**: por dominio, preguntar si crear parametrizacion base; decision en `user_domain_setup`. Resuelve D20 y F1 | ⏳ Pendiente. Decidir antes: presets en codigo vs tablas plantilla (finanzas ya usa tablas `fn_default_*` copiadas en el registro) |
| 6 | Deploy | ⏳ Bloqueado por D17 (Dockerfile) |

## Deploy recomendado

Cloudflare Pages (web), Cloud Run us-east1 con Workload Identity Federation (core), Neon (Postgres), dominio en Cloudflare. `main` = prod.

Orden al migrar datos reales: crear el usuario primero (recibe sus categorias) y luego correr los scripts de import (tienen guarda anti-duplicados).
