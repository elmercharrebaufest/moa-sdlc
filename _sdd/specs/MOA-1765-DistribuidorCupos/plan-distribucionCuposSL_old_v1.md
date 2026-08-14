# Plan: Implementar API REST Distribución de Cupos SL

**TL;DR:** Extender la solución .NET Framework 4.7.2 existente con 3 nuevos controllers REST en `WebDataAgro`, agregar services de parsing y distribución en `Business`, e integrar persistencia en la BD existente. Reutilizar la arquitectura de capas (Entities → Repository → Business → Controllers).

---

## Arquitectura General

```
Flujo de Datos:
┌─ HTML (distribuidor_cupos.html) ─ CORS ─→ WebDataAgro (Controllers REST)
│                                               ↓
│                                    Molinos.DataAgro.Business
│                                    ├─ SapFileParserService
│                                    ├─ DistribucionService
│                                    └─ ConfiguracionService
│                                               ↓
│                                    Molinos.DataAgro.Repository (DAL)
│                                               ↓
│                                    Base de Datos Existente
│                                               ↑
└── AutoMapper (Molinos.DataAgro.Mapping) ←───┘
```

---

## Pasos de Implementación

### FASE 1: Modelos y DTOs (Molinos.DataAgro.Entities)

1. Crear carpeta `/Distribution` con clases:
   - `ContratoSAP` — contiene todos los campos de US-01 response
   - `CcppBalance` — balance CCPP agrupado por CUIT|Material
   - `EstadisticasImport` — stats de importación
   - `DistribucionRequest`, `DistribucionConfig` — request body US-02
   - `ResultadoContrato` — salida individual por contrato US-02
   - `DistribucionResponse`, `ResumenPorMaterial` — full response US-02
   - `DistribucionMultiDiaRequest`, `ResultadoDia`, `DistribucionMultiDiaResponse` — US-03
   - `FilaSAP`, `ExportarExcelRequest` — US-04
   - `ConfiguracionDistribucion` — con validaciones Data Annotation US-05

2. Crear carpeta `/Domain` con entidades de persistencia:
   - `ConfiguracionPlanta` — entity para guardar configuración en BD

---

### FASE 2: Services de Lógica de Negocio (Molinos.DataAgro.Business)

#### 2A. SapFileParserService — parsear archivos SAP (US-01)
- Método `ParseAsync(Stream fileStream, DateTime fecha)` → `ImportResult`
- Detectar encoding (UTF-16LE BOM)
- Soportar formatos: TSV, XLS (via ExcelDataReader), XLSX
- Filtrar: `Descripción Centro == "Planta San Lorenzo"`, `Tipo == "CTO"`, cosechas válidas, excluir `MP-Venta granos`
- Procesar filas CCPP → balance por CUIT|Material
- Usar constantes de configuración (TARGET_CTR, VALID_COS, EXCL_DESC del HTML)

#### 2B. DistribucionService — algoritmo de distribución (US-02 y US-03)
- Método `Calcular(DistribucionRequest req)` → `DistribucionResponse`
- Lógica: **rank por prioridad** → **tope 30% CUIT** → **aplicar cuotas opcionales** → **spread uniforme (multi-día)**
- Implementar prioridad según `PRIO_MAP` del HTML (Fijo > Contrato Hijo > A Fijar > ...)
- Calcular `kgEfectivo = kg - ccppDescontado`
- Calcular `cuposNecesarios = ceil(kgEfectivo / 30000)` [CUPO_KG]
- Asignar cupos respetando límites y restricciones
- Método `CalcularMultiDia(DistribucionMultiDiaRequest req)` — iterar días con deducción única CCPP
- Marcar contratos "sin_cupos", "parcial", "completo", "sin_tope"

#### 2C. ConfiguracionService — persistencia de configuración (US-05)
- Método `ObtenerConfiguracion()` → `ConfiguracionDistribucion`
- Método `GuardarConfiguracion(ConfiguracionDistribucion config)` → void
- Validaciones: `cuitMaxPct ∈ [0, 1]`, suma cuotas ≤ 100%, etc.
- Valor por defecto: `cupoKg: 30000, cuitMaxPct: 0.30, cosechas: ["23-24", "24-25", "25-26"]`

#### 2D. ExcelExportService — generar .xlsx (US-04)
- Método `GenerarExcel(ExportarExcelRequest req)` → `byte[]`
- Crear workbook con hoja "Cupos"
- 3 columnas: FechaSugerida (dd/MM/yyyy), CantidadDeCupos (número), ContratoSAP (texto)
- Ordenar por fecha ascendente, luego SAP
- Usar EPPlus 5.x+

---

### FASE 3: Data Access (Molinos.DataAgro.Repository)

