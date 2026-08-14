# Copilot Instructions for DataAgro Repository

## Spec-Driven Development (SDD)

Las features nuevas se planifican y ejecutan con SDD. Ver metodología y
manual de uso completo en [_sdd/docs/specs.md](../_sdd/docs/specs.md) y
límites de autonomía en [AGENTS.md](../AGENTS.md). Para arrancar o
continuar el ciclo de una feature, invocar el prompt `/sdd-workflow`.

## Build and Test Commands

### Building
Open the solution and build in Visual Studio, or use command line:
```bash
# Build entire solution (Debug)
dotnet build DataAgro.sln /p:Configuration=Debug

# Build entire solution (Release)
dotnet build DataAgro.sln /p:Configuration=Release

# Build specific project
dotnet build Molinos.DataAgro.Business/Molinos.DataAgro.Business.csproj
```

### Running Tests
Tests use NUnit and can be run through Visual Studio Test Explorer or command line:
```bash
# Run all tests
nunit3-console Molinos.DataAgro.Test\bin\Debug\Molinos.DataAgro.Test.dll

# Run tests matching a pattern (e.g., HomeController tests)
nunit3-console Molinos.DataAgro.Test\bin\Debug\Molinos.DataAgro.Test.dll --where "test.Namespace.Contains('HomeController')"

# Run a single test fixture
nunit3-console Molinos.DataAgro.Test\bin\Debug\Molinos.DataAgro.Test.dll --where "test.Class == 'Molinos.DataAgro.Test.Controllers.HomeControllerTest'"
```

### Database Deployment
Use the Web Deploy scripts in `Molinos.DataAgro.Build/DeployBat/`:
```bash
DataAgro.web.bat  # Deploys to the specified environment
```

Environment-specific parameters are in `.DeployParameters.xml` files (DEV, QA, PROD, etc.)

## Architecture Overview

The project follows a **layered architecture pattern**:

### Core Layers

- **WebDataAgro** (MVC Application)
  - ASP.NET MVC controllers, views, and web-specific logic
  - Kendo UI integration for data grids
  - Handles HTTP requests and responses

- **Molinos.DataAgro.Business** (Business Logic)
  - **Managers**: Core business logic (e.g., `CupoManager`, `ContratoManager`, `ComprasManager`)
  - Naming convention: All business classes end with `Manager`
  - Dependency injection: All `Manager` classes auto-registered by Autofac (see `Startup.Dependencias.cs`)

- **Molinos.DataAgro.Repository** (Data Access)
  - `RepositorioEF`: Entity Framework 6 data access implementation
  - `DataAgroDbContext`: DbContext for SQL Server database
  - Implements `IRepositorio` interface

- **Molinos.DataAgro.Entities**
  - DTOs and domain entities
  - Enums (e.g., in `Entities.Common.Enums`)

- **Molinos.DataAgro.Interfaces**
  - Interfaces for managers and services
  - Core `IRepositorio` and business manager interfaces

- **Molinos.DataAgro.Agent**
  - Batch processing agents
  - Naming convention: Classes end with `Agent`
  - Auto-registered by Autofac alongside managers

- **Molinos.DataAgro.Report**
  - Reporting functionality

- **Base de Datos** (SQL Server Database Project)
  - Schema migrations and stored procedures

### Cross-Cutting Concerns

- **Molinos.DataAgro.Test**: NUnit test project with Moq for mocking
- **Molinos.DataAgro.Build**: Deployment scripts and build utilities
- **Molinos.DataAgro.Mapping**: DTO-to-entity mapping logic
- **Molinos.DataAgro.Process**: Process orchestration
- **Molinos.DataAgro.ServiceHost / ServiceClient**: WCF service integration

## Key Conventions

### Dependency Injection (Autofac)
- All classes named `*Manager` in `Molinos.DataAgro.Business` are auto-registered
- All classes named `*Agent` in `Molinos.DataAgro.Agent` are auto-registered
- Services registered in `Startup.Dependencias.cs` with scope `InstancePerLifetimeScope`
- NLog is injected into all types (configured to use the requesting type's full name)

### Naming Conventions
- **Manager classes**: Inherit from or implement `I*Manager` interface
  - Examples: `CupoManager`, `ContratoManager`, `ComprasManager`
  - Located in: `Molinos.DataAgro.Business/Managers/`
- **Agent classes**: Handle async/batch operations
  - Named with suffix: `*Agent`
  - Located in: `Molinos.DataAgro.Agent/`
- **Controllers**: Located in `WebDataAgro/Controllers/`
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
- **DbContext**: `DataAgroDbContext` (in Repository project)
- **Connection**: Configured in `Web.config` and environment-specific `*.DeployParameters.xml` files

## Important Files and Directories

- `DataAgro.sln`: Main solution file
- `.github/copilot-instructions.md`: This file
- `WebDataAgro/App_Start/Startup.*.cs`: Configuration files (Autofac, Auth, Hangfire, Bundles, Routes, Filters)
- `Molinos.DataAgro.Business/Managers/`: 70+ business logic managers
- `Molinos.DataAgro.Test/`: Comprehensive NUnit test suite
- `Base de Datos/`: SQL Server database project
- Documentation in `WebDataAgro/DocumentacionCopilot/` for specific features

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
