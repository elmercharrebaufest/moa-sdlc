# Historias de Usuario — API .NET Core: Distribución de Cupos SL

> **Contexto:** La aplicación `distribuidor_cupos.html` es actualmente 100% client-side. Estas historias migran su lógica a una API RESTful en .NET Core, exponiendo cada etapa del workflow como un endpoint independiente. El frontend HTML consumirá la API vía `fetch()`.

---

# US-01: Importar y parsear archivo SAP "A Recibir"

**Como** operador de planta
**Quiero** subir el archivo SAP de contratos a recibir (TSV/XLS/XLSX) a la API
**Para** obtener los contratos procesados y el balance CCPP sin tener que ejecutar la lógica de parsing en el navegador

## Detalles Técnicos (.NET Core API)

* **Endpoint:** `POST /api/contratos/importar`
* **Content-Type:** `multipart/form-data`
* **Request Body (Form):**
  ```
  file: archivo SAP (*.xls, *.xlsx, *.txt, *.tsv) — requerido
  fecha: string "YYYY-MM-DD" — fecha de distribución base (default: hoy)
  ```
* **Response (JSON):**
  ```json
  {
    "contratos": [
      {
        "numero": "0004500012345",
        "numeroSAP": "4500012345",
        "descCl": "Fijo",
        "prioLabel": "Fijo",
        "rank": 3,
        "material": "Soja Poroto",
        "kg": 900000,
        "cosecha": "25-26",
        "fechaContrato": "2026-01-15",
        "fechaDesde": "2026-01-15",
        "fechaHasta": "2026-06-30",
        "cuit": "20-12345678-9",
        "proveedor": "AGRO S.A.",
        "corredor": "CORREDOR SRL",
        "clasificacion": "ACOPIADOR",
        "opType": "acopiador",
        "opLabel": "Acopiador directo",
        "isSust": false,
        "valor": 450000.00,
        "moneda": "USD",
        "pricePt": 500.00,
        "priceRank": 0
      }
    ],
    "ccppByCuitMat": {
      "20-12345678-9|Soja Poroto": 60000
    },
    "ccppByMat": {
      "Soja Poroto": 60000
    },
    "materiales": ["Soja Poroto", "Maíz"],
    "estadisticas": {
      "porTipoOperador": [
        { "tipoId": "acopiador", "label": "Acopiador directo", "contratos": 12, "kgTotal": 3600000, "porcentaje": 65.4 }
      ],
      "porClase": [
        { "clase": "Fijo", "contratos": 8, "kgTotal": 2400000, "porcentaje": 43.6 }
      ],
      "sustentables": 5,
      "fechaDesdeMin": "2026-01-01",
      "fechaDesdeMax": "2026-05-01",
      "fechaHastaMin": "2026-06-01",
      "fechaHastaMax": "2026-12-31"
    },
    "errores": []
  }
  ```
* **Response Error (400):**
  ```json
  { "error": "Formato de archivo no válido. Solo se aceptan .xls, .xlsx, .txt, .tsv" }
  ```

## Criterios de Aceptación (Formato Gherkin)

* **Dado que** el operador sube un archivo TSV exportado de SAP con contratos de Planta San Lorenzo
* **Cuando** el endpoint recibe el archivo
* **Entonces** retorna solo los contratos con `Descripción Centro = 'Planta San Lorenzo'`, `Tipo = 'CTO'`, cosechas válidas (`23-24`, `24-25`, `25-26`) y excluye `MP-Venta granos`

---

* **Dado que** el archivo contiene filas de tipo `CCPP`
* **Cuando** se parsea el archivo
* **Entonces** el balance CCPP se calcula agrupado por `CUIT|Material` y por `Material`

---

* **Dado que** se sube un archivo con encoding UTF-16LE (export estándar de SAP)
* **Cuando** el parser detecta el BOM del archivo
* **Entonces** decodifica correctamente sin errores de caracteres especiales (ñ, tildes)

---

* **Dado que** se sube un archivo sin la columna `Tipo` o `Descripción`
* **Cuando** el parser no encuentra la fila de encabezados esperada
* **Entonces** retorna HTTP 422 con mensaje descriptivo del error

## Checklist de Tareas para Developers

