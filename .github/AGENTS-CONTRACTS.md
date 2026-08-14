# Agent Contracts and Runtime Governance

## 1. Propósito

Este documento define el contrato operativo del harness de agentes para que cada
rol tenga entrada clara, salida esperada, reglas de ejecución y criterios de
éxito/fallo.

## 2. Modelo general

Cada agente recibe una solicitud con un payload estructurado y debe devolver una
respuesta con resultado, evidencia y estado. El sistema no puede avanzar sin que
el resultado cumpla los criterios de aceptación.

## 3. Contracto base de entrada

```json
{
  "featureId": "MOA-2000",
  "jiraKey": "MOA-2000",
  "slug": "ajuste-cupos",
  "role": "implementer",
  "state": "approved",
  "ticket": {
    "title": "Título real del ticket",
    "description": "Descripción real del ticket",
    "url": "https://dev.azure.com/..."
  },
  "spec": {
    "requirementsPath": "_sdd/specs/MOA-2000-ajuste-cupos/requirements.md",
    "designPath": "_sdd/specs/MOA-2000-ajuste-cupos/design.md",
    "tasksPath": "_sdd/specs/MOA-2000-ajuste-cupos/tasks.md",
    "featureJsonPath": "_sdd/specs/MOA-2000-ajuste-cupos/feature.json"
  },
  "context": {
    "branch": "feature/MOA-2000-ajuste-cupos",
    "environment": "dev",
    "stack": "dotnet-framework|dotnet|angular|mixed"
  },
  "constraints": {
    "mustNotChange": [
      "entities/legacy/*.cs",
      "public apis without approval"
    ],
    "mustAskBefore": [
      "install new package",
      "change EF model",
      "change public API contract"
    ]
  }
}
```

## 4. Contracto base de salida

```json
{
  "agent": "implementer",
  "featureId": "MOA-2000",
  "status": "success|failed|blocked",
  "stateTransition": {
    "from": "approved",
    "to": "in_progress"
  },
  "summary": "Resumen de lo hecho",
  "artifacts": [
    "archivo1.cs",
    "archivo2.md",
    "feature.json"
  ],
  "evidence": {
    "testsExecuted": [
      "dotnet test --filter ...",
      "nunit3-console ..."
    ],
    "results": [
      "pass",
      "pass"
    ],
    "manualChecks": [
      "validación manual de UI"
    ]
  },
  "risks": [
    "compatibilidad con endpoint legacy"
  ],
  "blockingIssues": [],
  "nextAction": "tester|reviewer|human-approver"
}
```

## 5. Criterios de éxito y fallo por agente

### 5.1 Spec Author

Entrada esperada:

- Jira ticket real
- objetivo del cambio
- contexto del repositorio

Debe devolver:

- `requirements.md`
- `design.md`
- `tasks.md`
- `feature.json`
- `progress/current/<JIRA>.md`

Criterio de éxito:

- requisitos escritos con EARS
- cada requisito numerado
- hay trazabilidad posible a tests
- no hay ambigüedad funcional

Criterio de fallo:

- requisitos vagos
- falta de diseño técnico
- no hay tareas o no hay criterio de aceptación

### 5.2 Implementer

Entrada esperada:

- feature aprobada
- `requirements.md`, `design.md`, `tasks.md`

Debe devolver:

- código modificado
- pruebas o evidencia equivalente
- estado actualizado de ejecución

Criterio de éxito:

- cumple la tarea definida
- no rompe arquitectura ni compatibilidad esperada
- el cambio está en la capa correcta
- hay evidencia mínima de verificación

Criterio de fallo:

- rompe el stack o compatibilidad
- mezcla responsabilidades
- no prueba ni documenta la ejecución

### 5.3 Tester

Entrada esperada:

- implementación o cambio listo
- requisitos y tareas

Debe devolver:

- resultados de pruebas
- fallos abiertos o bloqueos
- evidencia para `feature.json`

Criterio de éxito:

- cada requisito relevante tiene validación
- casos de éxito y error cubiertos
- evidencia disponible y reproducible

Criterio de fallo:

- no hay validación documental
- pruebas omitidas sin justificación
- fallos no reportados

### 5.4 Reviewer

Entrada esperada:

- código y evidencia
- feature.json

Debe devolver:

- estado de calidad
- hallazgos y recomendaciones
- aprobación o bloqueo

Criterio de éxito:

- no hay regresión relevante
- la arquitectura es coherente
- la trazabilidad está presente

Criterio de fallo:

- código con deuda sin justificar
- duplicación excesiva
- riesgo de regresión no documentado

### 5.5 Security Reviewer

Entrada esperada:

- diff o cambio funcional
- contexto de runtime y despliegue

Debe devolver:

- nivel de riesgo
- hallazgos concretos
- decisión: aprobar / revisar / rechazar

Criterio de éxito:

- no hay secretos ni exposición de datos
- auth/autorización correctos
- sin vectores de XSS/SQLi/CSRF/SSRF relevantes

Criterio de fallo:

- hallazgos de seguridad alto o crítico
- exposición de credenciales, tokens o datos sensibles

### 5.6 Human Approver

Entrada esperada:

- PR final con evidencia completa
- QA manual y cambios riesgosos

Debe devolver:

- aprobación
- rechazo o bloqueo
- sign-off manual en `feature.json`

Criterio de éxito:

- QA manual firmada
- feature finalizada y mergeable

Criterio de fallo:

- PR abierto sin evidencia suficiente
- no hay sign-off de requisitos manuales