1. Crear `IConfigucionRepository` interface
2. Crear `ConfiguracionRepository` implementation:
   - `GetConfiguracion()` → `ConfiguracionPlanta` (con defaults si no existe)
   - `SaveConfiguracion(ConfiguracionPlanta)` → void
   - Mapear entity ↔ DTO

3. **Decisión BD**: Crear tabla `ConfiguracionesPlanta`:
   ```sql
   CREATE TABLE ConfiguracionesPlanta (
       Id INT PRIMARY KEY,
       PlantaCodigo VARCHAR(10),
       PlantaNombre VARCHAR(200),
       CupoKg INT,
       CuitMaxPct DECIMAL(3,2),
       CosechasValidas NVARCHAR(MAX), -- JSON array
       ClassesExcluidas NVARCHAR(MAX),
       LimitesPredeterminados NVARCHAR(MAX), -- JSON object
       PreciosReferencia NVARCHAR(MAX),
       CuotasPorOperador NVARCHAR(MAX),
       CuotasPorClase NVARCHAR(MAX),
       UltimaActualizacion DATETIME
   );
   ```
   - Seeding: insertar configuración por defecto para Planta San Lorenzo

---

### FASE 4: AutoMapper Configuration (Molinos.DataAgro.Mapping)

Crear `DistribucionMapperProfile`:
- ContratoSAP ↔ DTOs
- ResultadoContrato ↔ Entity
- ConfiguracionPlanta ↔ ConfiguracionDistribucion

---

### FASE 5: Controllers REST (WebDataAgro)

#### 5A. ContratosController (`POST /api/contratos/importar`)
- Atributo: `[EnableCors("LocalhostPolicy")]`
- Action: `ImportarArchivo(IFormFile file, [FromQuery] string fecha = null)`
- Validar: extensión, tamaño ≤ 10 MB
- Llamar `_sapParserService.ParseAsync(file.OpenReadStream(), fecha_parsed)`
- Retornar 200 con `ImportResult` o 400/422 con error

#### 5B. DistribucionController (`POST /api/distribucion/calcular`, `POST /api/distribucion/calcular-multi-dia`)
- `Calcular(DistribucionRequest req)` → `DistribucionResponse`
- `CalcularMultiDia(DistribucionMultiDiaRequest req)` → `DistribucionMultiDiaResponse`
- `ExportarExcel(ExportarExcelRequest req)` → `FileContentResult` (xlsx)
  - Header: `Content-Disposition: attachment; filename="Cupos_SL_{fecha}.xlsx"`
  - MIME: `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`

#### 5C. ConfiguracionController (`GET /api/configuracion`, `PUT /api/configuracion`)
- `GET()` → `ConfiguracionDistribucion`
- `PUT(ConfiguracionDistribucion config)` → 200 o 400 (con ModelState validation)

---

### FASE 6: Configuración CORS (WebDataAgro - Startup.cs/Program.cs)

```csharp
services.AddCors(options =>
{
    options.AddPolicy("LocalhostPolicy",
        builder =>
        {
            builder.WithOrigins("http://localhost:3000", "http://localhost:5000")
                   .AllowAnyMethod()
                   .AllowAnyHeader();
        });
});
```

---

### FASE 7: Unit Tests (Molinos.DataAgro.Test)

1. **SapFileParserServiceTests**
   - Parsear archivo UTF-16LE correctamente ✓
   - Filtrar solo contratos Planta San Lorenzo ✓
   - Calcular balance CCPP agrupado ✓
   - Error 422 si faltan columnas ✓

2. **DistribucionServiceTests**
   - Tope 30% CUIT no superado ✓
   - Cuotas por operador respetadas ✓
   - Multi-día con distribución uniforme ✓
   - Estado "sin_tope", "parcial", "completo" ✓

3. **ConfiguracionServiceTests**
   - Validar `cuitMaxPct` ∈ [0, 1] ✓
   - Suma cuotas ≤ 100% ✓
   - Persistir y recuperar correctamente ✓

4. **ExcelExportServiceTests**
   - Generar .xlsx con 3 columnas correctas ✓
   - ContratoSAP formateado como texto ✓

---

### FASE 8: Integración Frontend (distribuidor_cupos.html)

Actualizar los event handlers existentes para consumir la API en lugar de funciones locales:

```javascript
// Ejemplo: US-01 Importar
async function handleFile(file) {
    const formData = new FormData();
    formData.append('file', file);
    formData.append('fecha', getToday());
    
    const res = await fetch('/api/contratos/importar', {
        method: 'POST',
        body: formData
    });
    const data = await res.json();
    applyParsed(data);
}

// Ejemplo: US-02 Distribuir
async function runDistribution() {
    const req = {
        fecha: getToday(),
        contratos: S.contratos,
        ccppByCuitMat: S.ccppByCuitMat,
        limitesPorMaterial: /* obtener del UI */,
        configuracion: S.config
    };
    
    const res = await fetch('/api/distribucion/calcular', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(req)
    });
    const result = await res.json();
    renderResults(result, getToday());
}
```

