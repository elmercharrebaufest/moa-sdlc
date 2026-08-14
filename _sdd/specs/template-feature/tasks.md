# Tasks Checklist - <Feature Name>

El agente marca cada tarea completada como `[x]`. Ninguna tarea se marca
sin su test correspondiente en verde.

## Fase 1: Datos y Lógica de Negocio
- [ ] Crear/actualizar entidad, DTO, view model o contrato del caso de uso.
- [ ] Definir la capa de acceso a datos o servicio adecuado (`Repository`,
      `Manager`, `Service`, `UseCase`) con mínima dependencia circular.
- [ ] Implementar la lógica y validaciones de negocio con manejo de errores
      explícito y sin mezclar responsabilidades entre capas.

## Fase 2: Web / Frontend
- [ ] Crear/actualizar el endpoint, controlador, página o API relevante.
- [ ] Crear/actualizar la vista, componente o cliente frontend correspondiente:
      MVC, Razor, ASP.NET Core, Angular, jQuery/Kendo o SPA.
- [ ] Validar entrada, salida y errores del usuario sin exponer información
      sensible o datos no autorizados.

## Fase 3: Suite de Pruebas y Verificación
- [ ] Tests automatizados cubriendo R1, R2, R3... (NUnit/XUnit/MSTest y/o
      E2E si corresponde).
- [ ] Ejecutar la validación relevante del stack del proyecto en verde.
- [ ] Revisar calidad de código: complejidad, duplicación, seguridad,
      nulos, manejo de excepciones, rendimiento y trazabilidad.
- [ ] Completar `feature.json` de esta carpeta (`traceability` y
      `qaManualPending`) antes de abrir el PR final.