## 6. Allowlist de tools por rol

### 6.1 Spec Author

Permitido:

- leer specs base
- leer tickets y contexto del repositorio
- crear/editar Markdown de specs
- crear `feature.json`
- no modificar código de negocio

Prohibido:

- modificar código fuente funcional
- ejecutar build/test de la feature como cierre
- hacer commits o PR en nombre del repo

### 6.2 Implementer

Permitido:

- leer archivos del proyecto relacionados
- editar código fuente
- crear tests básicos asociados
- ejecutar build/tests relevantes
- actualizar `feature.json` si aplica

Prohibido:

- cambiar contratos públicos sin aprobación explícita
- instalar paquetes sin aprobación humana cuando se requiera
- modificar entidades/DB de forma no aprobada
- silenciar validaciones del proyecto

### 6.3 Tester

Permitido:

- ejecutar pruebas del stack
- revisar diffs y requisitos
- documentar evidencia
- marcar este estado en metadata de feature

Prohibido:

- cambiar código para que el test pase sin fundamentación
- ocultar fallos
- modificar requisitos para que encajen en una prueba

### 6.4 Reviewer

Permitido:

- leer código y diffs
- evaluar trazabilidad y calidad
- comentar en PR o documento de feature
- reclasificar riesgos

Prohibido:

- aprobar sin evidencia
- ignorar hallazgos de regresión o complejidad

### 6.5 Security Reviewer

Permitido:

- revisar código, configs, secrets, endpoints y pipelines
- evaluar políticas de auth/authorization
- revisar PR diff y configuración

Prohibido:

- aprobar cambios con hallazgos críticos no corregidos
- ocultar riesgo como “no aplica” sin justificar

### 6.6 Human Approver

Permitido:

- revisar PR
- decidir merge
- agregar sign-off manual
- cerrar `done`

Prohibido:

- dar sign-off sin evidencia
- ignorar requisitos manuales pendientes

## 7. Schema formal del estado

```json
{
  "featureId": "MOA-2000",
  "currentState": "verified",
  "allowedTransitions": [
    { "from": "draft", "to": "spec_ready" },
    { "from": "spec_ready", "to": "approved" },
    { "from": "approved", "to": "in_progress" },
    { "from": "in_progress", "to": "verified" },
    { "from": "verified", "to": "reviewed" },
    { "from": "reviewed", "to": "done" },
    { "from": "in_progress", "to": "draft" }
  ],
  "transitionRules": {
    "spec_ready": "require requirements and design",
    "approved": "require human approval and PR #1 merged",
    "verified": "require tests and evidence",
    "reviewed": "require quality and security checks",
    "done": "require human sign-off and merge"
  },
  "blockers": [
    "securityCritical",
    "missingTestEvidence",
    "manualQaUnsigned",
    "contractChangeWithoutApproval"
  ]
}
```

## 8. Runtime logs / audit trail

Cada ejecución debe dejar registro de:

- `timestamp`
- `featureId`
- `agentRole`
- `actor` (agente o humano)
- `inputSummary`
- `actionTaken`
- `outputSummary`
- `verificationEvidence`
- `status`
- `errorsOrWarnings`
- `nextStep`

Ejemplo:

```json
{
  "timestamp": "2026-08-14T16:00:00Z",
  "featureId": "MOA-2000",
  "agentRole": "tester",
  "actor": "copilot-agent",
  "inputSummary": "Validación de requisito R2",
  "actionTaken": "dotnet test --filter R2",
  "outputSummary": "Test passed",
  "verificationEvidence": [
    "build output",
    "test report",
    "manual QA note"
  ],
  "status": "success",
  "errorsOrWarnings": [],
  "nextStep": "reviewer"
}
```

## 9. Retry policy and rollback rules

### 9.1 Retry policy

- Un test fallido no debe reintentarse indefinidamente.
- El agente debe hacer un máximo de 1-2 reintentos si el error es claramente
  transitorio y documentalmente reproducible.
- Si el fallo persiste, se bloquea la transición y se retorna a la etapa
  responsable.

### 9.2 Rollback rules

- Si un agente detecta un problema de diseño, no se debe avanzar en la
  implementación.
- Si un cambio rompe compatibilidad, se debe revertir o aislar el cambio antes
  de continuar.
- Si hay riesgo de seguridad, se bloquea y se requiere corrección previa.
- Si hay regresión en un requisito ya aprobado, se vuelve al `implementer` o al
  `spec-author` según la causa raíz.

## 10. Definition of Done machine-readable

```json
{
  "definitionOfDone": {
    "requirementsCovered": true,
    "testsPassed": true,
    "manualQaSignedOff": true,
    "securityReviewPassed": true,
    "qualityReviewPassed": true,
    "featureJsonUpdated": true,
    "historySummaryAdded": true,
    "approvalByHuman": true,
    "branchMerged": true,
    "status": "done"
  }
}
```

## 11. Resultado esperado

El sistema de agentes no debe depender de la intuición ni de una ejecución
“manual improvisada”. Debe ser gobernado por:

- roles definidos
- contratos de entrada/salida
- criterios de éxito/fallo claros
- allowlist de tools por rol
- estados y transiciones explícitas
- logging y audit trail
- reglas de retry/rollback
- definition of done estructurada

Esto permite que el proceso sea reproducible, auditable y robusto para equipos
que trabajen con Copilot, GitHub/GitLab/Azure DevOps y pipelines de calidad.
