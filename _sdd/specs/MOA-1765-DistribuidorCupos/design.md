# Technical Design - Distribuidor de Cupos SL

> Fuente de análisis detallado:
> `_sdd/specs/MOA-1765-DistribuidorCupos/plan-distribucionCuposSL-v2.md`
> (DTOs completos, algoritmo, CSS/JS, estimaciones). Este documento es
> el resumen operativo para trazabilidad SDD; ante cualquier duda de
> detalle, el plan es la fuente de verdad.

## 1. Mapa de Impacto Técnico

### 📂 Entidades y Datos (`Molinos.DataAgro.Entities`)

Carpeta nueva `Dto/Distribucion/` con los DTOs (ver plan §FASE 1 para el
detalle campo a campo):

- `ContratoSapImportadoDto`, `EstadisticasImportacionDto`,
  `ImportacionSapResultadoDto` (US-01)
- `DistribucionCuposRequestDto`, `DistribucionConfigDto`,
  `ResultadoContratoDistribucionDto`, `FilaSapDto`,
  `ResumenDistribucionDto`, `DistribucionResponseDto` (US-02)
- `LimiteDiaDto`, `DistribucionMultiDiaRequestDto`, `ResultadoDiaDto`,
  `ResumenGlobalDto`, `DistribucionMultiDiaResponseDto` (US-03)
- `ConfiguracionDistribucionDto` con `[Range(0.0, 1.0)]` en `CuitMaxPct`
  (US-05)
- `ProcesarEnDataAgroRequestDto` (US-06, reutiliza `FilaSapDto`)

Entity nueva `ConfiguracionDistribucionPlanta` (EF6) → tabla
`ConfiguracionesDistribucionPlanta`, con dicts/listas serializados como
`NVARCHAR(MAX)` JSON.

### 📂 Lógica de Negocio (`Molinos.DataAgro.Business/Managers`)

- **`SapArchivoParserManager`** (implementa `ISapArchivoParserManager`):
  parseo TSV/XLS/XLSX con detección de encoding (BOM UTF-16LE/BE/UTF-8),
  filtros `Centro/Tipo/Cosecha/DescCl`, cálculo de balance CCPP.
- **`DistribucionCuposManager`** (implementa `IDistribucionCuposManager`):
  `Calcular()` (un día, tope 30% CUIT + cuotas opcionales) y
  `CalcularMultiDia()` (hasta 7 días, descuento único de CCPP, reparto
  uniforme).
- **`ConfiguracionDistribucionManager`** (implementa
  `IConfiguracionDistribucionManager`): `ObtenerConfiguracion()` /
  `GuardarConfiguracion()` con validaciones y defaults.
- Reutiliza **`ICupoManager.AltaMasivaSugerenciaCuposV2(DataSet)`**
  (ya existente en `Molinos.DataAgro.Business/Managers/CupoManager.cs`)
  para US-06, sin modificarlo.

Todas las clases `*Manager` se auto-registran en Autofac por convención
de nombre (ver `Startup.Dependencias.cs`); no requiere cambios en la
configuración de DI.

### 📂 Acceso a Datos (`Molinos.DataAgro.Repository`)

- Uso del `IRepositorio` genérico existente; agregar
  `DbSet<ConfiguracionDistribucionPlanta>` en `DataAgroDbContext.cs`.
- Script SQL nuevo en `Base de Datos/Scripts/` para crear la tabla
  `ConfiguracionesDistribucionPlanta` y su seed inicial (ver plan
  §FASE 4). **Requiere aprobación humana** por ser un cambio de schema
  (ver `AGENTS.md`, sección "ASK FIRST").

### 📂 Web (`WebDataAgro`, MVC 5 + Kendo/Razor)

- **`DistribucionController`** — `[Autorizacion(PermisosDataAgro.SugerenciaDeCupos)]`,
  `GET /Distribucion` → sirve `Views/Distribucion/Index.cshtml`.
- **`ContratosApiController`** — `[RoutePrefix("api/contratos")]`,
  `POST importar` (US-01, `multipart/form-data`).
- **`DistribucionApiController`** — `[RoutePrefix("api/distribucion")]`,
  `POST calcular` (US-02), `POST calcular-multi-dia` (US-03),
  `POST procesar-en-dataagro` (US-06, inyecta `ICupoManager`).
