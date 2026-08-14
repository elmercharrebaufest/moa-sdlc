---
applyTo: "**/*.{cs,csproj,sln,ts,tsx,js,json,yml,yaml,xml,props,targets}"
---

# Tooling and quality rules

## Regla general

No deshabilitar validaciones del compilador, del linter, de analyzers ni de
quality gates para ocultar un problema. El código debe corregirse en la causa,
no silenciarse.

## .NET / build quality

- Usar analyzers y StyleCop si el proyecto los soporta.
- Preferir `warnings as errors` en pipelines y builds de integración.
- No introducir `#pragma warning disable` o `SuppressMessage` salvo que exista
  una justificación explícita, documentada y temporal.
- Resolver `nullability`, `CA2000`, `CA1822`, `IDE...`, `SA...` y warnings de
  seguridad antes de cerrar la historia.

## SonarQube / static analysis

- Revisar bugs, vulnerabilities, code smells y duplicación antes de aprovar un
  PR.
- No dejar logs con datos sensibles ni mensajes internos que expongan tokens o
  secretos.
- Evitar `catch` vacío, `throw new Exception` genérico y validaciones débiles.

## Azure DevOps / pipeline gating

- Cada pipeline debe incluir validaciones de build, tests y análisis estático.
- Usar policies de branch y calidad para bloquear merge si falla la compilación o
  los tests.
- Cuando el proyecto requiere paquetes, SDKs o agentes específicos, documentarlo
  en el pipeline y en la guía del repo.

## Frontend tooling

- En Angular/SPA usar linting y tests unitarios como gate antes de merge.
- No mezclar tipos `any` sin justificación clara; preferir tipado estricto.
- Si se usa ESLint/TSLint/Prettier, mantener la configuración compartida en el
  repo y no silenciar reglas sin causa.
