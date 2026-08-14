# SDLC con IA

Plantilla de especificación y operación para proyectos .NET con agentes de IA,
orientada a repositorios con mezcla de stacks: .NET Framework legacy, .NET modern,
MVC, Razor, ASP.NET Core, Angular y frontend legado.

## Objetivo

Este repositorio define un template base para trabajar con Copilot/IA usando:

- Spec-Driven Development (SDD)
- Reglas de arquitectura y calidad
- Uso de patrones cuando se justifican
- Enfoque de Clean Code y Clean Architecture
- Validación de calidad previa a SonarQube
- Integración con Azure DevOps como plataforma de desarrollo y CI/CD

## Estructura principal

- `AGENTS.md`: límites de autonomía de la IA y reglas de seguridad.
- `.github/copilot-instructions.md`: instrucciones generales del repositorio.
- `.github/instructions/`: reglas reutilizables por stack o foco.
  - `net-application-patterns.instructions.md`
  - `legacy-dotnet.instructions.md`
  - `angular.instructions.md`
  - `azure-devops.instructions.md`
  - `tooling-rules.instructions.md`
  - `code-quality-review.instructions.md`
- `.github/prompts/`: prompts reutilizables para flujo de trabajo y revisión.
- `.github/skills/`: documentación orientada a habilidades especializadas.
- `_sdd/specs/`: especificaciones permanentes por feature.
- `_sdd/progress/current/`: estado efímero de la feature actual.
- `_sdd/progress/history/`: resumen histórico final.

## Flujo recomendado

1. Crear la feature desde un ticket real.
2. Generar la carpeta de spec bajo `_sdd/specs/<JIRA-KEY>-<slug>/`.
3. Definir requisitos EARS (R1, R2, R3...).
4. Diseñar impacto técnico, endpoints, capas y DTOs.
5. Ejecutar la implementación con validación de pruebas.
6. Completar `feature.json` con trazabilidad y QA manual.
7. Dejar resumen final en `progress/history`.

## Regla de arquitectura

Se prioriza:

- separación de responsabilidades
- dependencias dirigidas hacia el dominio
- capas claras
- inyección de dependencias
- patrones cuando aporten claridad y testabilidad
- Clean Architecture cuando la complejidad lo justifique

No se fuerza una arquitectura específica cuando un proyecto legacy ya tiene
patrones estabilizados; se debe respetar el contexto existente y solo introducir
patrones adicionales si son necesarios y justificables.

## Build y test

### .NET moderno / SDK-style

```bash
dotnet restore
dotnet build

dotnet test
```

### .NET Framework legacy

```bash
msbuild DataAgro.sln /p:Configuration=Debug /p:Platform="Any CPU"
msbuild MyProject.csproj /t:Build /p:Configuration=Release
```

### Azure DevOps

Este template asume Azure DevOps / Azure Repos como backend de delivery y CI/CD.
Se recomienda:

- branch policies
- PR reviews con checklist
- pipeline build + tests + análisis estático
- validación de deployment por entorno

## Tooling y calidad

Las reglas de calidad no se silencia. Debe preferirse:

- analyzers
- StyleCop / code analysis
- warnings as errors en CI
- SonarQube o análisis estático
- revisión explícita de seguridad, bugs y code smells

La guía de calidad está en:

- `.github/instructions/tooling-rules.instructions.md`
- `.github/instructions/code-quality-review.instructions.md`
- `.github/prompts/quality-gate-review.prompt.md`

## Uso con Copilot

Copilot debe operar con estos principios:

- no tocar código antes de verificar la especificación
- no saltarse los límites de AGENTS.md
- no ocultar errores con silenciadores
- generar tests para lógica nueva o modificada
- documentar QA manual cuando aplica

## Recomendaciones para usar este template

- Usar `feature.json` como fuente estructurada de trazabilidad.
- Mantener `requirements.md` con EARS y requisitos numerados.
- Usar `design.md` para documentar impacto técnico.
- Usar `tasks.md` para validar progreso y cierre real.
- Mantener `progress/current` solo como memoria operativa local y no committeada.

## Qué usar cuándo

### Si el equipo está definiendo una feature nueva

Usar:

- `/_sdd/specs/template-feature/requirements.md`
- `/_sdd/specs/template-feature/design.md`
- `/_sdd/specs/template-feature/tasks.md`
- `/_sdd/specs/template-feature/feature.json`
- el prompt `/sdd-workflow`
- roles especializados: `spec-author`, `implementer`, `tester`, `reviewer`,
  `security-reviewer`

