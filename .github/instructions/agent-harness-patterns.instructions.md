---
applyTo: "**/*.{md,yml,yaml,json,cs,ts,tsx,js}"
---

# Agent harness patterns

Usar estas reglas para diseñar y operar un sistema de agentes con estados,
stages, validación de evidencia y gates humanos.

## Objetivo

Un harness de agentes debe ser determinista, observables y seguro. No sirve un
agente que “parece funcionar”; debe existir un flujo claro con evidencia y un
estado explícito.

## Principios del harness

- Cada tarea debe tener un objetivo claro y un criterio de aceptación.
- Cada agente debe tener un rol concreto y una responsabilidad limitada.
- Cada etapa debe producir evidencia: test output, diff, logs, review notes,
  decisiones y resultados.
- Cada transición entre estados debe requerir validación explícita.
- Nunca se debe avanzar de fase sin cumplir el criterio mínimo de salida.
- Los bloqueos deben quedar documentados y no asumirse como resueltos.

## Estados recomendados

- `draft`: requisito o ticket sin especificación cerrada.
- `spec_ready`: especificación creada y aprobada por la parte de negocio.
- `approved`: la feature está lista para implementación.
- `in_progress`: implementación en curso.
- `verified`: pruebas y validación completadas.
- `reviewed`: calidad y seguridad revisadas.
- `done`: aprobado, mergeado y finalizado.

## Regla de loop

- Si falla una etapa, el control vuelve al agente responsable.
- Si se detecta un riesgo alto, no se debe continuar sin intervención humana.
- La evidencia es obligatoria para avanzar entre estados.

## Reglas por rol

### Spec Author / Planner

- Define requisitos en EARS.
- Crea `requirements.md`, `design.md` y `tasks.md`.
- No debe tocar código de implementación.
- Debe dejar claros los criterios de aceptación.

### Implementer

- Trabaja solo sobre la tarea aprobada.
- Mantiene el cambio mínimo y razonable.
- No oculta warnings ni desactiva validaciones.
- Debe dejar evidencia de que se cumple el requisito.

### Tester

- Ejecuta pruebas relevantes del stack.
- Mapea requisitos a validación.
- Registra fallos abiertos, no avances a ciegas.

### Reviewer

- Revisa calidad, arquitectura, trazabilidad y regresión.
- Confirma que el cambio encaja con el diseño y no rompe compatibilidad.

### Security Reviewer

- Revisa secretos, auth, autorización, XSS, SQLi, CSRF, SSRF, CORS y datos
  sensibles.
- Si hay hallazgo alto o crítico, bloquea el paso siguiente.

### Human Approver

- Aprueba PRs y sign-off manual.
- Decide si la feature pasa a `done`.
- No se debe considerar terminado sin aprobación humana.

## Acceptance criteria model

Cada tarea debe responder:

1. ¿Qué se quería lograr?
2. ¿Qué evidencia prueba que se logró?
3. ¿Qué condiciones bloquean la aprobación?
4. ¿Qué criterio define que el trabajo está hecho?

## Output esperado del harness

- un estado claro
- evidencia verificable
- traceability del requerimiento a la validación
- activos de revisión (quality/security)
- decision log con aprobación o bloqueo

Un harness robusto no depende de la “intuición” de la IA; depende de estados,
reglas, artefactos y gate humanos.
