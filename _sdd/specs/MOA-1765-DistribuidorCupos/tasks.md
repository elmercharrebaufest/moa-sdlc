# Tasks Checklist - Distribuidor de Cupos SL

El agente marca cada tarea completada como `[x]`. Ninguna tarea se marca
sin su test correspondiente en verde (o, si es tipo `manual`, sin quedar
listada en `qaManualPending` de `feature.json`).

Detalle completo de cada archivo/DTO en
`plan-distribucionCuposSL-v2.md`.

## Fase 1: Datos y Lógica de Negocio

- [x] Crear los DTOs en `Molinos.DataAgro.Entities/Dto/Distribucion/`
      (US-01 a US-06, ver plan §FASE 1).
- [x] Crear la entidad `ConfiguracionDistribucionPlanta` en
      `Molinos.DataAgro.Entities/`.
- [x] Crear las interfaces `ISapArchivoParserManager`,
      `IDistribucionCuposManager`, `IConfiguracionDistribucionManager`
      en `Molinos.DataAgro.Interfaces/Managers/`.
- [x] Implementar `SapArchivoParserManager` (parseo TSV/XLS/XLSX,
      detección de encoding, filtros, balance CCPP) — R2, R3, R4.
- [x] Implementar `DistribucionCuposManager.Calcular()` (tope 30% CUIT,
      cuotas opcionales, material sin límite) — R5, R6, R7.
- [x] Implementar `DistribucionCuposManager.CalcularMultiDia()`
      (máx. 7 días, descuento único CCPP, reparto uniforme) — R8, R9,
      R10.
- [x] Implementar `ConfiguracionDistribucionManager` (defaults,
      validaciones, persistencia) — R12, R13.

## Fase 2: Acceso a Datos

- [x] Script SQL de creación de tabla
      `Base de Datos/dbo/Tables/ConfiguracionesDistribucionPlanta.sql`
      (proyecto SSDT, sin publish/deploy — ver nota). No se agregó
      `DbSet` explícito: `DataAgroDbContext` automapea por reflexión
      todas las entidades del namespace `Molinos.DataAgro.Entities.Entities`
      (`OnModelCreating` → `MapearAssemblyDe<Contrato>`), por lo que la
      entidad `ConfiguracionDistribucionPlanta` no requiere cambios en
      el DbContext.

## Fase 3: Web (MVC + Vista Razor)

- [x] `routes.MapMvcAttributeRoutes()` ya existía en
      `App_Start/RouteConfig.cs` (sin cambios necesarios).
- [x] Crear `DistribucionController` (`GET /Distribucion`) — R1.
- [x] Crear `ContratosApiController` (`POST importar`) — R2, R4.
- [x] Crear `DistribucionApiController` (`calcular`,
      `calcular-multi-dia`, `procesar-en-dataagro`) — R5 a R10, R14,
      R15.
- [x] Crear `ConfiguracionDistribucionController` (`GET`/`PUT
      distribucion`) — R12, R13.
- [x] Agregar entrada "Distribuidor Cupos" en `Views/Shared/_Layout.cshtml`
      dentro del dropdown `#cupos`, protegida por el permiso existente
      `SugerenciaDeCupos`.
- [x] Crear `Views/Distribucion/Index.cshtml` (migración del POC,
      `Layout = null`, URLs inyectadas via `@Url.Action`) — R16.
- [x] Extraer `Content/distribucion-cupos.css` (variables `--dc-*`,
      clases utilitarias, sin estilos inline, sin base64) siguiendo
      `.github/instructions/instructions.frontend-template.md` — R17.
- [x] Extraer imagen de fondo del header a
      `Content/Images/distribucion-cupos-header-bg.jpg` — R17.
- [x] Extraer `Scripts/Distribucion/distribucion-cupos.js` (solo UI;
      eliminar `parseFile`, `processRows`, `distribute`,
      `distributeMultiDay` y constantes de negocio, que pasan al
      backend) — R11, R16.
- [x] Registrar bundles CSS/JS en `App_Start/BundleConfig.cs`.
- [x] Agregar botón "⚡ Procesar en DataAgro" en la Vista y su handler
      JS `procesarEnDataAgro()` — R14, R15.

## Fase 4: Suite de Pruebas y Verificación

- [x] `SapArchivoParserManagerTests` — cubre R2, R3, R4.
- [x] `DistribucionCuposManagerTests` — cubre R5, R6, R7, R8, R9, R10.
- [x] `ConfiguracionDistribucionManagerTests` — cubre R12, R13.
- [x] Tests de `ProcesarEnDataAgro` (conversión a OADate, validación de
      `sap` vacío, manejo de excepción) — cubre R14, R15.
- [x] Correr `MSBuild` (build completo) y `vstest.console`/NUnit —
      1228/1228 tests en verde, sin regresiones.
- [ ] Validación visual manual (Analista Funcional) comparando la Vista
      contra el POC `distribuidor_cupos.html` — R1, R11, R16, R17.
- [ ] Completar `feature.json` de esta carpeta (`traceability` y
      `qaManualPending`) antes de abrir el PR final.