Esto permite empezar desde especificación, no desde código. Es el flujo correcto
para requisitos complejos o cambios con impacto en varios componentes.

### Mapa operativo del loop por agente

Cuando se trabaja con IA en equipo, conviene diferenciar responsabilidades:

- `spec-author` / planner:
  - define el problema y la forma esperada de resolverlo
  - redacta requisitos, diseño y tareas
  - se ejecuta al arrancar la feature
- `implementer`:
  - escribe el código mínimo viable y consistente
  - se usa cuando hay que producir cambio real
- `tester`:
  - corre pruebas y verifica comportamiento
  - escribe o valida evidencia de validación
- `reviewer`:
  - valida no regresión, arquitectura, calidad y trazabilidad
- `security-reviewer`:
  - revisa amenazas de seguridad y controles
- `human-approver`:
  - aprueba PR, QA manual y sign-off final

El flujo real es:

```
spec-author → implementer → tester → reviewer → security-reviewer → human-approver
      ↑                                                       ↓
      └───────────────────── feedback loop / fix cycle ─────┘
```

Si falla una etapa, se vuelve al agente responsable y se corrige antes de
aprovechar el PR final.

### Si hay que cambiar un controller o endpoint backend

Usar:

- `.github/instructions/net-application-patterns.instructions.md`
- `.github/instructions/legacy-dotnet.instructions.md` si es .NET Framework
- `.github/instructions/security-review.instructions.md`
- `.github/instructions/tooling-rules.instructions.md`

Esto garantiza: arquitectura, validación, seguridad y compatibilidad.

### Si hay que trabajar en Angular / frontend SPA

Usar:

- `.github/instructions/angular.instructions.md`
- `.github/instructions/security-review.instructions.md`
- `.github/instructions/tooling-rules.instructions.md`

Esto ayuda a separar UI, servicios, estado, validación y seguridad del cliente.

### Si el proyecto es .NET Framework legacy (MVC 5 / Kendo / Web.config)

Usar:

- `.github/instructions/legacy-dotnet.instructions.md`
- `.github/instructions/net-application-patterns.instructions.md`
- `azure-pipelines.yml` y Azure DevOps rules

Esto evita migrar patrones sin necesidad y se ajusta al stack real.

### Si el cambio ya está hecho y hay que revisar calidad antes del merge

Usar:

- `.github/instructions/code-quality-review.instructions.md`
- `.github/instructions/security-review.instructions.md`
- `.github/prompts/quality-gate-review.prompt.md`
- `.github/skills/sonarqube-quality-review.md`

Esto sirve como pre-gate antes de SonarQube o antes de aceptar un PR.

### Si el equipo necesita operar con Azure DevOps

Usar:

- `.github/instructions/azure-devops.instructions.md`
- `.github/pull_request_template.md`
- `.github/CODEOWNERS`
- `azure-pipelines.yml`

Esto mantiene estándares de branch, PR, revisión y validación continua.

## Workflow por defecto de una feature

Este es el flujo recomendado para cualquier ticket nuevo:

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

### Regla práctica

- Si falla el requisito, vuelve al `spec-author`.
- Si falla la implementación, vuelve al `implementer`.
- Si falla la validación, vuelve al `tester`.
- Si falla arquitectura/calidad, vuelve al `reviewer`.
- Si falla seguridad, vuelve al `security-reviewer`.
- Si falta aprobación, el PR se queda bloqueado hasta el `human-approver`.

## Matriz rápida de uso por agente

| Etapa | Agente | Qué valida | Artefacto clave |
|---|---|---|---|
| Spec | `spec-author` | requisitos, diseño, alcance y tareas | `requirements.md`, `design.md`, `tasks.md` |
| Implementación | `implementer` | código, compatibilidad y arquitectura | código + cambios del PR |
| Verificación | `tester` | tests, evidencia y comportamiento | `feature.json`, logs/test output |
| Calidad | `reviewer` | trazabilidad, regresión, maintainability | `feature.json`, review comments |
| Seguridad | `security-reviewer` | auth, secretos, XSS, SQLi, exposición | security review findings |
| Aprobación | `human-approver` | sign-off, QA manual y merge | PR final |

## Repositorio de referencia

El template está pensado para adaptarse a proyectos reales con arquitectura
heterogénea, especialmente aquellos que mezclan tecnologías y requieren control
formal de cambios con Azure DevOps.
