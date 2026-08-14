# Agent Harness Operational Model

## Propósito

Este documento define el modelo operativo para usar agentes IA de manera
controlada, repetible y segura dentro del repositorio.

## Modelo general

```
Spec Author / Planner
        ↓
Implementer
        ↓
Tester / Verifier
        ↓
Reviewer
        ↓
Security Reviewer
        ↓
Human Approver
        ↓
Merge / Done
```

Cada etapa produce evidencia. Si falla una etapa, el control vuelve al agente
responsable de esa etapa y no se avanza al siguiente paso.

## Estados del ciclo

- `draft`: no hay especificación cerrada.
- `spec_ready`: la especificación existe y fue revisada.
- `approved`: la feature está lista para empezar la implementación.
- `in_progress`: la tarea está siendo ejecutada.
- `verified`: tests y validación ejecutados con evidencia.
- `reviewed`: calidad y seguridad revisadas.
- `done`: aprobada por humano, mergeada y cerrada.

## Transiciones

```
draft -> spec_ready -> approved -> in_progress -> verified -> reviewed -> done
   ↑                                                        ↓
   └─────────────────────── rollback / fix cycle ───────────┘
```

## Reglas del harness

- Cada agente tiene una responsabilidad mínima y clara.
- Cada etapa requiere evidencia explícita para avanzar.
- No se fusiona trabajo sin trazabilidad al requisito.
- Si un riesgo es alto o crítico, no se sigue con la siguiente etapa.
- La aprobación humana es obligatoria para finalizar la feature.

## Roles y responsabilidades

### 1. Spec Author / Planner

Responsable de:

- definir el problema
- redactar requisitos EARS
- crear `requirements.md`, `design.md`, `tasks.md`
- dejar criterio de aceptación y alcance

No debe implementar código.

### 2. Implementer

Responsable de:

- escribir la solución mínima
- respetar la arquitectura del proyecto
- mantener las capas y patrones correctos
- documentar compatibilidad o riesgo relevante

Debe dejar pruebas o validación equivalente.

### 3. Tester / Verifier

Responsable de:

- correr pruebas relevantes
- validar requerimientos
- documentar fallas, evidencia y riesgos funcionales

Debe mapear requisito → prueba → resultado.

### 4. Reviewer

Responsable de:

- revisión de calidad
- trazabilidad
- regresión
- mantenibilidad y arquitectura

Debe revisar que el cambio cumple la especificación y no introdujo deuda
innecesaria.

### 5. Security Reviewer

Responsable de:

- revisar secretos, auth, XSS, SQLi, CSRF, SSRF
- revisar manejo de errores y exposición de datos
- bloquear cambios con riesgo alto o crítico

Debe dejar evidencia de riesgo o aprobación.

### 6. Human Approver

Responsable de:

- validar PR final
- firmar QA manual y sign-off
- decidir si la feature pasa a `done`

No se completa la feature sin esta aprobación.

## Criterios de salida

Una etapa solo puede pasar a la siguiente si:

- hay evidencia ejecutada
- hay trazabilidad al requisito
- no hay fallo crítico conocido
- la validación fue registrada
- la aprobación humana o del gate pedido fue cumplida

## Definition of Done

Un cambio se considera concluido cuando:

- el requisito quedó cubierto
- los tests pasaron o la validación manual quedó documentada
- no quedaron hallazgos críticos de seguridad o calidad
- `feature.json` refleja trazabilidad
- el PR final fue aprobado por humano
- el estado final quedó en `done`

## Artefactos del harness

- `_sdd/specs/<JIRA-KEY>-<slug>/requirements.md`
- `_sdd/specs/<JIRA-KEY>-<slug>/design.md`
- `_sdd/specs/<JIRA-KEY>-<slug>/tasks.md`
- `_sdd/specs/<JIRA-KEY>-<slug>/feature.json`
- `_sdd/progress/current/<JIRA-KEY>.md`
- `_sdd/progress/history/<JIRA-KEY>.md`
- `.github/instructions/*.instructions.md`
- `.github/prompts/*.prompt.md`
- `.github/pull_request_template.md`

## Conclusión

El harness no debe depender de la improvisación del agente. Debe estar basado en
estados, roles, evidencia y aprobación humana. Esa es la diferencia entre un
agente útil y un agente no controlado.
