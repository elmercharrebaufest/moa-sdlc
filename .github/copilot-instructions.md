# Copilot Instructions for Project Repository

## Spec-Driven Development (SDD)

Las features nuevas se planifican y ejecutan con SDD. Ver metodología y
manual de uso completo en [_sdd/docs/specs.md](../_sdd/docs/specs.md) y
límites de autonomía en [AGENTS.md](../AGENTS.md). Para arrancar o
continuar el ciclo de una feature, invocar el prompt `/sdd-workflow`.

## Plantillas y guías aplicables

Este repositorio incluye plantillas y guías reutilizables para proyectos
.NET y frontend heterogéneos:

- `.github/instructions/net-application-patterns.instructions.md`:
  guía general para .NET Framework y .NET moderno, MVC, Razor, ASP.NET Core,
  Angular y validación de arquitectura.
- `.github/instructions/legacy-dotnet.instructions.md`:
  reglas específicas para soluciones .NET Framework/legacy que requieren MSBuild,
  Web.config, MVC 5, Kendo y compatibilidad de contratos.
- `.github/instructions/angular.instructions.md`:
  patrones para Angular o SPA.
- `.github/instructions/azure-devops.instructions.md`:
  lineamientos para Azure Repos, Azure Pipelines y revisión de PR.
- `.github/instructions/tooling-rules.instructions.md`:
  reglas de herramientas y calidad: analyzers, SonarQube, warnings as errors,
  pipeline gating y frontend lint/test.
- `.github/instructions/agent-harness-patterns.instructions.md`:
  guía de harness para agentes IA: estados, roles, ciclo, evidencia y gate
  humano.
- `.github/AGENTS-HARNESS.md`:
  modelo operativo del harness para equipo y agentes, con estados, roles,
  transiciones y definition of done.
- `.github/AGENTS-CONTRACTS.md`:
  contrato operativo para agentes IA: entrada/salida, criterios de éxito,
  allowlist de tools, estados formales, audit trail y definición de done
  estructurada.
- `.github/instructions/security-review.instructions.md`:
  checklist de seguridad para backend, frontend, APIs, Azure DevOps y despliegue.
- `.github/instructions/code-quality-review.instructions.md`:
  checklist de calidad antes de aceptar cambios, pensado para pasar luego por
  SonarQube o análisis estático.
- `.github/prompts/quality-gate-review.prompt.md`:
  prompt reutilizable para revisión final de PR o diff.
- `.github/prompts/implementer.prompt.md`: guía para la fase de implementación.
- `.github/prompts/tester.prompt.md`: guía para pruebas y verificación.
- `.github/prompts/reviewer.prompt.md`: guía para calidad y trazabilidad.
- `.github/prompts/security-reviewer.prompt.md`: guía para análisis de seguridad.
- `_sdd/specs/template-feature/*`:
  base para tickets con requisitos, diseño, tareas y trazabilidad.

## Build and Test Commands

### Building
Open the solution and build in Visual Studio, or use command line. For modern
SDK-style .NET projects use `dotnet build`; for legacy .NET Framework projects use
`MSBuild` / Visual Studio build agent.
```bash
# .NET modern / SDK-style
# Build entire solution (Debug)
dotnet build MyProject.sln /p:Configuration=Debug

# Build entire solution (Release)
dotnet build MyProject.sln /p:Configuration=Release

# Build specific project
dotnet build Example.MyProject.Business/Example.MyProject.Business.csproj

# Legacy .NET Framework / ASP.NET MVC 5
msbuild MyProject.sln /p:Configuration=Debug /p:Platform="Any CPU"
msbuild Example.MyProject.Business/Example.MyProject.Business.csproj /t:Build /p:Configuration=Release
```

### Running Tests
Tests use NUnit and can be run through Visual Studio Test Explorer or command line:
```bash
# Run all tests
nunit3-console Example.MyProject.Test\bin\Debug\Example.MyProject.Test.dll

# Run tests matching a pattern (e.g., HomeController tests)
nunit3-console Example.MyProject.Test\bin\Debug\Example.MyProject.Test.dll --where "test.Namespace.Contains('HomeController')"

# Run a single test fixture
nunit3-console Example.MyProject.Test\bin\Debug\Example.MyProject.Test.dll --where "test.Class == 'Example.MyProject.Test.Controllers.HomeControllerTest'"
```

### Database Deployment
Use the Web Deploy scripts in `Example.MyProject.Build/DeployBat/`:
```bash
MyProject.web.bat  # Deploys to the specified environment
```

Environment-specific parameters are in `.DeployParameters.xml` files (DEV, QA, PROD, etc.)

## Architecture Overview

The project follows a **layered architecture pattern**:

### Core Layers

- **WebMyProject** (MVC Application)
  - ASP.NET MVC controllers, views, and web-specific logic
  - Kendo UI integration for data grids
  - Handles HTTP requests and responses

- **Example.MyProject.Business** (Business Logic)
  - **Managers**: Core business logic (e.g., `CupoManager`, `ContratoManager`, `ComprasManager`)
  - Naming convention: All business classes end with `Manager`
  - Dependency injection: All `Manager` classes auto-registered by Autofac (see `Startup.Dependencias.cs`)