- [ ] Crear el modelo C# `ContratoSAP`, `CcppBalance`, `EstadisticasImport`
- [ ] Crear `SapFileParserService` que soporte UTF-16LE/BE/UTF-8 y formatos TSV y XLSX
- [ ] Crear `ContratosController` con `[HttpPost("importar")]`
- [ ] Validar extensión de archivo con `[AllowedExtensions]` Data Annotation
- [ ] Limitar tamaño máximo de archivo (sugerido: 10 MB)
- [ ] Configurar CORS para permitir la conexión desde el HTML
- [ ] Escribir pruebas unitarias básicas para `SapFileParserService`

---

# US-02: Ejecutar distribución de cupos (un día)

**Como** operador de planta
**Quiero** enviar los contratos parseados junto con la configuración de límites y cuotas
**Para** obtener la asignación óptima de cupos por contrato para un día determinado, respetando el tope del 30% por CUIT

## Detalles Técnicos (.NET Core API)

* **Endpoint:** `POST /api/distribucion/calcular`
* **Request Body (JSON):**
  ```json
  {
    "fecha": "2026-06-11",
    "contratos": [ /* array de ContratoSAP del endpoint anterior */ ],
    "ccppByCuitMat": { "20-12345678-9|Soja Poroto": 60000 },
    "limitesPorMaterial": {
      "Soja Poroto": 20,
      "Maíz": 10
    },
    "configuracion": {
      "aplicarCuotaOperador": false,
      "cuotasPorOperador": {
        "con_corredor": 30,
        "acopiador": 40,
        "productor": 20,
        "otros": 10
      },
      "aplicarCuotaClase": false,
      "cuotasPorClase": {
        "MP-Fason": 20,
        "MP-Prest/Devolución": 10,
        "Fijo": 50,
        "Contrato Hijo": 10,
        "A Fijar": 10
      },
      "excluirFason": false,
      "excluirAgenteCompra": false,
      "priorizarSustentables": false,
      "preciosReferencia": {
        "Soja Poroto": 280.50
      },
      "habilitarFiltroFechas": false,
      "filtroFechaDesdeMin": null,
      "filtroFechaDesdeMax": null,
      "filtroFechaHastaMin": null,
      "filtroFechaHastaMax": null
    }
  }
  ```
* **Response (JSON):**
  ```json
  {
    "fecha": "2026-06-11",
    "resultados": [
      {
        "numeroSAP": "4500012345",
        "proveedor": "AGRO S.A.",
        "cuit": "20-12345678-9",
        "material": "Soja Poroto",
        "opType": "acopiador",
        "clase": "Fijo",
        "kgContrato": 900000,
        "ccppDescontado": 60000,
        "kgEfectivo": 840000,
        "cuposNecesarios": 28,
        "cuposAsignados": 6,
        "estado": "parcial",
        "isSust": false,
        "pricePt": 500.00
      }
    ],
    "resumenPorMaterial": [
      {
        "material": "Soja Poroto",
        "limite": 20,
        "cuposAsignados": 20,
        "cuposRestantes": 0,
        "contratosAsignados": 4,
        "contratosSinCupos": 2
      }
    ],
    "resumenPorOperador": [ /* misma estructura */ ],
    "resumenPorClase": [ /* misma estructura */ ],
    "sap": [
      { "fechaSugerida": "11/06/2026", "cantidadDeCupos": 6, "contratoSAP": "4500012345" }
    ]
  }
  ```

## Criterios de Aceptación (Formato Gherkin)

* **Dado que** el límite diario para Soja Poroto es 20 cupos y hay 5 CUITs activos
* **Cuando** se ejecuta la distribución
* **Entonces** ningún CUIT recibe más de 6 cupos (30% de 20) para ese material

---

* **Dado que** `aplicarCuotaOperador = true` con `acopiador = 40%` y límite de 20 cupos
* **Cuando** se ejecuta la distribución
* **Entonces** el total de cupos asignados a contratos de tipo `acopiador` no supera 8 cupos (40% de 20)

---

* **Dado que** un contrato tiene balance CCPP de 60.000 kg y el contrato tiene 300.000 kg
* **Cuando** se calcula el `kgEfectivo`
* **Entonces** `kgEfectivo = 240.000` y la asignación se basa en ese valor reducido

