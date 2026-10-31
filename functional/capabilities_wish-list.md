# Pendientes e ideas

**Pendientes**: hay que hacerlos o corregirlos. **Ideas**: por evaluar. Lo tecnico va en [technical-pending.md](../technical/technical-pending.md).

## Pendientes

| id | Pendiente | Por que |
|---|---|---|
| F1 | **Plantilla para usuarios nuevos** | Es la fase 5 de auth (onboarding) + D20. Finanzas ya usa tablas `fn_default_*` (categorias, metodos de pago, fuentes, subcategorias) copiadas en `UserAdminService.register`; falta decidir si los demas dominios siguen ese mecanismo o los presets en codigo que planeaba la fase 5. |
| F2 | **Control transversal de gasto de tokens** | La base existe (`ai_usage_events`), pero `/ai/ask` no registra consumo y no hay tope por usuario. Con ~5-10 USD de creditos, es prerequisito para dar permisos de IA a otros. |
| F3 | **Decidir que hacer con el Dashboard** | Es la pagina de aterrizaje y es 100% mock (D15). Igual Study & review y Help. Opciones: conectarla (resumen del dia: checklist, gasto del mes, peso) o quitarla del menu. |
| F4 | Tareas de calendario ocultas en silencio al deshabilitar su categoria (D10) | Perdida de visibilidad de datos sin aviso. |
| F5 | Cambiar zona horaria del usuario (D19) | Decision de producto: preservar hora de pared o instante. |
| F6 | `FilterPanel` con etiquetas que no filtran (D15) | UI que miente. |

## Ideas

| Idea | Nota critica |
|---|---|
| Asistente para estudiar cualquier tema, personalizado | Reusar el patron de English (banco + memoria en base + LLM minimo). Unir con la siguiente: el material natural son mis propios `.md`. |
| Panel para repasar mis `.md` de notas | Ya existe el mock "Study & review". Con repeticion espaciada (FSRS) sobre tarjetas generadas de las notas; las notas viven en `ulm-repository` o en `blg_entities`, decidir cual. |
| MCP server para registrar desde Claude | Es la "fase 2 = agente" del modulo IA expuesta afuera: mismas funciones de service como tools (registrar gasto, peso, tarea). Requiere OAuth en el servidor MCP para conectarlo a claude.ai. Investigar antes de planear. |
| MFA | Con login solo por Google, el MFA ya lo da Google. Solo tiene sentido si se agrega login con contraseña. Prioridad baja. |
| Insights IA por dominio (`<dominio>.ai`) | Roadmap acordado: 5-10 botones deterministas (agregados -> 1 llamada LLM), luego agente con tool calling. El permiso existe, la funcionalidad no. |
| Trabajo futuro de English y blog | Listado en la seccion 14 de [ENGLISH-TEACHER-PLAN.md](../plans/ENGLISH-TEACHER-PLAN.md) y "Despues" en [PLAN-BLOG.md](../plans/PLAN-BLOG.md). No se duplica aqui. |
