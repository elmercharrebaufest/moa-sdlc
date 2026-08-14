# Spec-Driven Development (SDD) - Manual Metodológico

Este repositorio usa Spec-Driven Development (SDD). Principio central:
**el código es un detalle de implementación; la especificación
en Markdown es la guia del desarrollo del codigo.**

## El Ciclo PEV (Plan-Execute-Verify)

Toda nueva funcionalidad se estructura en tres fases discretas antes de
tocar código:

```
Fase 1: SPECIFY & PLAN    Fase 2: EXECUTE           Fase 3: VERIFY
[ spec-author (IA) ]  →   [ implementer (IA) ]  →   [ reviewer (IA) ]
Redacta requirements,     Aplica los cambios en     Corre tests, verifica
design y tasks.           código, tarea por tarea.  trazabilidad R<n>.
Espera aprobación                                   Marca pendientes de
humana vía PR.                                      QA manual.
```

## Flujo de branches y PRs

1 ticket de Jira = 1 branch = 1 carpeta de spec = 2 PRs (spec + final),
ambos contra `Dev`.

```
feature/<JIRA-KEY>-<slug>
   │
   PR #1 (spec)  → Dev   [aprobación humana de la interpretación del ticket]
   │
   (misma branch, sigue viva)
   │
   PR #2 (final) → Dev   [código + tests + QA manual + CI en verde]
```

## Directrices para el Spec Author (IA)

- Formato EARS para requisitos:
  `[Disparador/Precondición] → El sistema DEBE → [Acción/Resultado]`
  Ejemplo: "Cuando el frontend envíe un ID nulo, el backend debe
  responder con HTTP 400 Bad Request."
- Cada requisito numerado (R1, R2...) para que el reviewer pueda mapear
  cada uno a un test exacto.

## Puerta de Aprobación Humana

- No hay archivo de estado global: el estado vive en
  `_sdd/progress/current/<JIRA-KEY>.md`, campo "Estado Actual".
- Al terminar la Fase 1, el agente deja ese campo en `spec_ready` y abre
  el PR #1. El agente se detiene ahí, no toca código.
- El humano revisa el PR, comenta o aprueba. Al aprobar y mergear, edita
  a mano el campo "Estado Actual" a `approved` en
  `_sdd/progress/current/<JIRA-KEY>.md`.
- Solo con "Estado Actual: approved" el agente `implementer` puede tocar
  código. Antes de escribir cualquier línea de código debe releer ese
  archivo para confirmarlo.

## Manual de Uso

Guía rápida para el humano que arranca o continúa una feature con SDD.

### 1. Arrancar una feature nueva
1. Invocar el prompt `/sdd-workflow` indicando el `<JIRA-KEY>` real (traído
   de Jira, nunca inventado).
2. La IA (Spec Author) crea la branch `feature/<JIRA-KEY>-<slug>`, redacta
   `_sdd/specs/<JIRA-KEY>-<slug>/{requirements,design,tasks,feature.json}`
   y crea `_sdd/progress/current/<JIRA-KEY>.md` con
   "Estado Actual: spec_ready".
3. La IA abre el PR #1 (spec) y se detiene ahí.

### 2. Aprobar la especificación
1. Revisar el PR #1: ¿los requisitos (R1, R2...) reflejan bien el ticket?
2. Comentar o pedir ajustes si hace falta. Al conformar, mergear el PR #1
   a `Dev`.
3. Editar a mano `_sdd/progress/current/<JIRA-KEY>.md` y poner
   "Estado Actual: approved".

### 3. Continuar con la implementación
1. Invocar `/sdd-workflow` de nuevo con el mismo `<JIRA-KEY>`.
2. La IA (Implementer) confirma que el estado es `approved`, ejecuta las
   tareas de `tasks.md` una por una, y solo marca `[x]` con test en verde.
3. Cuando toda la suite (`dotnet build` / `nunit3-console`) está en verde,
   la IA (Reviewer) completa `_sdd/specs/<JIRA-KEY>-<slug>/feature.json`
   (trazabilidad R<n>→test y pendientes de QA manual — el archivo de
   `progress/current` es gitignored y no llega al PR), agrega el resumen
   narrativo en `_sdd/progress/history/<JIRA-KEY>.md`, y deja
   "Estado Actual: verified".

### 4. Cerrar la feature
1. Revisar el PR #2 (final): código, tests, `feature.json` (trazabilidad
   y QA manual) y el resumen en `progress/history`.
2. Confirmar el sign-off de los requisitos tipo "manual" (campo
   `signOff` en `feature.json`).
3. Mergear el PR #2 a `Dev`. El humano actualiza `estado: "done"` en
   `feature.json` y borra el archivo de `progress/current` (su historia
   ya quedó en `progress/history` y `feature.json`).

### 5. Qué es efímero vs. permanente
- **Efímero (gitignored, nunca se commitea):** `_sdd/progress/current/<JIRA-KEY>.md`.
- **Permanente (se commitea):** `_sdd/specs/<JIRA-KEY>-<slug>/*`
  (incluye `feature.json` con la trazabilidad y QA manual),
  `_sdd/progress/history/<JIRA-KEY>.md` (resumen narrativo).

### Ejemplo mínimo end-to-end
```
/sdd-workflow MOA-2000
→ crea feature/MOA-2000-ajuste-cupos
→ crea _sdd/specs/MOA-2000-ajuste-cupos/{requirements,design,tasks,feature.json}
→ crea _sdd/progress/current/MOA-2000.md (Estado Actual: spec_ready)
→ abre PR #1, se detiene

[humano aprueba y mergea PR #1, edita Estado Actual: approved]

/sdd-workflow MOA-2000
→ implementa tasks.md, marca [x] con tests verdes
→ completa feature.json (trazabilidad + qaManualPending)
→ agrega resumen en _sdd/progress/history/MOA-2000.md
→ deja Estado Actual: verified, abre PR #2

[humano revisa, aprueba QA manual y mergea PR #2]
```
