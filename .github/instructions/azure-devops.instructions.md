---
applyTo: "**/*.{cs,csproj,sln,md,yml,yaml,json}"
---

# Azure DevOps guidance for Copilot

Este repositorio usa Azure DevOps como plataforma de entrega y validación.

## Reglas de DevOps

- Mantener los cambios alineados con el flujo de branches, PRs y revisiones del
  equipo en Azure Repos.
- Usar nombres de branch consistentes con el ticket: `feature/<JIRA-KEY>-<slug>`.
- Cada PR debe dejar evidencia clara de negocio, tests y QA manual cuando
  corresponda.
- Cuando el cambio afecta infraestructura o despliegue, documentar el impacto en
  el pipeline, variables y entornos de Azure DevOps.
- No asumir que el pipeline ejecuta solo `dotnet build`; en legacy .NET puede
  requerirse `MSBuild` y compilación con Visual Studio/Windows Build Agents.

## Pipeline expectations

- El pipeline debe validar build, tests y no regredir seguridad/arquitectura.
- Si hay análisis estático (SonarQube, StyleCop, analyzers), los resultados
  deben quedar visibles en el PR o en Azure DevOps.
- En `azure-pipelines.yml`, preferir stages y jobs claros por entorno:
  `build`, `test`, `package`, `deploy`.
- No ocultar warnings importantes ni desactivar validaciones del pipeline para
  “hacer pasar” un build.

## Revisión de PR

- En cada PR revisar: requisitos, trazabilidad, tests, riesgo, despliegue y QA.
- Si el cambio incluye APIs o contratos, validar compatibilidad hacia atrás y
  no romper consumidores existentes sin aprobación.
- Cuando exista riesgo manual, dejar evidencia en `feature.json` y en el PR.
