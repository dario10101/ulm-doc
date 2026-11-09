# ULM - Plan de auth y multiusuario

Estado: fases 0-5 hechas. Terminado. El deploy sigue en [PLAN-DEPLOY.md](../PLAN-DEPLOY.md).
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
| 5 | **Onboarding**: al registrarse, el usuario recibe sus parametros iniciales sin preguntarle (copia de las plantillas `fn_default_*` en `UserAdminService.register`, misma transaccion que el alta). Los demas dominios no necesitan semilla: English y blog crean su registro al primer uso; checklist, calendario, peso y comidas los arma el usuario. Se descarto preguntar por dominio (`user_domain_setup`). Resuelve D20 y F1 | ✅ |
