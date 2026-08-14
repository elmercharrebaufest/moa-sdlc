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

## Loop del ciclo y roles de agentes

El flujo no es lineal “una sola IA hace todo”. Debe verse como un ciclo
repetitivo con roles separados y retroalimentación continua:

```
Spec Author → Implementer → Tester/Verifier → Reviewer → Security Review
      ↑                                                     ↓
      └─────────────── Human approval / PR gate ─────────────┘
```

### Roles recomendados

El equipo puede usar un conjunto de prompts especializados para hacer explícitos
los roles dentro del proceso:

- `spec-author` / planner: define requisitos, diseño y plan del trabajo.
- `implementer`: realiza la implementación mínima y concreta.
- `tester`: valida con pruebas técnicas y documenta evidencia.
- `reviewer`: revisa calidad, arquitectura y trazabilidad.
- `security-reviewer`: revisa seguridad, secretos, auth, XSS, SQLi, etc.
- `human-approver`: aprueba el PR y el sign-off manual.

Los prompts que lo materializan en este repositorio son:

- `.github/prompts/sdd-workflow.prompt.md`
- `.github/prompts/implementer.prompt.md`
- `.github/prompts/tester.prompt.md`
- `.github/prompts/reviewer.prompt.md`
- `.github/prompts/security-reviewer.prompt.md`

### Rollback y reasignación del ciclo

Cuando una etapa falla, no se “salta” ni se fuerza aprobación. El ciclo vuelve al
agente propietario del punto que falló:

- requisitos ambiguos o incompletos → `spec-author`
- error en la solución → `implementer`
- pruebas fallidas o evidencia insuficiente → `tester`
- diseño, calidad o trazabilidad pobre → `reviewer`
- riesgo de seguridad → `security-reviewer`
- falta de sign-off humano → `human-approver`

El proceso solo se cierra cuando existe evidencia, aprobación y trazabilidad.

## Workflow por defecto de una feature

```
Ticket Jira
   ↓
Spec Author / Planner
   ↓
requirements.md + design.md + tasks.md + feature.json
   ↓
Human approval (PR #1)
   ↓
Implementer
   ↓
Code + change set
   ↓
Tester
   ↓
Tests + evidence + validation
   ↓
Reviewer
   ↓
Quality review + traceability
   ↓
Security Reviewer
   ↓
Security risk check
   ↓
Human approver
   ↓
PR final + sign-off + merge
```

### Reglas del cierre de la feature

- La feature no pasa a `done` sin aprobación humana.
- Los requisitos manuales quedan en `feature.json` hasta que haya sign-off.
- La trazabilidad R<n> → test debe existir antes del PR final.
- El `progress/current` es efímero; el artefacto permanente es el spec + JSON +
  historial final.


- **Spec Author / Planner**: redacta requisitos, diseño y tareas. Define el
  problema y la intención antes de tocar código.
- **Implementer / Builder**: ejecuta las tareas de `tasks.md`, escribe código
  mínimo para cumplir el requisito y deja la feature en un estado funcional.
- **Tester / Verification Agent**: ejecuta pruebas unitarias, integración o
  validación manual; valida que cada tarea tenga evidencia de verificación.
- **Reviewer / Quality Gate**: valida no regresión, calidad, complejidad,
  maintainability, trazabilidad y cumplimiento del plan.
- **Security Reviewer**: revisa riesgos de seguridad, autenticación,
  autorización, validación de entrada, secretos y exposición de datos.
- **Human approver / Product Owner / Dev**: aprueba PRs, firma QA manual,
  decide si el cambio pasa a `done`.

### Regla del loop

- Cada ciclo debe cerrar con evidencia: requisitos → implementación → pruebas →
  revisión → aprobación.
- Si falla una etapa, el loop vuelve al agente correspondiente:
  - si falla el diseño → vuelve al Spec Author
  - si falla la lógica → vuelve al Implementer
  - si falla la validación → vuelve al Tester
  - si falla calidad/seguridad → vuelve al Reviewer / Security Reviewer
- El ciclo se repite hasta que el cambio tenga prueba, trazabilidad y
  sign-off humano.

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

## Tipos de artefactos del flujo SDD y Copilot