---

* **Dado que** `limitesPorMaterial` contiene un material con valor 0 o no está incluido
* **Cuando** se ejecuta la distribución
* **Entonces** los contratos de ese material reciben `estado = "sin_tope"` y 0 cupos asignados

## Checklist de Tareas para Developers

- [ ] Crear modelos C# `DistribucionRequest`, `DistribucionConfig`, `ResultadoContrato`, `DistribucionResponse`
- [ ] Implementar `DistribucionService.Calcular()` con la lógica de prioridad, tope CUIT 30%, y cuotas opcionales
- [ ] Crear `DistribucionController` con `[HttpPost("calcular")]`
- [ ] Validar que `limitesPorMaterial` no tenga valores negativos
- [ ] Validar que las cuotas por operador y por clase sumen ≤ 100%
- [ ] Configurar CORS para permitir la conexión desde el HTML
- [ ] Escribir pruebas unitarias básicas para `DistribucionService`

---

# US-03: Ejecutar distribución de cupos multi-día

**Como** operador de planta
**Quiero** ejecutar la distribución para hasta 7 días consecutivos en un solo request
**Para** planificar la asignación de cupos de la semana sin ejecutar el algoritmo día por día

## Detalles Técnicos (.NET Core API)

* **Endpoint:** `POST /api/distribucion/calcular-multi-dia`
* **Request Body (JSON):**
  ```json
  {
    "contratos": [ /* array ContratoSAP */ ],
    "ccppByCuitMat": { "20-12345678-9|Soja Poroto": 60000 },
    "fechas": ["2026-06-11", "2026-06-12", "2026-06-13"],
    "limitesPorDia": [
      { "fecha": "2026-06-11", "limites": { "Soja Poroto": 20, "Maíz": 10 } },
      { "fecha": "2026-06-12", "limites": { "Soja Poroto": 18, "Maíz": 8 } },
      { "fecha": "2026-06-13", "limites": { "Soja Poroto": 20, "Maíz": 10 } }
    ],
    "distribuirUniforme": true,
    "configuracion": { /* misma estructura que US-02 */ }
  }
  ```
* **Response (JSON):**
  ```json
  {
    "resultadosPorDia": [
      {
        "fecha": "2026-06-11",
        "resultados": [ /* array ResultadoContrato */ ],
        "sap": [ /* array { fechaSugerida, cantidadDeCupos, contratoSAP } */ ]
      }
    ],
    "sapConsolidado": [
      { "fechaSugerida": "11/06/2026", "cantidadDeCupos": 6, "contratoSAP": "4500012345" },
      { "fechaSugerida": "12/06/2026", "cantidadDeCupos": 6, "contratoSAP": "4500012345" }
    ],
    "resumenGlobal": {
      "totalCuposAsignados": 58,
      "totalContratosAsignados": 12,
      "porcentajeCoberturaKg": 78.3
    }
  }
  ```
* **Response Error (400):**
  ```json
  { "error": "El máximo de días permitido es 7" }
  ```

## Criterios de Aceptación (Formato Gherkin)

* **Dado que** se solicita distribución para 3 días con `distribuirUniforme = true`
* **Cuando** se ejecuta el algoritmo
* **Entonces** el CCPP se descuenta una sola vez del stock total antes de iterar los días, y los cupos de cada contrato se distribuyen lo más uniformemente posible entre los días con disponibilidad

---

* **Dado que** un contrato recibió todos sus cupos necesarios en el día 1
* **Cuando** se procesa el día 2
* **Entonces** ese contrato no aparece en los resultados del día 2 (ya satisfecho)

---

* **Dado que** se envían más de 7 fechas en el request
* **Cuando** el endpoint recibe el request
* **Entonces** retorna HTTP 400 con mensaje `"El máximo de días permitido es 7"`

---

* **Dado que** `limitesPorDia` no incluye una fecha que sí está en `fechas`
* **Cuando** el endpoint recibe el request
* **Entonces** retorna HTTP 422 indicando qué fecha no tiene límites definidos

## Checklist de Tareas para Developers

