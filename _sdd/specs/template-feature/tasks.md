# Tasks Checklist - <Feature Name>

El agente marca cada tarea completada como `[x]`. Ninguna tarea se marca
sin su test correspondiente en verde.

## Fase 1: Datos y Lógica de Negocio
- [ ] Crear/actualizar la entidad o DTO en `Molinos.DataAgro.Entities/`.
- [ ] Configurar el mapeo en el DbContext (solo si aplica, ver AGENTS.md).
- [ ] Implementar la lógica en `<Nombre>Manager.cs` y su interfaz
      `I<Nombre>Manager`.

## Fase 2: Web (MVC + Kendo UI)
- [ ] Crear/actualizar el controlador en `WebDataAgro/Controllers/`.
- [ ] Crear/actualizar la vista `.cshtml` (Kendo Grid / jQuery AJAX si
      corresponde).

## Fase 3: Suite de Pruebas y Verificación
- [ ] Tests NUnit en `Molinos.DataAgro.Test` cubriendo R1, R2, R3...
- [ ] Correr `nunit3-console` — todo en verde.
- [ ] Completar `feature.json` de esta carpeta (`traceability` y
      `qaManualPending`) antes de abrir el PR final.
