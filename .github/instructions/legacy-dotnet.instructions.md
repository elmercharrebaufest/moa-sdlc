---
applyTo: "**/*.cs,**/*.csproj,**/*.sln,**/*.config,**/*.vb,**/*.asax,**/*.cshtml"
---

# Legacy .NET guidance

Usar esta guía para proyectos .NET Framework, MVC 5, Web Forms, Kendo, VB.NET o
soluciones que dependen de MSBuild/Visual Studio.

## Construcción y compilación

- Para soluciones legacy, preferir `MSBuild` en lugar de `dotnet build` cuando
  el proyecto no está basado en SDK-style.
- Ejemplos:
  - `msbuild MySolution.sln /p:Configuration=Debug /p:Platform="Any CPU"`
  - `msbuild MyProject.csproj /t:Build /p:Configuration=Release`
- Si el repositorio usa Web.config, Global.asax, paquetes NuGet antiguos o
  IIS, respetar ese modelo y no convertirlo a ASP.NET Core sin necesidad.

## Patrones de legado

- Mantener la capa de negocio y la capa de UI separadas.
- No mezclar lógica de presentación con lógica de acceso a datos ni validación
  de negocio.
- En MVC, mantener routing, model binding y validación del framework actual.
- En Kendo/Grid/jQuery, respetar el patrón del proyecto y la serialización
  existente.

## Regla de compatibilidad

- No cambiar contratos de APIs o firmas de controladores existentes sin
  aprobación explícita del humano.
- Si se necesita un cambio de contract, documentar impacto y compatibilidad.
- Evitar migraciones masivas o refactors de arquitectura que no se justifiquen
  con un problema concreto.