- [ ] Crear modelos C# `DistribucionMultiDiaRequest`, `LimiteDia`, `ResultadoDia`, `DistribucionMultiDiaResponse`
- [ ] Implementar `DistribucionService.CalcularMultiDia()` con deducción única de CCPP y lógica de spread uniforme
- [ ] Agregar endpoint `[HttpPost("calcular-multi-dia")]` en `DistribucionController`
- [ ] Validar que `fechas.Count` esté entre 2 y 7
- [ ] Validar que cada fecha en `fechas` tenga su entrada correspondiente en `limitesPorDia`
- [ ] Configurar CORS para permitir la conexión desde el HTML
- [ ] Escribir pruebas unitarias básicas para `DistribucionService.CalcularMultiDia()`

---

# US-04: Exportar resultado de distribución a Excel (.xlsx)

**Como** operador de planta
**Quiero** enviar los resultados de la distribución a la API y recibir un archivo Excel listo para SAP
**Para** descargar el archivo `Cupos_SL_<fecha>.xlsx` sin depender de la librería SheetJS en el navegador

## Detalles Técnicos (.NET Core API)

* **Endpoint:** `POST /api/distribucion/exportar-excel`
* **Request Body (JSON):**
  ```json
  {
    "fecha": "2026-06-11",
    "sap": [
      { "fechaSugerida": "11/06/2026", "cantidadDeCupos": 6, "contratoSAP": "4500012345" },
      { "fechaSugerida": "12/06/2026", "cantidadDeCupos": 4, "contratoSAP": "4500067890" }
    ]
  }
  ```
* **Response (éxito — 200):**
  ```
  Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
  Content-Disposition: attachment; filename="Cupos_SL_11062026.xlsx"
  [binary stream]
  ```
  El archivo contiene una hoja llamada `Cupos` con exactamente tres columnas:
  | FechaSugerida | CantidadDeCupos | ContratoSAP |
  |---|---|---|
  | 11/06/2026 | 6 | 4500012345 |
* **Response Error (400):**
  ```json
  { "error": "El array 'sap' no puede estar vacío" }
  ```

## Criterios de Aceptación (Formato Gherkin)

* **Dado que** se envía un array `sap` con 15 filas de resultado
* **Cuando** el endpoint procesa el request
* **Entonces** retorna un archivo `.xlsx` con una hoja `Cupos` que contiene exactamente 15 filas de datos más la fila de encabezados

---

* **Dado que** el archivo generado se abre en Excel
* **Cuando** el usuario revisa la columna `ContratoSAP`
* **Entonces** los valores son de tipo texto (no número) para preservar los ceros a la izquierda si los hubiere

---

* **Dado que** se envía un array `sap` vacío `[]`
* **Cuando** el endpoint recibe el request
* **Entonces** retorna HTTP 400 con mensaje `"El array 'sap' no puede estar vacío"`

---

* **Dado que** la distribución incluye múltiples fechas (multi-día)
* **Cuando** se exporta el `sapConsolidado`
* **Entonces** el archivo contiene todas las filas de todos los días, ordenadas por `FechaSugerida` ascendente y luego por `ContratoSAP`

## Checklist de Tareas para Developers

- [ ] Crear modelos C# `ExportarExcelRequest`, `FilaSAP`
- [ ] Implementar `ExcelExportService` usando `ClosedXML` o `EPPlus` para generar el `.xlsx`
- [ ] Crear `ExportController` con `[HttpPost("exportar-excel")]` que retorne `FileContentResult`
- [ ] Asegurar que la columna `ContratoSAP` se formatee como texto (`@`) en Excel
- [ ] Configurar CORS para permitir la conexión desde el HTML
- [ ] Escribir pruebas unitarias básicas para `ExcelExportService`

---

# US-05: Consultar y persistir configuración de parámetros de distribución

**Como** operador de planta
**Quiero** guardar y recuperar la configuración habitual de límites, topes y parámetros de la planta
**Para** no tener que reingresar los mismos valores cada vez que ejecuto la distribución diaria

## Detalles Técnicos (.NET Core API)

* **Endpoints:**
  * `GET /api/configuracion` — obtiene la configuración activa
  * `PUT /api/configuracion` — guarda o reemplaza la configuración activa
