# Plan: Distribución de Cupos SL — Frontend ASP.NET MVC

> **Versión:** 3.0 — Frontend migrado a ASP.NET MVC Razor Views; cálculos 100% en el backend  
> **Fecha:** 2026-06-30  
> **Reemplaza:** `plan-distribucionCuposSL-v2.md`

---

## Cambios respecto al plan v2

| Plan v2 | Plan v3 (este documento) |
|---|---|
| `distribuidor_cupos.html` (standalone HTML + JS con cálculos client-side) | Razor View en `WebDataAgro/Views/Distribucion/Index.cshtml` |
| Lógica de parseo (`parseFile`, `processRows`) en JavaScript | Movida a `SapArchivoParserManager` (backend) |
| Algoritmo de distribución (`distribute`) en JavaScript | Movido a `DistribucionCuposManager` (backend) |
| CORS necesario (HTML abierto localmente o desde otro origen) | **CORS eliminado** — mismo origen, misma app |
| `AllowCorsAttribute` + preflight en `Global.asax` | No requerido |
| `fetch('/api/...')` con URL hard-coded | `fetch('@Url.Action(...)')` via Razor |
| HTML sin autenticación | Protegida por `[Autorizacion]` igual que el resto de la app |
| CSS inline dentro del HTML (`<style>` + ~102 atributos `style="..."` inline en markup y en JS) | Extraído a `Content/distribucion-cupos.css` con clases utilitarias (ver FASE 6A) + bundle |
| JS mezclado con HTML | Extraído a `Scripts/Distribucion/distribucion-cupos.js` + bundle |

---

## Alcance: 5 Endpoints REST + 1 Vista MVC

> **US-04 (Export Excel) no es un endpoint del backend.**  
> El frontend genera el `.xlsx` con SheetJS, incluyendo formato texto en la columna `ContratoSAP`. El backend devuelve el array `sap` en las responses de US-02/US-03, y el frontend exporta directamente.

| US | Tipo | Método | Ruta | Descripción |
|---|---|---|---|---|
| — | Vista MVC | GET | `/Distribucion` | Sirve la pantalla principal (Razor View) |
| US-01 | API | POST | `/api/contratos/importar` | Parsear archivo SAP (TSV/XLS/XLSX) + calcular balance CCPP |
| US-02 | API | POST | `/api/distribucion/calcular` | Distribución un día |
| US-03 | API | POST | `/api/distribucion/calcular-multi-dia` | Distribución multi-día (máx. 7 días) |
| US-04 | Frontend | — | — | Export Excel con SheetJS (sin endpoint backend) |
| **US-06** | **API** | **POST** | **`/api/distribucion/procesar-en-dataagro`** | **Enviar resultado al algoritmo de DataAgro** |
| US-05 | API | GET/PUT | `/api/configuracion/distribucion` | Leer/guardar parámetros de planta |

---

## Arquitectura de Capas

```
Browser (Razor View)
│  Views/Distribucion/Index.cshtml
│  Content/distribucion-cupos.css
│  Scripts/Distribucion/distribucion-cupos.js  (solo UI, sin cálculos)
│
│  fetch() — misma app, mismo origen, sin CORS
│
WebDataAgro (Controllers MVC5)
├─ DistribucionController         ← GET /Distribucion (sirve la Vista)
├─ ContratosApiController         ← POST /api/contratos/importar
├─ DistribucionApiController      ← POST /api/distribucion/calcular[‑multi‑dia]
└─ ConfiguracionDistribucionController ← GET/PUT /api/configuracion/distribucion
        │
Molinos.DataAgro.Business
├─ SapArchivoParserManager        ← parseo del archivo + cálculo CCPP
├─ DistribucionCuposManager       ← algoritmo distribución 1 día y multi-día
└─ ConfiguracionDistribucionManager ← persistencia de configuración
        │
Molinos.DataAgro.Repository
└─ IRepositorio (genérico existente)
        │
Base de Datos SQL Server
└─ ConfiguracionesDistribucionPlanta (tabla nueva)
```

---

## FASE 1: DTOs y Entities (`Molinos.DataAgro.Entities`)

### Carpeta nueva: `Dto/Distribucion/`

#### US-01 — Parseo

| Clase | Descripción |
|---|---|
| `ContratoSapImportadoDto` | Todos los campos del contrato parseado: `Numero`, `NumeroSAP`, `DescCl`, `PrioLabel`, `Rank`, `Material`, `Kg`, `Cosecha`, `FechaContrato`, `FechaDesde`, `FechaHasta`, `Cuit`, `Proveedor`, `Corredor`, `Clasificacion`, `OpType`, `OpLabel`, `IsSust`, `Valor`, `Moneda`, `PricePt`, `PriceRank` |
| `EstadisticasImportacionDto` | Stats por operador, por clase, sustentables, fechas min/max |
| `ImportacionSapResultadoDto` | Contenedor de respuesta US-01: `Contratos`, `CcppByCuitMat`, `CcppByMat`, `Materiales`, `Estadisticas`, `Errores` |

#### US-02 — Distribución un día

| Clase | Descripción |
|---|---|
| `DistribucionCuposRequestDto` | `Fecha`, `Contratos`, `CcppByCuitMat`, `LimitesPorMaterial`, `Configuracion` |
| `DistribucionConfigDto` | `AplicarCuotaOperador`, `CuotasPorOperador`, `AplicarCuotaClase`, `CuotasPorClase`, `ExcluirFason`, `ExcluirAgenteCompra`, `PriorizarSustentables`, `PreciosReferencia`, `HabilitarFiltroFechas`, filtros de fecha |
| `ResultadoContratoDistribucionDto` | `NumeroSAP`, `Proveedor`, `Cuit`, `Material`, `OpType`, `Clase`, `KgContrato`, `CcppDescontado`, `KgEfectivo`, `CuposNecesarios`, `CuposAsignados`, `Estado` (`sin_cupos`/`parcial`/`completo`/`sin_tope`), `IsSust`, `PricePt` |
| `FilaSapDto` | `FechaSugerida` (dd/MM/yyyy), `CantidadDeCupos`, `ContratoSAP` |
| `ResumenDistribucionDto` | `Material`/`Operador`/`Clase`, `Limite`, `CuposAsignados`, `CuposRestantes`, `ContratosAsignados`, `ContratosSinCupos` |
| `DistribucionResponseDto` | `Fecha`, `Resultados`, `ResumenPorMaterial`, `ResumenPorOperador`, `ResumenPorClase`, `Sap` |

#### US-03 — Distribución multi-día

| Clase | Descripción |
|---|---|
| `LimiteDiaDto` | `Fecha`, `Limites` (dict material → cupos) |
| `DistribucionMultiDiaRequestDto` | `Contratos`, `CcppByCuitMat`, `Fechas`, `LimitesPorDia`, `DistribuirUniforme`, `Configuracion` |
| `ResultadoDiaDto` | `Fecha`, `Resultados`, `Sap` |
| `ResumenGlobalDto` | `TotalCuposAsignados`, `TotalContratosAsignados`, `PorcentajeCoberturaKg` |
| `DistribucionMultiDiaResponseDto` | `ResultadosPorDia`, `SapConsolidado`, `ResumenGlobal` |

#### US-05 — Configuración

| Clase | Descripción |
|---|---|
| `ConfiguracionDistribucionDto` | `PlantaCodigo`, `PlantaNombre`, `CupoKg`, `CuitMaxPct` (con `[Range(0.0, 1.0)]`), `CosechasValidas`, `ClasesExcluidas`, `LimitesPredeterminadosPorMaterial`, `PreciosReferencia`, `CuotasPorOperador`, `CuotasPorClase`, `UltimaActualizacion` |

