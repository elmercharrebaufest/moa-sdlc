---
agent: 'agent'
description: 'Implementa la solución mínima y correcta para una feature o corrección de bug, respetando arquitectura, tests y validaciones del proyecto.'
---

# Implementer workflow

Trabajá sobre la feature o tarea especificada en `_sdd/specs/<JIRA-KEY>-<slug>/`.

## Objetivo

Implementar la solución mínima, clara y testeable, sin mezclar responsabilidades ni
romper la arquitectura del proyecto.

## Reglas

- Leer primero `requirements.md`, `design.md` y `tasks.md`.
- Confirmar que el estado de la feature está en `approved` o `in_progress`.
- No tocar código ajeno a la tarea si no es necesario.
- Mantener compatibilidad con el stack actual: .NET Framework legacy si aplica,
  .NET moderno si aplica, MVC/Razor/Angular segun el proyecto.
- Usar Clean Code y Clean Architecture cuando aporten claridad y mantenibilidad,
  pero sin forzar patrones innecesarios.
- No ocultar errores, no silenciar warnings ni desactivar validaciones.
- Documentar decisiones importantes si afectan diseño, compatibilidad o riesgo.

## Checklist antes de cerrar la tarea

- [ ] La solución cumple el requisito R<n> asociado.
- [ ] La lógica queda en la capa correcta.
- [ ] No se mezclan responsabilidades ni se duplican patrones innecesarios.
- [ ] Hay tests o validación equivalente para la lógica nueva o modificada.
- [ ] Se respetan las reglas de seguridad, calidad y tooling del repositorio.

## Output esperado

Entregá:

1. resumen corto del cambio
2. archivos principales modificados
3. validaciones ejecutadas o pendientes
4. riesgo de compatibilidad o arquitectura
5. estado de tests y evidencia disponible