* **Request Body `PUT` (JSON):**
  ```json
  {
    "plantaCodigo": "SL",
    "plantaNombre": "Planta San Lorenzo",
    "cupoKg": 30000,
    "cuitMaxPct": 0.30,
    "cosechasValidas": ["23-24", "24-25", "25-26"],
    "clasesExcluidas": ["MP-Venta granos"],
    "limitesPredeterminadosPorMaterial": {
      "Soja Poroto": 20,
      "Maíz": 10
    },
    "preciosReferencia": {
      "Soja Poroto": 280.50,
      "Maíz": 165.00
    },
    "cuotasPorOperador": {
      "con_corredor": 30,
      "acopiador": 40,
      "productor": 20,
      "otros": 10
    },
    "cuotasPorClase": {
      "MP-Fason": 20,
      "MP-Prest/Devolución": 10,
      "Fijo": 50,
      "Contrato Hijo": 10,
      "A Fijar": 10
    }
  }
  ```
* **Response `GET` (JSON):**
  ```json
  {
    "plantaCodigo": "SL",
    "plantaNombre": "Planta San Lorenzo",
    "cupoKg": 30000,
    "cuitMaxPct": 0.30,
    "cosechasValidas": ["23-24", "24-25", "25-26"],
    "clasesExcluidas": ["MP-Venta granos"],
    "limitesPredeterminadosPorMaterial": { "Soja Poroto": 20, "Maíz": 10 },
    "preciosReferencia": { "Soja Poroto": 280.50 },
    "cuotasPorOperador": { "con_corredor": 30, "acopiador": 40, "productor": 20, "otros": 10 },
    "cuotasPorClase": { "MP-Fason": 20, "MP-Prest/Devolución": 10, "Fijo": 50, "Contrato Hijo": 10, "A Fijar": 10 },
    "ultimaActualizacion": "2026-06-11T10:23:00Z"
  }
  ```
* **Response Error `PUT` (400):**
  ```json
  { "error": "cuitMaxPct debe ser un valor entre 0.0 y 1.0" }
  ```

## Criterios de Aceptación (Formato Gherkin)

* **Dado que** no existe ninguna configuración guardada
* **Cuando** el frontend hace `GET /api/configuracion`
* **Entonces** retorna HTTP 200 con los valores por defecto del sistema (`cupoKg: 30000`, `cuitMaxPct: 0.30`, cosechas válidas actuales)

---

* **Dado que** el operador guarda una nueva configuración con `PUT`
* **Cuando** hace posteriormente `GET /api/configuracion`
* **Entonces** retorna la configuración actualizada con el `ultimaActualizacion` correcto

---

* **Dado que** se intenta guardar `cuitMaxPct: 1.5`
* **Cuando** el endpoint valida el request
* **Entonces** retorna HTTP 400 con el mensaje correspondiente

---

* **Dado que** `cuotasPorOperador` suma más de 100%
* **Cuando** el endpoint valida el request
* **Entonces** retorna HTTP 400 con mensaje `"Las cuotas por operador no pueden superar el 100%"`

## Checklist de Tareas para Developers

- [ ] Crear el modelo C# `ConfiguracionDistribucion` con Data Annotations de validación
- [ ] Crear `ConfiguracionController` con `[HttpGet]` y `[HttpPut]`
- [ ] Implementar `ConfiguracionService` que persista en base de datos o archivo JSON (según arquitectura del proyecto)
- [ ] Seed de valores por defecto en la base de datos / archivo de configuración inicial
- [ ] Validar que `cuitMaxPct` esté entre 0.0 y 1.0
- [ ] Validar que la suma de `cuotasPorOperador` y `cuotasPorClase` sea ≤ 100 cada una
- [ ] Configurar CORS para permitir la conexión desde el HTML
- [ ] Escribir pruebas unitarias básicas para las validaciones del modelo

---

## Resumen de Endpoints

| # | Método | Endpoint | Descripción |
|---|---|---|---|
| US-01 | POST | `/api/contratos/importar` | Parsear archivo SAP |
| US-02 | POST | `/api/distribucion/calcular` | Distribución un día |
| US-03 | POST | `/api/distribucion/calcular-multi-dia` | Distribución multi-día |
| US-04 | POST | `/api/distribucion/exportar-excel` | Exportar a .xlsx |
| US-05 | GET/PUT | `/api/configuracion` | Leer/guardar parámetros |
