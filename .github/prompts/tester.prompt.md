---
agent: 'agent'
description: 'Valida una feature con pruebas, evidencia de ejecución y verificación del comportamiento esperado.'
---

# Tester workflow

Validá la feature o cambio solicitado según la especificación y la evidencia de la
implementación.

## Objetivo

Comprobar que el cambio cumple los requisitos y que no presenta regresión.

## Reglas

- Leer primero `requirements.md` y `tasks.md`.
- Mapear cada requisito R<n> a su validación automatizada o manual.
- Ejecutar los tests relevantes del stack del proyecto.
- Si hay cambio de lógica de negocio, priorizar pruebas unitarias.
- Si hay cambio de UI, validar interacción clave y documentar manual QA si no hay
  test automatizado.
- Para .NET legacy, usar `nunit3-console` o el entorno de test existente;
  para .NET moderno, usar `dotnet test` cuando corresponda.
- Registrar evidencia: comando ejecutado, resultado, y cobertura funcional.

## Checklist

- [ ] Cada requisito relevante tiene evidencia de verificación.
- [ ] Los tests pasan o se documenta la validación manual.
- [ ] Hay casos de éxito y de error.
- [ ] No hubo regresión en lógica relacionada.
- [ ] Los resultados quedan registrados en `feature.json` o en el estado de la
      feature.

## Output esperado

Entregá:

1. resultados de pruebas ejecutadas
2. requisitos validados
3. fallos abiertos o pendientes
4. evidencia de validación manual si aplica
5. recomendación de continuar, corregir o bloquear
