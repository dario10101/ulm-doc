# ULM Doc

Documentacion viva de **Unified Life Manager (ULM)**: app personal de gestion de vida (planificacion, finanzas, salud, blog, aprendizaje).

Aqui vive el estado del proyecto, su diseño tecnico y los planes de cada iniciativa. Los repos de codigo no repiten esta informacion: la citan.

## Estructura

```
functional/
  capabilities_done.md          # Funcionalidades terminadas + que tan genericas son
  capabilities_in-progress.md   # Lo incompleto, con link a su plan
  capabilities_wish-list.md     # Pendientes funcionales (F*) e ideas
technical/
  technical-design.md           # Arquitectura, stack, reglas permanentes
  technical-pending.md          # Deuda y pendientes tecnicos (D*, T*)
plans/
  PLAN-<TEMA>.md                # Un plan por iniciativa: decisiones, fases, estado
  done/                         # Planes terminados (historico de referencia)
```

## Por donde empezar

- **Que hace la app hoy**: [capabilities_done.md](./functional/capabilities_done.md)
- **En que se esta trabajando**: [capabilities_in-progress.md](./functional/capabilities_in-progress.md)
- **Como esta construida**: [technical-design.md](./technical/technical-design.md)

## Repos del proyecto

| Repo | Rol |
|---|---|
| [ulm-core](https://github.com/dario10101/ulm-core) | Backend (FastAPI + PostgreSQL) |
| [ulm-web](https://github.com/dario10101/ulm-web) | Frontend (Vue 3 + TypeScript) |
| `ulm-repository` | Guias tecnicas y contenido `.md` |
| `ulm-infra` | Infraestructura (pendiente) |

Los links relativos a `../../ulm-core/...` (`../../../` desde `plans/done/`) funcionan si los repos se clonan como carpetas hermanas.

## Convenciones

- Cada plan indica su estado en la linea 3 (`Estado: ...` o `Ultima actualizacion: ...`).
- Los planes conservan el nombre `PLAN-<TEMA>.md` porque el codigo los cita por nombre: no renombrarlos.
- Un plan terminado se mueve a `plans/done/` (mismo nombre; el codigo lo cita sin ruta). Al moverlo, ajustar sus links relativos y los que apuntan a el.
- Un item vive en un solo lugar. Lo terminado pasa a `capabilities_done.md` y lo resuelto se borra de los pendientes.
- Las reglas de cuando actualizar estan en el `CLAUDE.md` de la raiz del workspace.