---

## Verificación y Testing

1. **Tests Unitarios**: Ejecutar suite en Molinos.DataAgro.Test ✓
2. **Integración Manual**:
   - Subir `Ejemplo - A recibir 06.05.2026.xls` → validar parseo ✓
   - Ejecutar distribución un día → validar topes y cuotas ✓
   - Ejecutar distribución 3 días → validar spread uniforme ✓
   - Exportar Excel → descargar y abrir en Excel ✓
   - Guardar configuración → verificar en BD ✓
3. **Frontend**: Validar flujo completo en HTML ✓

---

## Archivos Clave a Crear/Modificar

**Entities (Molinos.DataAgro.Entities)**
- `ContratoSAP.cs`
- `DistribucionRequest.cs`, `DistribucionResponse.cs`, `DistribucionMultiDiaResponse.cs`
- `ConfiguracionDistribucion.cs` (con validaciones)
- `ConfiguracionPlanta.cs` (entity persistencia)

**Business (Molinos.DataAgro.Business)**
- `SapFileParserService.cs`
- `DistribucionService.cs`
- `ConfiguracionService.cs`
- `ExcelExportService.cs`

**Repository (Molinos.DataAgro.Repository)**
- `IConfigucionRepository.cs`
- `ConfiguracionRepository.cs`

**Mapping (Molinos.DataAgro.Mapping)**
- `DistribucionMapperProfile.cs`

**Controllers (WebDataAgro)**
- `ContratosController.cs`
- `DistribucionController.cs` (con submétodo ExportarExcel)
- `ConfiguracionController.cs`

**Config (WebDataAgro)**
- `Startup.cs` — Agregar CORS, DI services, AutoMapper

**Tests (Molinos.DataAgro.Test)**
- `SapFileParserServiceTests.cs`
- `DistribucionServiceTests.cs`
- `ConfiguracionServiceTests.cs`
- `ExcelExportServiceTests.cs`

**Database**
- Script SQL: crear tabla `ConfiguracionesPlanta`

---

## Decisiones Importantes

| Decisión | Justificación |
|----------|---|
| Extender .NET Framework 4.7.2 | Reutilizar arquitectura existente, minimizar cambios |
| Persistencia en BD | Configuración compartida entre usuarios/turnos |
| EPPlus 5.x+ | Open-source, versión 5+ sin licencia, stabilioad |
| Archivos en memoria | Simplifica implementación, archivos SAP son temporales |
| CORS localhost | Desarrollo local; parametrizar en appsettings.json para prod |

---

## Criterios de Aceptación del Plan

✅ Todos los 5 endpoints funcionan según especificación  
✅ Parseo de UTF-16LE SAP funciona correctamente  
✅ Tope 30% CUIT respetado en todas las distribuciones  
✅ Excel exportado abre sin errores en MS Excel  
✅ Configuración persiste entre sesiones  
✅ CORS funciona desde HTML local  
✅ Tests unitarios cubren casos críticos  
✅ Error handling devuelve status HTTP correctos (400, 422)

---

## Decisiones Clave de Arquitectura

### Proyectos a Modificar
1. **Molinos.DataAgro.Entities** (4.7.2) - Nuevos DTOs y entidades
2. **Molinos.DataAgro.Business** (4.7.2) - Services de lógica
3. **Molinos.DataAgro.Repository** (4.7.2) - DAL para configuración
4. **Molinos.DataAgro.Mapping** (4.5.2) - AutoMapper profiles
5. **WebDataAgro** (4.7.2) - Controllers REST
6. **Molinos.DataAgro.Test** (4.7.2) - Tests unitarios

### NuGet Packages a Agregar
- `EPPlus >= 5.0` (Excel export)
- `ExcelDataReader` (para leer XLS/XLSX)
- `NPOI` o `EPPlus` (para parsing SAP files UTF-16LE)
- Posiblemente actualizar `AutoMapper` si es muy antigua

### Configuración de Base de Datos
- Tabla `ConfiguracionesPlanta` con campos JSON para arrays y objetos complejos
- Seeding: valor por defecto para "Planta San Lorenzo"
- Patrón: simple pero escalable para futuras plantas

---

## Próximos Pasos

1. ✅ Aprobación del plan arquitectónico
2. **FASE 1-2**: Crear DTOs + servicios núcleo
3. **FASE 3-4**: Integrar DAL y mapeos
4. **FASE 5**: Controllers REST
5. **FASE 6**: Configuración CORS
6. **FASE 7**: Tests unitarios
7. **FASE 8**: Integración frontend + validación E2E
8. **BD**: Script SQL de seeding
9. **Deployment**: Configurar appsettings.json para producción