### Carpeta: `Entities/`

| Clase | Descripción |
|---|---|
| `ConfiguracionDistribucionPlanta` | Entity EF6 → tabla `ConfiguracionesDistribucionPlanta`. Campos JSON serializados como `NVARCHAR(MAX)` para dicts y listas |

---

## FASE 2: Interfaces (`Molinos.DataAgro.Interfaces/Managers`)

```csharp
// ISapArchivoParserManager.cs
public interface ISapArchivoParserManager
{
    ImportacionSapResultadoDto ParsearArchivo(Stream stream, string extension, DateTime fecha);
}

// IDistribucionCuposManager.cs
public interface IDistribucionCuposManager
{
    DistribucionResponseDto Calcular(DistribucionCuposRequestDto request);
    DistribucionMultiDiaResponseDto CalcularMultiDia(DistribucionMultiDiaRequestDto request);
}

// IConfiguracionDistribucionManager.cs
public interface IConfiguracionDistribucionManager
{
    ConfiguracionDistribucionDto ObtenerConfiguracion();
    void GuardarConfiguracion(ConfiguracionDistribucionDto dto);
}
```

---

## FASE 3: Business Logic (`Molinos.DataAgro.Business/Managers`)

### 3A. `SapArchivoParserManager`

**Constantes** (del POC HTML, trasladadas a C#):

```csharp
private const string TARGET_CTR = "Planta San Lorenzo";
private const int CUPO_KG = 30000;
private static readonly HashSet<string> VALID_COS = new HashSet<string> { "23-24", "24-25", "25-26" };
private static readonly HashSet<string> EXCL_DESC = new HashSet<string> { "MP-Venta granos", "" };
private static readonly Dictionary<string, (int rank, string label)> PRIO_MAP =
    new Dictionary<string, (int, string)>
    {
        { "MP-Fason",            (1, "MP-Fason") },
        { "MP-Prest/Devolución", (2, "MP-Prest/Dev") },
        { "Fijo",                (3, "Fijo") },
        { "Contrato Hijo",       (4, "Contrato Hijo") },
        { "A Fijar",             (5, "A Fijar") },
        // ... completar según HTML
    };
```

**Lógica de parseo:**

- **Detección de encoding**: `new StreamReader(stream, detectEncodingFromByteOrderMarks: true)` maneja UTF-16LE BOM automáticamente
- **TSV/TXT**: `reader.ReadLine().Split('\t')`
- **XLS**: `ExcelReaderFactory.CreateBinaryReader(stream)` (ExcelDataReader v2, ya instalado)
- **XLSX**: `ExcelReaderFactory.CreateOpenXmlReader(stream)` (ExcelDataReader v2, ya instalado)
- Encontrar fila de encabezados (contiene columna `Tipo`) — si no existe → lanzar `InvalidOperationException("No se encontró la fila de encabezados esperada")`
- Por cada fila:
  - Si `Tipo == "CCPP"` y `Centro == TARGET_CTR` → acumular en `ccppByCuitMat[cuit|material]`
  - Si `Tipo != "CTO"` o `Centro != TARGET_CTR` → saltar
  - Si `descCl` en `EXCL_DESC` → saltar
  - Si cosecha no está en `VALID_COS` → saltar
  - Calcular `opType`, `opLabel`, `isSust`, `rank` según PRIO_MAP
  - Agregar a lista de contratos

### 3B. `DistribucionCuposManager`

**`Calcular()`:**

1. **Filtros previos**: aplicar `ExcluirFason`, `ExcluirAgenteCompra`, filtros de fecha si `HabilitarFiltroFechas`
2. **Rank**: ordenar por `PRIO_MAP.rank` ASC, luego por `priceRank` DESC (si aplica), luego por `kgEfectivo` DESC
3. **Por material** (iterar `limitesPorMaterial`):
   - Límite = 0 o ausente → todos `estado = "sin_tope"`, 0 cupos
   - `topeCuit = floor(limite * cuitMaxPct)` (min 1)
   - Si `aplicarCuotaOperador`: `topeOperador[opType] = floor(limite * cuota% / 100)`
   - Si `aplicarCuotaClase`: `topeClase[clase] = floor(limite * cuota% / 100)`
   - Para cada contrato (en orden de rank):
     - `ccppKg = ccppByCuitMat[cuit|material] ?? 0`
     - `kgEfectivo = max(0, kg - ccppKg)`
     - `cuposNecesarios = kgEfectivo > 0 ? ceil(kgEfectivo / CUPO_KG) : 0`
     - `disponible = min(cuposRestantesMaterial, topeCuitRestante, topeOpRestante, topeClaseRestante)`
     - `cuposAsignados = min(cuposNecesarios, disponible)`
     - `estado = cuposNecesarios == 0 ? "sin_cupos" : cuposAsignados == 0 ? "sin_cupos" : cuposAsignados < cuposNecesarios ? "parcial" : "completo"`
     - Descontar de todos los contadores
4. Generar `sap[]`: una fila por `(fecha, numeroSAP, cuposAsignados)` donde cuposAsignados > 0

**`CalcularMultiDia()`:**

1. Validar: `fechas.Count` entre 2 y 7 — si no: `throw new ArgumentException("El máximo de días permitido es 7")`
2. Validar: cada fecha en `fechas` tiene entrada en `limitesPorDia` — si no: `throw new InvalidOperationException($"La fecha {fecha} no tiene límites definidos")`
3. Descontar CCPP **una sola vez** antes de iterar
4. Si `distribuirUniforme`:
   - Pre-calcular `cuposTotalesNecesarios` por contrato
   - Distribuir cupos en partes iguales entre días disponibles (round-robin si no divide exacto)
5. Iterar días en orden → llamar lógica de `Calcular()` con cupos restantes del contrato
6. Acumular `sapConsolidado`, calcular `resumenGlobal`

### 3C. `ConfiguracionDistribucionManager`

```
ObtenerConfiguracion():
  var entity = repositorio.ObtenerPrimero<ConfiguracionDistribucionPlanta>(x => x.PlantaCodigo == "SL")
  Si null → retornar ConfiguracionDistribucionDto con defaults (cupoKg=30000, cuitMaxPct=0.30, cosechas 23-24/24-25/25-26)
  Deserializar JSON fields con Newtonsoft.Json → retornar DTO

GuardarConfiguracion(dto):
  Validar cuitMaxPct ∈ [0.0, 1.0]
  Validar sum(cuotasPorOperador) <= 100
  Validar sum(cuotasPorClase) <= 100
  Upsert: obtener entity existente o crear nueva con Id=1
  Serializar dicts a JSON
  repositorio.GuardarCambios()
```

---

## FASE 4: Repository (`Molinos.DataAgro.Repository`)

### Modificar `DataAgroDbContext.cs`

```csharp
public DbSet<ConfiguracionDistribucionPlanta> ConfiguracionesDistribucionPlanta { get; set; }
```

No se necesita nuevo repositorio — se usa el `IRepositorio` genérico existente.

### Script SQL (`Base de Datos/Scripts/`)

```sql
CREATE TABLE ConfiguracionesDistribucionPlanta (
    Id                   INT          NOT NULL DEFAULT 1 PRIMARY KEY,
    PlantaCodigo         VARCHAR(10)  NOT NULL,
    PlantaNombre         VARCHAR(200) NULL,
    CupoKg               INT          NOT NULL DEFAULT 30000,
    CuitMaxPct           DECIMAL(5,4) NOT NULL DEFAULT 0.30,
    CosechasValidas      NVARCHAR(500) NULL,
    ClasesExcluidas      NVARCHAR(500) NULL,
    LimitesPredeterminados NVARCHAR(MAX) NULL,  -- JSON dict
    PreciosReferencia    NVARCHAR(MAX) NULL,    -- JSON dict
    CuotasPorOperador    NVARCHAR(MAX) NULL,    -- JSON dict
    CuotasPorClase       NVARCHAR(MAX) NULL,    -- JSON dict
    UltimaActualizacion  DATETIME     NULL
);

INSERT INTO ConfiguracionesDistribucionPlanta
    (Id, PlantaCodigo, PlantaNombre, CupoKg, CuitMaxPct,
     CosechasValidas, ClasesExcluidas, UltimaActualizacion)
VALUES
    (1, 'SL', 'Planta San Lorenzo', 30000, 0.30,
     '23-24,24-25,25-26', 'MP-Venta granos', GETDATE());
```

---

## FASE 5: Controllers (`WebDataAgro`)

### Prerequisito: Habilitar Attribute Routing

En `App_Start/RouteConfig.cs`, agregar **antes** del route Default:

```csharp
routes.MapMvcAttributeRoutes();  // ← agregar esta línea
routes.MapRoute(name: "Default", ...);
```

> **No se necesita CORS.** La Vista Razor se sirve desde la misma app; todas las llamadas `fetch()` son mismo origen.

### 5A. `DistribucionController.cs` (sirve la Vista)

```
Namespace:  WebDataAgro.Controllers
Base:       System.Web.Mvc.Controller
Auth:       [Autorizacion] (atributo de autorización existente en el proyecto)
```

```csharp
[Autorizacion]
public class DistribucionController : Controller
{
    [HttpGet]
    public ActionResult Index()
    {
        return View();
    }
}
```

### 5A-bis. Entrada en el menú horizontal — dentro de "Cupos"

> **El menú horizontal superior está *hardcodeado en Razor*, no en una tabla SQL.** Se verificó `WebDataAgro/Views/Shared/_Layout.cshtml` (líneas 641-696): cada dropdown y cada `<li>` es markup estático protegido por `@if (PermisosHelper.Is(PermisosDataAgro.<Permiso>))`. No existe una tabla de menú/opciones en base de datos para este módulo — agregar una entrada implica editar directamente este `.cshtml` (requiere build + deploy, no es data-driven).

En `WebDataAgro/Views/Shared/_Layout.cshtml`, dentro del `<ul class="dropdown-menu opacidad" id="cupos">` existente (el mismo dropdown que ya contiene "Sugerencia de Cupos", "Cupos", "Disponibilidad de Cupos", "Alta Masiva de Sugerencias"), agregar un nuevo `<li>` para "Distribuidor Cupos":

```cshtml
<li class="dropdown">
    <a onclick="return false" id="cupos-a" class="dropdown-toggle nav-text6 padding-left nav-no-padding background-hover" data-toggle="dropdown" href="#">Cupos <span class="caret"></span></a>
    <ul class="dropdown-menu opacidad" id="cupos">
        @* ... entradas existentes (Sugerencia de Cupos, Cupos, Disponibilidad, Cupos No Propios) ... *@

        <li>
            <a href="~/Cupo/AltaMasivaCupos" class="nav-link misreporteslink">
                <i class="icon-bar-chart"></i> Alta Masiva de Sugerencias
            </a>
        </li>

        @* ── Nueva entrada ── *@
        @if (PermisosHelper.Is(PermisosDataAgro.SugerenciaDeCupos))
        {
            <li>
                <a href="~/Distribucion" class="nav-link misreporteslink">
                    <i class="icon-bar-chart"></i> Distribuidor Cupos
                </a>
            </li>
        }
    </ul>
</li>
```

**Sin permiso nuevo — se reutiliza `SugerenciaDeCupos` (708).** Se decidió NO crear un permiso dedicado (`DistribuidorCuposSL`) porque:
- El módulo es conceptualmente una variante de "Sugerencia de Cupos" (genera sugerencias de distribución de cupos), por lo que los mismos usuarios/roles que hoy acceden a `SugerenciaCupo` son los destinatarios naturales de esta nueva pantalla.
- Evita el costo de dar de alta el permiso en el ABM de Roles/Permisos, asignarlo a los roles de Planta San Lorenzo, y coordinar ese cambio de datos en QA/PROD antes del deploy — reduce superficie de cambio y riesgo.
- El controller `DistribucionController` (y sus endpoints API) se protegen con `[Autorizacion(PermisosDataAgro.SugerenciaDeCupos)]`, igual que hoy hace `SugerenciaCupoController` (mismo criterio ya vigente).

> El `if` externo del dropdown "Cupos" (línea `@if (PermisosHelper.Is(PermisosDataAgro.VisualizarCupos) || PermisosHelper.Is(PermisosDataAgro.SugerenciaDeCupos) || ...)`) **no requiere cambios**, ya que `SugerenciaDeCupos` ya forma parte de esa condición OR.
>
> Si en el futuro el negocio pide un permiso independiente (para restringir el acceso a "Distribuidor Cupos" sin dar también "Sugerencia de Cupos"), se puede agregar como tarea separada sin impacto en el resto del plan.

### 5B. `ContratosApiController.cs`

```
Namespace:    WebDataAgro.Controllers
Base:         System.Web.Mvc.Controller
Route prefix: [RoutePrefix("api/contratos")]
Auth:         [Autorizacion]
```

**Acción `Importar`:**
- `[HttpPost, Route("importar")]`
- Parámetros: `HttpPostedFileBase file, string fecha = null`
- Validar extensión: `.xls`, `.xlsx`, `.txt`, `.tsv` → 400 si inválida
- Validar tamaño: `file.ContentLength <= 10 * 1024 * 1024` → 400 si supera
- Parsear fecha o usar `DateTime.Today`
- Llamar `_sapParserManager.ParsearArchivo(file.InputStream, extension, fechaParsed)`
- `catch (InvalidOperationException)` → HTTP 422
- Return: `Json(resultado, JsonRequestBehavior.AllowGet)`

### 5C. `DistribucionApiController.cs`

```
Route prefix: [RoutePrefix("api/distribucion")]
Auth:         [Autorizacion]
```

Helper privado para leer JSON body (MVC 5 no bindea body JSON automáticamente en POST):

```csharp
private T LeerJsonBody<T>()
{
    Request.InputStream.Seek(0, SeekOrigin.Begin);
    var json = new StreamReader(Request.InputStream).ReadToEnd();
    return JsonConvert.DeserializeObject<T>(json);
}
```

**Acción `Calcular`:** `[HttpPost, Route("calcular")]`
- Leer body: `var req = LeerJsonBody<DistribucionCuposRequestDto>()`
- Validar que `LimitesPorMaterial` no tenga valores negativos → 400
- Llamar `_distribucionManager.Calcular(req)` → `Json(resultado)`

**Acción `CalcularMultiDia`:** `[HttpPost, Route("calcular-multi-dia")]`
- Leer body: `var req = LeerJsonBody<DistribucionMultiDiaRequestDto>()`
- `catch (ArgumentException)` → 400 con mensaje
- `catch (InvalidOperationException)` → 422 con mensaje
- Llamar `_distribucionManager.CalcularMultiDia(req)` → `Json(resultado)`

### 5D. `ConfiguracionDistribucionController.cs`

```
Route prefix: [RoutePrefix("api/configuracion")]
Auth:         [Autorizacion]
```

- `[HttpGet, Route("distribucion")]` → `Json(_configManager.ObtenerConfiguracion(), JsonRequestBehavior.AllowGet)`
- `[HttpPut, Route("distribucion")]`
  - Leer body con `LeerJsonBody<ConfiguracionDistribucionDto>()`
  - `catch (ArgumentException ex)` → 400 con `{ error: ex.Message }`
  - Llamar `_configManager.GuardarConfiguracion(dto)` → 200 OK

---

## FASE 6: Vista Razor (`WebDataAgro`)

**Objetivo:** Convertir `distribuidor_cupos.html` (POC standalone) en una Vista Razor integrada en `WebDataAgro`, eliminando toda lógica de cálculo del frontend.

### 6A. CSS → `Content/distribucion-cupos.css`

**Diagnóstico de la maqueta (`distribuidor_cupos.html`):**
- Un bloque `<style>` de ~185 líneas con variables `:root` (`--g`, `--gl`, `--gd`, `--gold`, `--org`, `--teal`, `--blu`, `--pur`, `--bg`, `--tx`, etc.) y ~90 reglas de clase (`.hdr`, `.card`, `.btn`, `.tabs`, `.bp`, `.bs`, `.sc`, etc.)
- **~102 atributos `style="..."` inline**, repartidos en dos grupos:
  - **Markup estático** (~40 casos): spacing puntual (`margin-top`, `padding`), layout ad-hoc (`display:flex`, `display:grid`), y color puntual (`color:var(--org)`)
  - **HTML generado dinámicamente en JS** (~60 casos, dentro de template strings de `innerHTML` en funciones como `buildLimGrid()`, `buildStatsUI()`, `renderResults()`): estilos inyectados por JS al construir filas de tablas y tarjetas
- Un **fondo de header en base64** (imagen JPEG ~30 KB) embebido directamente en la regla `.hdr-bg{background-image:url('data:image/jpeg;base64,...')}` — anti-patrón para el repositorio (hincha el CSS, no cachea como archivo estático, ensucia diffs de git)

**Plan de conversión (sin cambiar el aspecto visual del POC):**

1. **Extraer el `<style>` completo** a `WebDataAgro/Content/distribucion-cupos.css`, tal cual, como base.
2. **Prefijar las variables CSS** para evitar colisión con cualquier variable global futura del sitio: `--g` → `--dc-g`, `--gold` → `--dc-gold`, etc. (la vista usa `Layout = null`, por lo que hoy no hay colisión real con el layout global, pero prefijar es una buena práctica de aislamiento a bajo costo).
3. **Extraer la imagen de fondo del header**: decodificar el base64 y guardarlo como `WebDataAgro/Content/Images/distribucion-cupos-header-bg.jpg`, referenciado como `background-image:url('/Content/Images/distribucion-cupos-header-bg.jpg')`. Reduce el CSS de ~35 KB a ~2 KB y permite cacheo HTTP normal del navegador.
4. **Convertir los inline styles estáticos del markup a clases utilitarias** agregadas a `distribucion-cupos.css`, siguiendo el patrón de nombres cortos ya usado en el POC (`.mt-*`, `.pt-*`, `.d-flex`, `.gap-*`):

   | Inline style repetido en el POC | Clase utilitaria nueva |
   |---|---|
   | `margin-top:14px` / `margin-top:12px` / `margin-top:10px` / `margin-top:8px` / `margin-top:20px` / `margin-top:4px` / `margin-top:6px` | `.mt-14`, `.mt-12`, `.mt-10`, `.mt-8`, `.mt-20`, `.mt-4`, `.mt-6` |
   | `margin-bottom:0` / `margin-bottom:10px` | `.mb-0`, `.mb-10` |
   | `display:flex;align-items:center;gap:12px` (y variantes de `gap`) | `.d-flex-center`, con modificador `.gap-8`/`.gap-12`/`.gap-14` |
   | `display:grid;grid-template-columns:1fr 1fr;gap:16px` | `.grid-2col` |
   | `text-align:center` | `.text-center` (ya estándar en Bootstrap, reutilizable si la vista llegara a cargar Bootstrap) |
   | `color:var(--dc-org)` / `color:#7B1FA2` puntual en `<strong>`/`<span>` | `.txt-org`, `.txt-purple` |
   | `font-size:11px` / `font-size:12px` / `font-size:13px` sueltos | `.fs-11`, `.fs-12`, `.fs-13` |
   | `padding:0` en `.cb` puntual | `.p-0` |

   El markup pasa de `<div style="margin-top:14px;padding-top:12px;border-top:1px solid var(--border)">` a `<div class="mt-14 pt-12 bt-border">`.

5. **Convertir los inline styles generados dinámicamente en JS** (dentro de `distribucion-cupos.js`, ver FASE 6B): en vez de que `buildLimGrid()`, `buildStatsUI()`, `renderResults()`, etc. arme `style="text-align:center;width:100%"` como string concatenado, esas funciones deben emitir `class="text-center w-100"` usando las mismas clases utilitarias del punto 4. Esto evita duplicar CSS embebido en JS y mantiene una única fuente de verdad de estilos.
6. **No se reutilizan clases de Bootstrap/Kendo existentes en `WebDataAgro`** porque la vista usa `Layout = null` (pantalla completa sin el layout global, ver 6C) y por lo tanto no carga `bootstrap.min.css` ni `kendo.*.css`. Esto es intencional: evita colisiones de nombres genéricos ya usados por Bootstrap (`.card`, `.btn`, `.badge`) con las clases homónimas del POC, que tienen estilos propios distintos. Si en el futuro se decide envolver esta vista dentro del layout estándar de DataAgro, este punto debe revisarse (renombrar clases o aislarlas con un contenedor `.distribucion-cupos-scope`).

Registrar bundle en `App_Start/BundleConfig.cs`:

```csharp
bundles.Add(new StyleBundle("~/Content/distribucion-cupos")
    .Include("~/Content/distribucion-cupos.css"));
```

### 6B. JavaScript → `Scripts/Distribucion/distribucion-cupos.js`

Extraer el bloque `<script>` del HTML a `WebDataAgro/Scripts/Distribucion/distribucion-cupos.js`.

**Eliminar del JS** (lógica trasladada al backend):

| Función JS del POC | Reemplazada por |
|---|---|
| `parseFile(buf)` | `SapArchivoParserManager.ParsearArchivo()` |
| `processRows(rows, today)` — filtros, PRIO_MAP, CCPP, opType | `SapArchivoParserManager.ParsearArchivo()` |
| `computeStats(contracts, dates)` | Incluido en `ImportacionSapResultadoDto.Estadisticas` |
| `distribute(contracts, limits, ccppByCuitMat, ...)` | `DistribucionCuposManager.Calcular()` |
| Lógica multi-día interna (`distributeMultiDay`) | `DistribucionCuposManager.CalcularMultiDia()` |
| Constantes `PRIO_MAP`, `VALID_COS`, `EXCL_DESC`, `TARGET_CTR`, `CUPO_KG` | Definidas en C# en los Managers |

**Conservar en JS** (solo presentación/UI):

| Función JS | Responsabilidad |
|---|---|
| `handleFile(file)` | Upload del archivo → llama `API_IMPORTAR` |
| `applyParsed(data)` | Aplica el response de US-01 al estado local |
| `buildAll()` | Reconstruye la UI de límites, stats, precios |
| `buildLimGrid()` | Grilla de topes diarios por material |
| `buildStatsUI()` | Tablas de estadísticas (operador, clase, sustentables) |
| `buildPctInputs()` | Inputs de cuotas % por operador/clase |
| `buildDateFilterUI()` | UI de filtro de fechas |
| `buildPricesUI()` | UI de precios de referencia |
| `runDistribution()` | Arma el request y llama `API_CALCULAR` o `API_CALCULAR_MD` |
| `renderResults(data)` | Renderiza tablas de resultados con el response del backend |
| `exportExcel()` | SheetJS — sin cambios |
| `loadConfig()` / `saveConfig()` | Llaman `API_CONFIG_GET` / `API_CONFIG_PUT` |

Registrar bundle en `App_Start/BundleConfig.cs`:

```csharp
bundles.Add(new ScriptBundle("~/bundles/distribucion-cupos")
    .Include("~/Scripts/Distribucion/distribucion-cupos.js"));
```

### 6C. Vista → `Views/Distribucion/Index.cshtml`

```cshtml
@{
    Layout = null; // pantalla completa sin nav (herramienta operativa)
    ViewBag.Title = "Distribuidor de Cupos – Planta San Lorenzo";
}
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>@ViewBag.Title</title>
    @Styles.Render("~/Content/distribucion-cupos")
</head>
<body>

    <!-- ── Estructura HTML del POC (sin cambios en el markup) ── -->

    <!-- Las URLs de los endpoints se inyectan via Razor para evitar hard-coding -->
    <script>
        var API_IMPORTAR    = '@Url.Action("Importar",              "ContratosApi")';
        var API_CALCULAR    = '@Url.Action("Calcular",              "DistribucionApi")';
        var API_CALCULAR_MD = '@Url.Action("CalcularMultiDia",      "DistribucionApi")';
        var API_CONFIG_GET  = '@Url.Action("ObtenerConfiguracion",  "ConfiguracionDistribucion")';
        var API_CONFIG_PUT  = '@Url.Action("GuardarConfiguracion",  "ConfiguracionDistribucion")';
    </script>

    @Scripts.Render("~/bundles/jquery")
    @Scripts.Render("~/bundles/distribucion-cupos")
    <!-- SheetJS para export Excel (CDN o local) -->
    <script src="https://cdn.sheetjs.com/xlsx-latest/package/dist/xlsx.full.min.js"></script>
</body>
</html>
```

### 6D. Calls a la API en el JS

```javascript
// US-01: subir archivo SAP al backend para parseo
async function handleFile(file) {
    showLoading();
    const fd = new FormData();
    fd.append('file', file);
    fd.append('fecha', document.getElementById('today-date').value);
    const res = await fetch(API_IMPORTAR, { method: 'POST', body: fd });
    hideLoading();
    if (!res.ok) { showError((await res.json()).error); removeFile(); return; }
    const data = await res.json();
    // data contiene: contratos, ccppByCuitMat, ccppByMat, materiales, estadisticas, errores
    applyParsed(data);
    buildAll();
    ['card-limits','multiday-section'].forEach(id =>
        document.getElementById(id).classList.remove('hidden'));
}

// US-02 / US-03: distribución (1 día o multi-día)
async function runDistribution() {
    const isMulti = document.getElementById('chk-multiday').checked;
    const url  = isMulti ? API_CALCULAR_MD : API_CALCULAR;
    const body = isMulti ? buildMultiDayRequest() : buildSingleDayRequest();
    showLoading();
    const res = await fetch(url, {
        method:  'POST',
        headers: { 'Content-Type': 'application/json' },
        body:    JSON.stringify(body)
    });
    hideLoading();
    const data = await res.json();
    if (!res.ok) { showError(data.error); return; }
    renderResults(data);
    // data.sap / data.sapConsolidado alimentan SheetJS sin cambios
}

// US-05: cargar config al iniciar
async function loadConfig() {
    const res = await fetch(API_CONFIG_GET);
    if (res.ok) applyConfigToUI(await res.json());
}

// US-05: guardar config
async function saveConfig() {
    const res = await fetch(API_CONFIG_PUT, {
        method:  'PUT',
        headers: { 'Content-Type': 'application/json' },
        body:    JSON.stringify(buildConfigFromUI())
    });
    if (!res.ok) showError((await res.json()).error);
    else showSuccess('Configuración guardada');
}
```

### 6E. Route para la Vista

El route Default existente ya cubre `GET /Distribucion` → `DistribucionController.Index()`. No se requiere route adicional.

---

## FASE 7: Unit Tests (`Molinos.DataAgro.Test`)

### `SapArchivoParserManagerTests.cs`
- ✅ Parsear TSV UTF-16LE → retorna contratos con campos correctos
- ✅ Filtrar solo `Planta San Lorenzo` y `Tipo == "CTO"`
- ✅ Excluir `MP-Venta granos`
- ✅ Calcular balance CCPP agrupado por `CUIT|Material`
- ✅ Cosecha fuera de `VALID_COS` → contrato no incluido
- ✅ Archivo sin columna `Tipo` → lanza `InvalidOperationException`

### `DistribucionCuposManagerTests.cs`
- ✅ Ningún CUIT recibe más del 30% de cupos por material
- ✅ `kgEfectivo = kg - ccppDescontado`
- ✅ `cuposNecesarios = ceil(kgEfectivo / 30000)`
- ✅ `aplicarCuotaOperador=true` → cupos por tipo no superan cuota
- ✅ Material con límite 0 → todos `estado = "sin_tope"`
- ✅ Multi-día: CCPP descontado una sola vez
- ✅ Multi-día `distribuirUniforme=true` → cupos distribuidos uniformemente
- ✅ Multi-día > 7 fechas → lanza `ArgumentException`
- ✅ Fecha sin límites en multi-día → lanza `InvalidOperationException`

### `ConfiguracionDistribucionManagerTests.cs`
- ✅ `cuitMaxPct = 1.5` → falla validación con mensaje claro
- ✅ Cuotas por operador suman 110% → falla validación
- ✅ Sin registro en DB → retorna defaults (`cupoKg=30000, cuitMaxPct=0.30`)
- ✅ Guardar → `UltimaActualizacion` se actualiza

---

## FASE 8: US-06 — Procesar resultado en DataAgro (`AltaMasivaSugerenciaCuposV2`)

### Descripción funcional

Junto al botón **"Exportar Excel"** aparece un nuevo botón **"⚡ Procesar en DataAgro"**. Al pulsarlo, el resultado de la distribución (`sap` o `sapConsolidado`) se envía al backend, que lo convierte en el `DataSet` que espera `AltaMasivaSugerenciaCuposV2` y ejecuta el algoritmo de alta masiva de sugerencias de cupos ya existente en la aplicación.

### Flujo

```
Usuario pulsa "⚡ Procesar en DataAgro"
  │
  ▼
JS → POST /api/distribucion/procesar-en-dataagro
     body: { sap: [ { fechaSugerida, cantidadDeCupos, contratoSAP } ] }
  │
  ▼
DistribucionApiController.ProcesarEnDataAgro()
  ├─ Construir DataSet en memoria con 3 columnas:
  │    ContratoSAP      → string  (ej: "4500012345")
  │    FechaSugerida    → double  (OADate — ver nota ⚠️)
  │    CantidadDeCupos  → int
  │
  ├─ _cupoManager.AltaMasivaSugerenciaCuposV2(dsExcel)
  │    (usa ICupoManager ya registrado en Autofac)
  │
  └─ Return: Json({ resultado: true, resume: [...] })
```

> ⚠️ **Conversión de fecha a OADate**  
> `AltaMasivaSugerenciaCuposV2` lee la fecha así:
> ```csharp
> FechaSugerida = DateTime.FromOADate(double.Parse(row["FechaSugerida"].ToString()))
> ```
> Por lo tanto, al construir el `DataSet` desde el JSON ("dd/MM/yyyy"), el controller **debe convertir**:
> ```csharp
> var fecha = DateTime.ParseExact(fila.FechaSugerida, "dd/MM/yyyy", CultureInfo.InvariantCulture);
> row["FechaSugerida"] = fecha.ToOADate();  // ← double
> ```

### Endpoint

**`POST /api/distribucion/procesar-en-dataagro`**

- **Request body (JSON):**
  ```json
  {
    "sap": [
      { "fechaSugerida": "11/06/2026", "cantidadDeCupos": 6, "contratoSAP": "4500012345" },
      { "fechaSugerida": "12/06/2026", "cantidadDeCupos": 4, "contratoSAP": "4500067890" }
    ]
  }
  ```
- **Response (200):**
  ```json
  { "resultado": true, "resume": [], "mensaje": "Cupos procesados correctamente" }
  ```
- **Response (400):**
  ```json
  { "resultado": false, "error": "El array 'sap' no puede estar vacío" }
  ```
- **Response (500):**
  ```json
  { "resultado": false, "error": "Error al procesar: <mensaje>" }
  ```

### Implementación en `DistribucionApiController`

```csharp
// Requiere inyectar ICupoManager en el constructor del controller
[HttpPost, Route("procesar-en-dataagro")]
public ActionResult ProcesarEnDataAgro()
{
    try
    {
        var req = LeerJsonBody<ProcesarEnDataAgroRequestDto>();

        if (req?.Sap == null || !req.Sap.Any())
            return Json(new { resultado = false, error = "El array 'sap' no puede estar vacío" });

        // Construir DataSet con el formato que espera AltaMasivaSugerenciaCuposV2
        var dt = new DataTable();
        dt.Columns.Add("ContratoSAP",      typeof(string));
        dt.Columns.Add("FechaSugerida",    typeof(double));  // OADate
        dt.Columns.Add("CantidadDeCupos",  typeof(int));

        foreach (var fila in req.Sap)
        {
            var fecha = DateTime.ParseExact(fila.FechaSugerida, "dd/MM/yyyy",
                            System.Globalization.CultureInfo.InvariantCulture);
            dt.Rows.Add(fila.ContratoSAP, fecha.ToOADate(), fila.CantidadDeCupos);
        }

        var ds = new DataSet();
        ds.Tables.Add(dt);

        var resume = _cupoManager.AltaMasivaSugerenciaCuposV2(ds);

        return Json(new { resultado = true, resume, mensaje = "Cupos procesados correctamente" });
    }
    catch (Exception ex)
    {
        logger.Error(ex, "ProcesarEnDataAgro error");
        return Json(new { resultado = false, error = "Error al procesar: " + ex.Message });
    }
}
```

### DTO nuevo: `ProcesarEnDataAgroRequestDto`

```csharp
// En Entities/Dto/Distribucion/
public class ProcesarEnDataAgroRequestDto
{
    public List<FilaSapDto> Sap { get; set; }
}
```

> `FilaSapDto` ya existe (creada en Fase 1). No se necesita DTO nuevo de respuesta — se reutiliza el objeto anónimo de respuesta.

### Cambios en la UI (JS)

```javascript
// Botón nuevo en la barra de acciones junto a "Exportar Excel"
async function procesarEnDataAgro() {
    const sapData = S.lastResult?.sap || S.lastResult?.sapConsolidado || [];
    if (!sapData.length) { showError('No hay resultado para procesar.'); return; }

    setBtnProcesar('loading');
    const res = await fetch(API_PROCESAR_DATAAGRO, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ sap: sapData })
    });
    const data = await res.json();
    setBtnProcesar('idle');

    if (!res.ok || !data.resultado) {
        showError(data.error || 'Error al procesar en DataAgro');
    } else {
        showSuccess('✅ Cupos enviados a DataAgro correctamente');
    }
}
```

La URL `API_PROCESAR_DATAAGRO` se inyecta via Razor igual que las otras:
```cshtml
var API_PROCESAR_DATAAGRO = '@Url.Action("ProcesarEnDataAgro", "DistribucionApi")';
```

### Archivo nuevo en la vista: botón HTML

```html
<!-- Barra de acciones — junto al botón de Export Excel existente -->
<button id="btn-procesar-dataagro" class="btn btn-primary" onclick="procesarEnDataAgro()" disabled>
    ⚡ Procesar en DataAgro
</button>
```

El botón se habilita solo cuando hay resultado disponible (`S.lastResult` no es null).

### Test: `DistribucionApiControllerTests` (o `CupoManagerIntegrationTests`)

- ✅ `sap` vacío → retorna 400 con mensaje
- ✅ Fechas en formato "dd/MM/yyyy" se convierten correctamente a OADate
- ✅ `ContratoSAP` se almacena como string en el DataSet
- ✅ Llamada a `AltaMasivaSugerenciaCuposV2` con DataSet correcto → retorna resultado

---

## Decisión: US-04 Export Excel — Frontend vs Backend

| Criterio | Backend (EPPlus 4.x) | **Frontend (SheetJS)** ✅ |
|---|---|---|
| Código backend nuevo | Sí (endpoint + streaming + MIME) | No |
| Dependencias NuGet | EPPlus 4.x (API distinta a 5.x) | SheetJS CDN (ya en POC) |
| Formato texto columna SAP | Requiere config EPPlus específica | Ya implementado y validado |
| Mantenimiento | C# backend | JS en el bundle |
| Riesgo de regresión | Medio | Bajo (ya funciona) |

**Decisión**: Usar SheetJS en el frontend. Si en el futuro se necesita generación server-side (emails automáticos, batch), se agrega el endpoint con EPPlus 4.x.

---

## NuGet — Sin instalaciones nuevas

| Paquete | Versión | Proyecto | Estado |
|---|---|---|---|
| `ExcelDataReader` | 2.1.2.3 | Business | ✅ Ya instalado |
| `Newtonsoft.Json` | 12.0.3 | Business + WebDataAgro | ✅ Ya instalado |
| EPPlus | 4.1.0 | Business | No se usa en este plan |

---

## Resumen de Archivos a Crear/Modificar

### Nuevos

| Proyecto | Carpeta | Archivo |
|---|---|---|
| Entities | `Dto/Distribucion/` | `ContratoSapImportadoDto.cs` |
| Entities | `Dto/Distribucion/` | `EstadisticasImportacionDto.cs` |
| Entities | `Dto/Distribucion/` | `ImportacionSapResultadoDto.cs` |
| Entities | `Dto/Distribucion/` | `DistribucionCuposRequestDto.cs` |
| Entities | `Dto/Distribucion/` | `DistribucionConfigDto.cs` |
| Entities | `Dto/Distribucion/` | `ResultadoContratoDistribucionDto.cs` |
| Entities | `Dto/Distribucion/` | `FilaSapDto.cs` |
| Entities | `Dto/Distribucion/` | `ResumenDistribucionDto.cs` |
| Entities | `Dto/Distribucion/` | `DistribucionResponseDto.cs` |
| Entities | `Dto/Distribucion/` | `LimiteDiaDto.cs` |
| Entities | `Dto/Distribucion/` | `DistribucionMultiDiaRequestDto.cs` |
| Entities | `Dto/Distribucion/` | `ResultadoDiaDto.cs` |
| Entities | `Dto/Distribucion/` | `ResumenGlobalDto.cs` |
| Entities | `Dto/Distribucion/` | `DistribucionMultiDiaResponseDto.cs` |
| Entities | `Dto/Distribucion/` | `ConfiguracionDistribucionDto.cs` |
| Entities | `Dto/Distribucion/` | `ProcesarEnDataAgroRequestDto.cs` |
| Interfaces | `Managers/` | `ISapArchivoParserManager.cs` |
| Interfaces | `Managers/` | `IDistribucionCuposManager.cs` |
| Interfaces | `Managers/` | `IConfiguracionDistribucionManager.cs` |
| Business | `Managers/` | `SapArchivoParserManager.cs` |
| Business | `Managers/` | `DistribucionCuposManager.cs` |
| Business | `Managers/` | `ConfiguracionDistribucionManager.cs` |
| WebDataAgro | `Controllers/` | `DistribucionController.cs` (sirve la Vista) |
| WebDataAgro | `Controllers/` | `ContratosApiController.cs` |
| WebDataAgro | `Controllers/` | `DistribucionApiController.cs` (incluye `ProcesarEnDataAgro`) |
| WebDataAgro | `Controllers/` | `ConfiguracionDistribucionController.cs` |
| WebDataAgro | `Views/Distribucion/` | `Index.cshtml` |
| WebDataAgro | `Content/` | `distribucion-cupos.css` (variables prefijadas `--dc-*` + clases utilitarias, sin base64) |
| WebDataAgro | `Content/Images/` | `distribucion-cupos-header-bg.jpg` (imagen extraída del base64 embebido en el POC) |
| WebDataAgro | `Scripts/Distribucion/` | `distribucion-cupos.js` |
| Base de Datos | `Scripts/` | `ConfiguracionesDistribucionPlanta.sql` |

### Modificados

| Proyecto | Archivo | Cambio |
|---|---|---|
| Repository | `DataAgroDbContext.cs` | Agregar `DbSet<ConfiguracionDistribucionPlanta>` |
| WebDataAgro | `App_Start/RouteConfig.cs` | Agregar `routes.MapMvcAttributeRoutes()` |
| WebDataAgro | `App_Start/BundleConfig.cs` | Registrar bundle CSS y JS de distribucion-cupos |
| WebDataAgro | `Views/Shared/_Layout.cshtml` | Agregar entrada "Distribuidor Cupos" dentro del dropdown "Cupos" (protegida por el permiso existente `SugerenciaDeCupos`, sin permiso nuevo) |

---

## Criterios de Aceptación

- ✅ `GET /Distribucion` sirve la pantalla protegida por `[Autorizacion]`
- ✅ El menú horizontal superior, dentro de "Cupos", muestra la opción "Distribuidor Cupos" para usuarios con el permiso existente `SugerenciaDeCupos` (no se crea un permiso nuevo)
- ✅ `POST /api/contratos/importar` parsea TSV UTF-16LE y retorna contratos + balance CCPP (sin lógica en el browser)
- ✅ `POST /api/distribucion/calcular` respeta tope 30% por CUIT en todos los casos
- ✅ `POST /api/distribucion/calcular-multi-dia` descuenta CCPP una sola vez y distribuye uniformemente
- ✅ `GET/PUT /api/configuracion/distribucion` persiste entre sesiones
- ✅ Export Excel desde el frontend descarga `.xlsx` con `ContratoSAP` como texto (SheetJS)
- ✅ Botón "⚡ Procesar en DataAgro" llama a `AltaMasivaSugerenciaCuposV2` con DataSet bien formado (fechas como OADate)
- ✅ Sin CORS — todos los `fetch()` son mismo origen
- ✅ Las URLs de los endpoints no están hard-coded en el JS: se inyectan via `@Url.Action()`
- ✅ Errores de validación retornan HTTP 400/422 con mensaje descriptivo en JSON
- ✅ Tests unitarios cubren los casos críticos de cada Manager
- ✅ El markup no contiene atributos `style="..."` inline (ni estáticos ni generados por JS): todo el estilo vive en `distribucion-cupos.css` vía clases utilitarias
- ✅ El CSS no contiene imágenes embebidas en base64: la imagen de fondo del header es un archivo estático en `Content/Images/`
- ✅ El resultado visual de la pantalla es idéntico al del POC `distribuidor_cupos.html` (validación visual lado a lado por el Analista Funcional)

---

## Estimación con asistencia de IA (Copilot)

> La estimación está dividida en **tiempo de desarrollo asistido por IA** (generación + review + ajustes) y el **tiempo sin IA** de referencia para comparar el ahorro. Esta revisión incorpora el equipo completo (Dev, QA, Analista Funcional) y los tiempos de despliegue a QA y PROD, que la estimación anterior no contemplaba.

### Supuestos

- El desarrollador conoce el stack (.NET Framework 4.7.2 / MVC 5 / EF6 / Autofac)
- La IA genera el primer borrador de cada archivo; el developer revisa, ajusta y valida
- El POC HTML (`distribuidor_cupos.html`) es fuente de verdad para la lógica de negocio
- La conversión de fechas a OADate se valida con un caso de prueba puntual antes de dar por cerrado US-06
- **Equipo**: 1 Developer, 1 QA, 1 Analista Funcional (part-time, para validación de negocio y aceptación)
- **Despliegues**: DEV es continuo (sin costo adicional); QA y PROD requieren ventana coordinada + script SQL manual (`ConfiguracionesDistribucionPlanta`) + smoke test

---

### 1. Desarrollo (Developer, con IA)

| Fase | Descripción | Sin IA | **Con IA** | Ahorro |
|---|---|---|---|---|
| **1** | 16 DTOs + 1 entity | 3 h | **0.5 h** | 83% |
| **2** | 3 Interfaces | 0.5 h | **0.1 h** | 80% |
| **3A** | `SapArchivoParserManager` (parseo + encoding + XLS/XLSX) | 6 h | **1.5 h** | 75% |
| **3B** | `DistribucionCuposManager` (rank + topes + multi-día) | 8 h | **2.5 h** | 69% |
| **3C** | `ConfiguracionDistribucionManager` (CRUD + validaciones) | 3 h | **0.75 h** | 75% |
| **4** | `DataAgroDbContext` + script SQL + seeding | 1.5 h | **0.5 h** | 67% |
| **5** | 4 controllers + RouteConfig | 4 h | **1 h** | 75% |
| **5-bis** | Entrada de menú "Distribuidor Cupos" en `_Layout.cshtml` (reutiliza permiso existente `SugerenciaDeCupos`, sin ABM de permisos nuevo) | 0.5 h | **0.15 h** | 70% |
| **6A** | Conversión CSS/estilos: prefijar variables, extraer imagen base64, crear clases utilitarias, reemplazar ~102 `style=""` inline (markup + JS) | 3 h | **1 h** | 67% |
| **6B** | Vista Razor + JS (adaptar POC, endpoints vía `@Url.Action`) + BundleConfig | 4 h | **1.5 h** | 63% |
| **7** | 3 clases de tests unitarios (≈ 25 casos) | 5 h | **1.5 h** | 70% |
| **8 — US-06** | Endpoint `procesar-en-dataagro` + botón UI + OADate | 3 h | **0.75 h** | 75% |
| **Fix de bugs propios** | Corrección de hallazgos del Dev antes de pasar a QA | 3 h | **1.5 h** | 50% |
| **Subtotal Desarrollo** | | **44.5 h** | **~13.75 h** | **69%** |

> La IA no reduce el tiempo de "fix de bugs propios" en la misma proporción que la generación inicial, porque el debugging requiere entender comportamiento en runtime, no solo generar código.
> El ítem "5-bis" se simplifica al reutilizar el permiso `SugerenciaDeCupos` ya existente: no requiere alta de permiso ni cambios en el ABM de Roles/Permisos, solo agregar el `<li>` y validar que el `@if` de acceso al dropdown ya lo contempla.
> El ítem "6A" es nuevo respecto a versiones anteriores del plan: cubre convertir la maqueta HTML (con CSS embebido, variables sin prefijo, imagen en base64 y ~102 atributos `style=""` inline, varios generados dinámicamente en JS) a los estándares de `WebDataAgro` (CSS en archivo propio con clases utilitarias, imagen como asset estático). La IA acelera bastante la extracción mecánica de estilos a clases, pero el developer debe validar visualmente que el resultado sea pixel-idéntico al POC.

---

### 2. QA (testing manual + regresión)

| Actividad | Sin IA (asistencia previa) | **Con IA (casos de prueba asistidos)** | Notas |
|---|---|---|---|
| Diseño de casos de prueba (a partir de Gherkin de las US) | 4 h | **1.5 h** | IA arma la matriz de casos desde los criterios de aceptación |
| Ejecución funcional US-01 a US-05 (5 endpoints) | 6 h | **4 h** | Ejecución manual real, la IA no reemplaza el click-testing |
| Ejecución US-06 (Procesar en DataAgro) — caso crítico: validar cupos creados en BD | 3 h | **2 h** | Incluye verificar `Negocio`/`Cupo` en SQL Server tras el alta masiva |
| Prueba de regresión sobre `CupoController` (por reutilizar `AltaMasivaSugerenciaCuposV2`) | 4 h | **3 h** | Riesgo de romper el flujo existente de alta masiva por Excel |
| Reporte de bugs + reverificación (1 ciclo) | 4 h | **2.5 h** | IA ayuda a redactar reportes y reproducir casos |
| **Subtotal QA** | **21 h** | **~13 h** | **38% ahorro** (QA es más manual, menor apalancamiento de IA) |

---

### 3. Analista Funcional (validación de negocio)

| Actividad | Tiempo estimado | Notas |
|---|---|---|
| Revisión de criterios de aceptación vs. implementación (US-01 a US-06) | 3 h | Comparar contra el POC HTML y las historias de usuario |
| Validación de datos reales: correr un archivo SAP real y verificar resultados de distribución | 3 h | Requiere conocimiento de negocio (topes, cuotas, prioridades) |
| Aceptación de UAT (User Acceptance Testing) en QA | 2 h | Firma de conformidad antes de pasar a PROD |
| **Subtotal Analista Funcional** | **8 h** | La IA no reduce este tiempo — es juicio de negocio, no generación de código |

---

### 4. Despliegues

| Actividad | Tiempo | Notas |
|---|---|---|
| Deploy a **DEV** | incluido en desarrollo | Continuo, sin ventana formal |
| Preparación de release notes + script SQL de migración | 1 h | `ConfiguracionesDistribucionPlanta.sql` + checklist |
| Deploy a **QA** (Web Deploy + script SQL + smoke test) | 2 h | Según `Molinos.DataAgro.Build/DeployBat/` |
| Ventana de UAT en QA (ver Analista Funcional arriba) | — | Ya contabilizado |
| Deploy a **PROD** (ventana coordinada, backup previo, rollback plan) | 3 h | Incluye backup de BD, deploy fuera de horario, smoke test post-deploy |
| Monitoreo post-deploy PROD (primeras corridas reales) | 2 h | Seguimiento del primer día de uso real en Planta San Lorenzo |
| **Subtotal Despliegues** | **8 h** | Sin apalancamiento de IA (actividad operativa) |

---

### Resumen Total del Proyecto

| Rol / Actividad | Sin IA | **Con IA** | Ahorro |
|---|---|---|---|
| Desarrollo (Dev) | 44.5 h | **13.75 h** | 69% |
| QA | 21 h | **13 h** | 38% |
| Analista Funcional | 8 h | **8 h** | 0% (no aplica IA) |
| Despliegues (DEV/QA/PROD) | 8 h | **8 h** | 0% (no aplica IA) |
| **Total** | **81.5 h** | **~42.75 h** | **48%** |

> **Nota clave**: el ahorro global (48%) es notablemente menor que el ahorro solo-desarrollo (69%), porque QA, Analista Funcional y Despliegues son actividades humanas/operativas donde la IA aporta poco o nada. Estimar "tiempo con IA" mirando solo la fase de desarrollo (como hacía la versión anterior de este plan) **subestima el esfuerzo real del proyecto en casi el doble**.

---

### Distribución del tiempo total (~42.75 h con IA)

```
Desarrollo (Dev)                 ██████████████░░░░░░░░░░░░░░   13.75 h  (32%)
QA                                ██████████████░░░░░░░░░░░░░░   13.00 h  (30%)
Analista Funcional                █████████░░░░░░░░░░░░░░░░░░░    8.0 h  (19%)
Despliegues (DEV/QA/PROD)          █████████░░░░░░░░░░░░░░░░░░░    8.0 h  (19%)
```

---

### Calendario estimado (dedicación parcial, no full-time)

| Semana | Actividad |
|---|---|
| 1 | Desarrollo Fases 1–5 (DTOs, Interfaces, Managers, Controllers) |
| 1–2 | Desarrollo Fases 6–8 (Vista, tests, US-06) + fix de bugs propios |
| 2 | Deploy a QA + inicio de testing QA |
| 2–3 | Ciclo de QA (funcional + regresión) + corrección de bugs encontrados |
| 3 | Validación del Analista Funcional (UAT) en QA |
| 3–4 | Deploy a PROD + monitoreo post-deploy |

**Duración total estimada: 3 a 4 semanas** con dedicación parcial del equipo (no es trabajo full-time continuo; se solapa con otras tareas del squad).

---

### Notas sobre la estimación

- **Fases 3A y 3B** (desarrollo) son las más intensas porque requieren validar que el algoritmo C# produce exactamente los mismos resultados que el POC JavaScript.
- **US-06** tiene bajo esfuerzo de desarrollo (reutiliza `ICupoManager`) pero **alto riesgo de regresión en QA**, porque toca un flujo de alta masiva ya en producción (`AltaMasivaSugerenciaCuposV2`). Por eso el ítem de "regresión sobre CupoController" en QA pesa 3-4h.
- **QA y Analista Funcional tienen bajo apalancamiento de IA** porque su trabajo es mayormente ejecución manual y juicio de negocio, no generación de artefactos.
- **Despliegues a QA/PROD** siguen el proceso ya documentado en `Molinos.DataAgro.Build/DeployBat/` — no se estima tiempo de aprobación de change management si aplica en la organización (fuera de alcance de este plan).
- Si el equipo puede paralelizar (QA arma casos de prueba mientras el Dev termina fases finales), la duración calendario puede reducirse a ~2.5 semanas.
