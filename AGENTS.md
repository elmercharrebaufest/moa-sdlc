# AGENTS.md - Límites de Autonomía de IA

Cualquier agente/Copilot que interactúe con este repositorio debe apegarse
estrictamente a este mapa de límites.

## 1. ALWAYS (autónomo)
- Generar tests para toda lógica de negocio nueva o modificada.
- Actualizar `feature.json` (trazabilidad R<n> → test y QA manual
  pendiente) en `_sdd/specs/<JIRA-KEY>-<slug>/` al completar tareas.
- Crear branches como `feature/<JIRA-KEY>-<slug>`, siempre a partir de un
  ticket real traído por el MCP de Jira (nunca inventar el JIRA-KEY).
- Escribir memoria de sesión en `_sdd/progress/current/<JIRA-KEY>.md`.
- Escribir el resumen narrativo en `_sdd/progress/history/<JIRA-KEY>.md`
  al cerrar la feature.
- Formatear los archivos editados con el linter del proyecto.

## 2. ASK FIRST (pedir aprobación humana)
- Instalar dependencias npm o agregar paquetes NuGet.
- Modificar entidades que generen nuevas migraciones de EF Core.
- Cambiar firmas de controladores de API existentes o contratos externos.
- Modificar o desactivar tests unitarios previamente definidos.

## 3. NEVER (prohibido)
- Ejecutar `git push` sin acción explícita del humano.
- Abrir Pull Requests por sí mismo (los abre siempre el dev).
- Commitear `_sdd/progress/current/*` (es memoria local, va en .gitignore).
- Marcar una tarea `[x]` en tasks.md sin su test correspondiente en verde.
- Marcar un requisito de tipo "manual" como cubierto sin sign-off humano.
- Desactivar validaciones de seguridad, CORS, reglas de autorización o JWT.
- Confirmar o subir contraseñas, archivos .env, tokens o API keys.
- Desactivar el linter (`// eslint-disable`, `#pragma warning disable`) para
  ocultar un warning sin resolver la causa.
- Deploy autónomo o merge directo a `main`/`Dev` sin PR y review humano.