- **Example.MyProject.Repository** (Data Access)
  - `RepositorioEF`: Entity Framework 6 data access implementation
  - `MyProjectDbContext`: DbContext for SQL Server database
  - Implements `IRepositorio` interface

- **Example.MyProject.Entities**
  - DTOs and domain entities
  - Enums (e.g., in `Entities.Common.Enums`)

- **Example.MyProject.Interfaces**
  - Interfaces for managers and services
  - Core `IRepositorio` and business manager interfaces

- **Example.MyProject.Agent**
  - Batch processing agents
  - Naming convention: Classes end with `Agent`
  - Auto-registered by Autofac alongside managers

- **Example.MyProject.Report**
  - Reporting functionality

- **Base de Datos** (SQL Server Database Project)
  - Schema migrations and stored procedures

### Cross-Cutting Concerns

- **Example.MyProject.Test**: NUnit test project with Moq for mocking
- **Example.MyProject.Build**: Deployment scripts and build utilities
- **Example.MyProject.Mapping**: DTO-to-entity mapping logic
- **Example.MyProject.Process**: Process orchestration
- **Example.MyProject.ServiceHost / ServiceClient**: WCF service integration

## Key Conventions

### Dependency Injection (Autofac)
- All classes named `*Manager` in `Example.MyProject.Business` are auto-registered
- All classes named `*Agent` in `Example.MyProject.Agent` are auto-registered
- Services registered in `Startup.Dependencias.cs` with scope `InstancePerLifetimeScope`
- NLog is injected into all types (configured to use the requesting type's full name)

### Naming Conventions
- **Manager classes**: Inherit from or implement `I*Manager` interface
  - Examples: `CupoManager`, `ContratoManager`, `ComprasManager`
  - Located in: `Example.MyProject.Business/Managers/`
- **Agent classes**: Handle async/batch operations
  - Named with suffix: `*Agent`
  - Located in: `Example.MyProject.Agent/`
- **Controllers**: Located in `WebMyProject/Controllers/`
- **Tests**: Mirror source structure (e.g., test for `HomeController` is `HomeControllerTest`)

### Testing with NUnit
- **Test Fixture**: Use `[TestFixture]` attribute on test classes
- **Test Methods**: Use `[Test]` attribute
- **Mocking**: Use Moq library
  - Example: `var mock = new Mock<IHomeManager>();`
- **Verify**: Use `mock.Verify()` to confirm method calls

### Localization
- Culture set to `es-AR` (Spanish - Argentina)
- Date format: `dd/MM/yyyy`
- Applied in `Global.asax.cs` Application_Start

### Web UI Integration
- **Kendo UI** used for data grids
- **jQuery AJAX** for async calls
- **KendoGridMvcModelBinder** for binding grid requests
- **Bundle config** in `App_Start/BundleConfig.cs` for CSS/JS optimization

## Important Patterns

### Manager Pattern
All business logic flows through Manager classes:
```csharp
// Managers always have an interface
IHomeManager homeManager;
var result = homeManager.GetSomeData();
```

### Repository Pattern
Data access through `IRepositorio` interface:
```csharp
IRepositorio repository;
var entity = repository.GetById<Entity>(id);
```

### Service Injection
Controllers receive managers and services via constructor injection (Autofac):
```csharp
public HomeController(IHomeManager homeManager, ILogger logger)
{
    _homeManager = homeManager;
    _logger = logger;
}
```

### Logging
NLog is configured per type:
```csharp
private ILogger _logger; // Injected via Autofac
_logger.Info("Message");
_logger.Error("Error", exception);
```

## Database Context

- **ORM**: Entity Framework 6
- **Database**: SQL Server
- **DbContext**: `MyProjectDbContext` (in Repository project)
- **Connection**: Configured in `Web.config` and environment-specific `*.DeployParameters.xml` files

## Important Files and Directories

- `MyProject.sln`: Main solution file
- `.github/copilot-instructions.md`: This file
- `WebMyProject/App_Start/Startup.*.cs`: Configuration files (Autofac, Auth, Hangfire, Bundles, Routes, Filters)
- `Example.MyProject.Business/Managers/`: 70+ business logic managers
- `Example.MyProject.Test/`: Comprehensive NUnit test suite
- `Base de Datos/`: SQL Server database project
- Documentation in `WebMyProject/DocumentacionCopilot/` for specific features

## Deployment Environments

- **DEV**: `DEV.DeployParameters.xml`
- **QA**: `QA.DeployParameters.xml`, `QA2.DeployParameters.xml`, `QA3.DeployParameters.xml`
- **PreProd**: `PreProd.DeployParameters.xml`
- **AWS QA**: `AWS.QA2.DeployParameters.xml`, `AWS.QA3.DeployParameters.xml`
- **AWS Prod**: `AWS.PRD2.DeployParameters.xml`
- **PROD**: `PROD.DeployParameters.xml`

## Code Style Notes

- Spanish naming is used extensively throughout the codebase (manager names, entity properties, comments)
- Comments and documentation are in Spanish
- Use consistent with existing patterns when extending functionality