Este repo usa varios tipos de archivos con funciones distintas. Entenderlos ayuda
porque cada uno cumple un propósito diferente:

- **`AGENTS.md`**: define los límites de autonomía, prohibiciones y reglas del
  proyecto para agentes IA. Es la política operativa: qué puede hacer, qué debe
  pedir antes, qué no puede tocar. Es obligatorio leerlo antes de ejecutar
  cambios grandes.
- **`.github/copilot-instructions.md`**: instrucciones generales del repositorio
  para Copilot. Es el “manual base” para el proyecto: arquitectura, tests,
  estándares, y cómo se usa el SDD.
- **`.github/instructions/*.instructions.md`**: reglas reutilizables aplicables a
  tipos de archivo o stacks concretos (ej. .NET, frontend, quality gate). Sirven
  para que Copilot aplique patrones específicos sin repetir texto en cada prompt.
  Se usan con el mecanismo de `applyTo`.
- **`.github/prompts/*.prompt.md`**: plantillas de ejecución para tareas
  específicas. Por ejemplo `/sdd-workflow` o una revisión final de calidad.
  Son prompts que el agente puede invocar para repetir un flujo estándar.
- **`.github/skills/*.md`**: documentación especializada de habilidades o
  enfoques reutilizables: code review, security review, quality gate, etc. Son
  más descriptivos que operativos; ayudan a guiar el razonamiento del agente en
  temas específicos.
- **Clean Code**: aplicar buenas prácticas de legibilidad, nombres claros,
  funciones pequeñas, coherencia de estilo y manejo correcto de errores, pero
  sin imponer dogmas ni refactors extremos. La idea es mejorar mantenibilidad,
  no convertir el código en un ejercicio académico.
- **Clean Architecture / architecture patterns**: priorizar diseño con
  dependencias dirigidas hacia el dominio y not hacia la infraestructura.
  Cuando la complejidad lo justifique, usar Factory, Strategy, Repository,
  Adapter, Unit of Work, CQRS o casos de uso bien definidos; siempre con
  criterio y no por “patrón por patrón”.
- **`_sdd/specs/template-feature/*`**: plantilla base para crear una feature real.
  Delimitan el formato de `requirements.md`, `design.md`, `tasks.md` y
  `feature.json` para que todas las specs sean consistentes.
- **`_sdd/specs/<JIRA-KEY>-<slug>/requirements.md`**: especificación de negocio.
  Aquí se escriben los requisitos en formato EARS (R1, R2, R3...). Define qué
  debe hacer el sistema y qué tipo de verificación corresponde: unitario o
  manual.
- **`_sdd/specs/<JIRA-KEY>-<slug>/design.md`**: diseño técnico. Explica impacto
  en capas, datos, endpoints, DTOs, controladores, vistas o frontend. Sirve para
  asegurar que el implementation se alinea con la arquitectura del proyecto.
- **`_sdd/specs/<JIRA-KEY>-<slug>/tasks.md`**: checklist de ejecución. Divide la
  tarea por fases y permite marcar avances con `[x]` solo cuando la tarea tiene
  evidencia de validación (test verde, build, etc.).
- **`_sdd/specs/<JIRA-KEY>-<slug>/feature.json`**: registro estructurado de la
  feature. Sirve como fuente de verdad para trazabilidad (`R<n>` → test), QA
  manual y estado de la feature. Es permanente y se commitea.
- **`_sdd/progress/current/<JIRA-KEY>.md`**: estado efímero de la feature en
  curso. Es la “memoria viva” del agente: estado actual, hechos verificados,
  bloqueos y tarea activa. No se commitea.
- **`_sdd/progress/history/<JIRA-KEY>.md`**: resumen narrativo final de la
  feature. Sirve para dejar evidencia histórica de la implementación y del
  cierre de la feature.

En resumen:

- `specs` define la intención del cambio.
- `progress/current` mantiene el estado operativo.
- `progress/history` deja la historia del cierre.
- `instructions`, `prompts` y `skills` guían cómo Copilot ejecuta el trabajo.
- `AGENTS.md` y `copilot-instructions.md` son las reglas y contexto del
  repositorio que le dan consistencia a la IA.

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
