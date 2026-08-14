---
agent: agent
description: 'Guía al agente a través del ciclo PEV de Spec-Driven Development (Spec Author → Implementer → Reviewer) para un ticket de Jira.'
---

# SDD Workflow

Ejecutá el ciclo PEV descrito en `_sdd/docs/specs.md` para el ticket
`${input:jiraKey:JIRA-KEY del ticket, ej. MOA-2000}`.

Primero, buscá `_sdd/progress/current/${input:jiraKey}.md`:
- Si no existe, el estado es `draft` → seguí la sección **Fase 1**.
- Si existe, leé el campo "Estado Actual" y saltá a la sección que
  corresponda.

No asumas ningún estado sin leer el archivo. No avances de fase sin la
aprobación humana explícita que exige cada sección.

## Fase 1 — Spec Author (Estado: draft | spec_ready)

Aplica cuando el estado es `draft` o el archivo de progreso no existe.

1. Traé el ticket real de Jira con el MCP de Atlassian usando
   `${input:jiraKey}`. Nunca inventes el contenido del ticket.
2. Creá la branch `feature/${input:jiraKey}-<slug>` (slug derivado del
   título del ticket).
3. A partir de `_sdd/specs/template-feature/{requirements,design,tasks,feature.json}`,
   generá `_sdd/specs/${input:jiraKey}-<slug>/{requirements,design,tasks,feature.json}`:
   - `requirements.md`: requisitos en formato EARS, numerados R1, R2...
     con tipo de verificación (`unit | manual`).
   - `design.md`: impacto en backend/frontend, DTOs, endpoints.
   - `tasks.md`: checklist de tareas agrupadas por fase técnica.
   - `feature.json`: `jiraKey`, `slug` y `estado: "draft"`, con
     `traceability` y `qaManualPending` vacíos (se completan en Fase 3).
4. Creá `_sdd/progress/current/${input:jiraKey}.md` (a partir de su
   template) con "Estado Actual: spec_ready".
5. Abrí el PR #1 (spec) contra `Dev`. **Detenete acá** — no toques código,
   no continúes a Fase 2 sin que el humano apruebe y mergee el PR #1 y
   actualice manualmente el "Estado Actual" a `approved`.

## Fase 2 — Implementer (Estado: approved | in_progress)

Aplica cuando `_sdd/progress/current/${input:jiraKey}.md` dice `approved`
o `in_progress`.

1. Confirmá explícitamente que el estado es `approved` o `in_progress`
   antes de tocar cualquier archivo de código. Si dice `draft` o
   `spec_ready`, no continúes: volvé a la Fase 1.
2. Actualizá el estado a `in_progress` si todavía dice `approved`.
3. Recorré `tasks.md` de la carpeta de spec, tarea por tarea:
   - Implementá el cambio mínimo necesario para esa tarea.
   - Corré los tests NUnit relevantes (`nunit3-console` o Test Explorer).
   - Marcá `[x]` **solo** si el test correspondiente está en verde.
   - Actualizá la sección "Verified Facts" y "Open Failures" en
     `_sdd/progress/current/${input:jiraKey}.md` con lo comprobado.
4. Respetá los límites de `AGENTS.md`: pedí aprobación humana antes de
   instalar paquetes, modificar entidades con migraciones EF Core, o
   cambiar firmas de controladores existentes. Nunca hagas `git push` ni
   abras PRs vos mismo.

## Fase 3 — Reviewer (Estado: in_progress con tests verdes → verified)

Aplica cuando todas las tareas de `tasks.md` están `[x]` y la suite de
tests corre en verde.

1. **Checklist de validación** (repasalo antes de avanzar, no lo saltees):
   - [ ] `dotnet build` y `nunit3-console` (tests) en verde.
   - [ ] Toda tarea `[x]` en `tasks.md` tiene un test identificable que la
     cubre.
   - [ ] Todo requisito R<n> tipo `unit` de `requirements.md` tiene al
     menos un test mapeado en `feature.json`.
   - [ ] Los requisitos tipo `manual` están listados en
     `qaManualPending` de `feature.json`, con `signOff: null`. **No
     necesitan test automatizado para avanzar** a `verified` y abrir el
     PR #2 — el sign-off humano reemplaza al test, pero la feature no
     pasa a `done` hasta que el humano lo complete explícitamente (nunca
     lo completes vos).
2. Completá `_sdd/specs/${input:jiraKey}-<slug>/feature.json`:
   - `traceability`: un objeto `{ requisito, test, estado }` por cada
     R<n> de `requirements.md`.
   - `qaManualPending`: un objeto `{ requisito, descripcion, signOff:
     null }` por cada requisito tipo `manual` (array vacío si no hay).
   Este JSON reemplaza a `progress/current/${input:jiraKey}.md` como
   fuente de verdad estructurada, ya que `progress/current` es gitignored
   y no llega al PR.
3. Agregá una línea corta en `_sdd/progress/history/${input:jiraKey}.md`
   (a partir de su template) con el resumen narrativo y un link al
   `feature.json` para el detalle de trazabilidad/QA manual.
4. Dejá "Estado Actual: verified" en `progress/current/${input:jiraKey}.md`
   y en el campo `estado` de `feature.json`.
5. Abrí (o actualizá) el PR #2 (final) contra `Dev`. **Detenete acá** — el
   humano revisa QA manual en `feature.json`, aprueba y mergea. Recién ahí
   la feature pasa a `done` (actualizá `estado` en `feature.json` y
   borrá/archivá `progress/current/${input:jiraKey}.md`).
