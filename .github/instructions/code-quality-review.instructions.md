---
applyTo: "**/*.{cs,cshtml,razor,ts,tsx,js,html,csproj,sln,json,md}"
---

# Code quality review instructions for Copilot

Usar esta guía como revisión previa a aprobación final del código, antes de que
pase por análisis estático o SonarQube.

## Checklist de calidad

- Verificar que la lógica es correcta y no introduce regresiones.
- Confirmar que cada cambio nuevo tiene tests o evidencia de validación.
- Buscar errores de nullability, desbordes, conversiones implícitas, `async`/
  `await` incorrectos y manejo de excepciones.
- Revisar seguridad: inyección SQL, XSS, CSRF, endpoint público sin autorización,
  logs con secretos, validación insuficiente y uso indebido de `eval`/`innerHTML`.
- Revisar rendimiento: N+1, loops grandes, caché inexistente, I/O innecesario,
  consultas pesadas y serialización repetida.
- Revisar mantenibilidad: complejidad ciclomatica, duplicación, nombres ambiguos,
  acoplamiento alto, código muerto y funciones demasiado largas.
- Confirmar que la solución sigue la arquitectura del proyecto: capas,
  separación, inyección de dependencias y patrones existentes.

## SonarQube-oriented expectations

Tratar estas condiciones como pre-gate para SonarQube:

- No dejar code smells sin justificación.
- Evitar duplicación acoplada a lógica de negocio.
- No introducir warnings de seguridad, vulnerabilidades o bugs confirmados.
- No mezclar responsabilidades de UI, servicio y acceso a datos.
- No dejar logs de depuración en producción ni mensajes con datos sensibles.
- No ocultar errores con `catch` vacío o `throw new Exception` genérico.

## Revisión final recomendada

Antes de aprobar un cambio, responder:

1. ¿La solución resuelve el problema sin introducir regresión?
2. ¿Está cubierto por tests o validación manual documentada?
3. ¿El código es legible, seguro, testeable y soportable?
4. ¿Hay evidencia de que el cambio cumple requisitos de negocio y QA?
5. ¿Hay deuda técnica no justificada o casos peligrosos pendientes?

Si hay dudas, documentar el riesgo y pedir revisión humana.