- **`ConfiguracionDistribucionController`** — `[RoutePrefix("api/configuracion")]`,
  `GET distribucion` / `PUT distribucion` (US-05).
- Todos protegidos con `[Autorizacion]` reutilizando el permiso
  existente `SugerenciaDeCupos` (no se crea permiso nuevo — ver plan
  §FASE 5A-bis para la justificación).
- Requiere `routes.MapMvcAttributeRoutes()` en `App_Start/RouteConfig.cs`
  (agregar antes del route `Default`).
- Entrada nueva en el menú horizontal `Views/Shared/_Layout.cshtml`,
  dentro del dropdown `#cupos`, protegida por el mismo permiso.
- **Vista:** `Views/Distribucion/Index.cshtml`, `Layout = null`
  (pantalla completa, sin nav global), URLs de API inyectadas via
  `@Url.Action(...)`.
- **CSS:** `Content/distribucion-cupos.css` — extraído del POC,
  variables prefijadas `--dc-*`, sin base64 (imagen de header movida a
  `Content/Images/distribucion-cupos-header-bg.jpg`), con clases
  utilitarias reemplazando los ~102 `style="..."` inline (estáticos y
  generados por JS). Sigue
  `.github/instructions/instructions.frontend-template.md` (usar
  `template_dataagro.html` / `template_dataagro_instrucciones.md` como
  referencia de estilo de controles cuando la vista se integre al
  layout estándar o se agreguen controles nuevos no cubiertos por el
  POC).
- **JS:** `Scripts/Distribucion/distribucion-cupos.js` — solo UI
  (upload, render, export SheetJS, llamadas fetch); toda la lógica de
  cálculo (parseo, distribución) se elimina del JS y se mueve a los
  Managers de backend.
- Bundles nuevos registrados en `App_Start/BundleConfig.cs`
  (`StyleBundle` y `ScriptBundle`).

## 2. Contrato de Datos

Los contratos de request/response completos (JSON) para cada endpoint
están documentados en:
- `req/UserStories_API_Cupos.md` (formato funcional original, US-01 a
  US-05)
- `plan-distribucionCuposSL-v2.md` §FASE 1 y §FASE 8 (DTOs C# y US-06)

Resumen de endpoints:

| US | Método | Ruta | Controller |
|---|---|---|---|
| — | GET | `/Distribucion` | `DistribucionController.Index` |
| US-01 | POST | `/api/contratos/importar` | `ContratosApiController.Importar` |
| US-02 | POST | `/api/distribucion/calcular` | `DistribucionApiController.Calcular` |
| US-03 | POST | `/api/distribucion/calcular-multi-dia` | `DistribucionApiController.CalcularMultiDia` |
| US-05 | GET/PUT | `/api/configuracion/distribucion` | `ConfiguracionDistribucionController` |
| US-06 | POST | `/api/distribucion/procesar-en-dataagro` | `DistribucionApiController.ProcesarEnDataAgro` |

US-04 (exportar Excel) no tiene endpoint — se resuelve 100% en el
frontend con SheetJS (ver plan, sección "Decisión: US-04 Export Excel").

## 3. Estrategia de Pruebas

- **Tests NUnit** en `Molinos.DataAgro.Test`, con Moq para
  `IRepositorio`/managers dependientes, siguiendo el patrón existente
  del proyecto:
  - `SapArchivoParserManagerTests` — R2, R3, R4
  - `DistribucionCuposManagerTests` — R5, R6, R7, R8, R9, R10
  - `ConfiguracionDistribucionManagerTests` — R12, R13
  - Tests de `DistribucionApiController.ProcesarEnDataAgro` (o
    integración de `CupoManager`) — R14, R15
- **Requisitos tipo `manual`** (R1, R11, R16, R17): validación visual y
  funcional en la Vista Razor, no cubribles con NUnit — quedan
  documentados en `feature.json` → `qaManualPending` con `signOff: null`
  hasta validación humana (Analista Funcional, comparación lado a lado
  con el POC).
- Cambios de schema (tabla `ConfiguracionesDistribucionPlanta`) y el
  script SQL correspondiente requieren aprobación humana antes de
  aplicarse (ver `AGENTS.md`).
