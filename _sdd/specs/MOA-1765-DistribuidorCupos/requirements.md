# Requirements - Distribuidor de Cupos SL (migración a ASP.NET MVC)

## 1. Contexto de Negocio

El POC `distribuidor_cupos.html` (ver `req/distribuidor_cupos.html` y
`req/UserStories_API_Cupos.md`) resuelve, 100% client-side, la
distribución diaria/semanal de cupos de recepción de granos para la
Planta San Lorenzo: parsea el archivo SAP "A Recibir", calcula el
balance de CCPP, aplica el algoritmo de reparto (tope 30% por CUIT,
cuotas opcionales por tipo de operador/clase) y exporta un Excel listo
para SAP. Esta feature migra esa lógica a `WebDataAgro` como una Vista
Razor + Managers de backend (`Molinos.DataAgro.Business`), eliminando
el cálculo en el navegador y agregando persistencia de configuración y
la integración con el algoritmo existente de alta masiva de sugerencias
de cupos (`ICupoManager.AltaMasivaSugerenciaCuposV2`).

El análisis técnico completo (DTOs, Managers, Controllers, Vista, CSS/JS,
tests) está detallado en
`_sdd/specs/MOA-1765-DistribuidorCupos/plan-distribucionCuposSL-v2.md`.
Las historias de usuario funcionales originales (US-01 a US-05, formato
API standalone) están en `req/UserStories_API_Cupos.md`; el plan v3
las reinterpreta como Managers + Controllers MVC de la misma app (sin
CORS), y agrega US-06 (procesar en DataAgro).

- Jira: MOA-1765 — Distribuidor de Cupos SL

## 2. Requisitos en Formato EARS

- **R1 - Servir la Vista protegida:**
  - **Cuando** un usuario con el permiso `SugerenciaDeCupos` navega a
    `GET /Distribucion`, **el sistema debe** devolver la Vista
    `Views/Distribucion/Index.cshtml` (HTTP 200); sin ese permiso debe
    denegar el acceso igual que el resto de las pantallas protegidas
    por `[Autorizacion]`.
  - Tipo de verificación: manual

- **R2 - Importar y parsear archivo SAP (US-01):**
  - **Cuando** el frontend envía `POST /api/contratos/importar` con un
    archivo `.xls`/`.xlsx`/`.txt`/`.tsv`, **el sistema debe** parsearlo
    con `SapArchivoParserManager`, filtrando solo `Centro = 'Planta San
    Lorenzo'`, `Tipo = 'CTO'`, cosechas válidas (`23-24`, `24-25`,
    `25-26`) y excluyendo `MP-Venta granos`, y devolver contratos,
    balance CCPP (por `CUIT|Material` y por `Material`), materiales y
    estadísticas.
  - Tipo de verificación: unit

- **R3 - Detección de encoding UTF-16LE:**
  - **Cuando** se sube un archivo con BOM UTF-16LE (export estándar de
    SAP), **el sistema debe** decodificarlo correctamente sin errores
    de caracteres especiales (ñ, tildes).
  - Tipo de verificación: unit

- **R4 - Error de formato/encabezado inválido (US-01):**
  - **Cuando** el archivo no tiene extensión válida o no contiene la
    fila de encabezados esperada (columna `Tipo`), **el sistema debe**
    responder HTTP 400 (extensión inválida) o HTTP 422 (encabezado no
    encontrado) con mensaje descriptivo, sin lanzar excepción no
    controlada.
  - Tipo de verificación: unit

- **R5 - Distribución de cupos de un día con tope 30% por CUIT (US-02):**
  - **Cuando** el frontend envía `POST /api/distribucion/calcular` con
    contratos, balance CCPP y límites por material, **el sistema debe**
    calcular `kgEfectivo = kg - ccppDescontado`,
    `cuposNecesarios = ceil(kgEfectivo / 30000)` y asignar cupos por
    contrato respetando que ningún CUIT reciba más del 30% del límite
    del material.
  - Tipo de verificación: unit

- **R6 - Cuotas opcionales por operador/clase (US-02):**
  - **Cuando** `configuracion.aplicarCuotaOperador` o
    `aplicarCuotaClase` es `true`, **el sistema debe** limitar los
    cupos asignados a cada tipo de operador/clase según el porcentaje
    configurado sobre el límite del material.
  - Tipo de verificación: unit

- **R7 - Material sin límite configurado (US-02):**
  - **Cuando** `limitesPorMaterial` no incluye un material o su valor es
    0, **el sistema debe** marcar todos los contratos de ese material
    con `estado = "sin_tope"` y 0 cupos asignados.
  - Tipo de verificación: unit

- **R8 - Distribución multi-día, máximo 7 fechas (US-03):**
  - **Cuando** el frontend envía `POST /api/distribucion/calcular-multi-dia`
    con más de 7 fechas, **el sistema debe** responder HTTP 400 con el
    mensaje `"El máximo de días permitido es 7"` sin ejecutar el
    algoritmo.
  - Tipo de verificación: unit

- **R9 - Multi-día: fecha sin límites definidos (US-03):**
  - **Cuando** `limitesPorDia` no incluye una entrada para alguna fecha
    presente en `fechas`, **el sistema debe** responder HTTP 422
    indicando qué fecha no tiene límites definidos.
  - Tipo de verificación: unit

- **R10 - Multi-día: CCPP descontado una sola vez y reparto uniforme
    (US-03):**
  - **Cuando** se ejecuta `calcular-multi-dia` con
    `distribuirUniforme = true`, **el sistema debe** descontar el
    balance CCPP del stock una única vez antes de iterar los días, y
    repartir los cupos de cada contrato lo más uniformemente posible
    entre los días con disponibilidad, sin volver a asignar cupos a un
    contrato ya satisfecho en un día anterior.
  - Tipo de verificación: unit

- **R11 - Exportar Excel desde el frontend (US-04):**
  - **Cuando** el usuario pulsa "Exportar Excel" con un resultado de
    distribución disponible, **el sistema debe** generar (vía SheetJS
    en el navegador, sin endpoint backend) un archivo
    `Cupos_SL_<fecha>.xlsx` con la hoja `Cupos` y las columnas
    `FechaSugerida`, `CantidadDeCupos`, `ContratoSAP` (esta última como
    texto, preservando ceros a la izquierda).
  - Tipo de verificación: manual

- **R12 - Persistir configuración de distribución (US-05):**
  - **Cuando** el frontend hace `GET /api/configuracion/distribucion`
    sin configuración previa guardada, **el sistema debe** devolver los
    valores por defecto (`cupoKg = 30000`, `cuitMaxPct = 0.30`,
    cosechas válidas `23-24`/`24-25`/`25-26`); cuando hace
    `PUT /api/configuracion/distribucion` con una configuración válida,
    **el sistema debe** persistirla y devolverla en el siguiente `GET`
    con `ultimaActualizacion` actualizado.
  - Tipo de verificación: unit

- **R13 - Validación de configuración inválida (US-05):**
  - **Cuando** se envía `PUT /api/configuracion/distribucion` con
    `cuitMaxPct` fuera de `[0.0, 1.0]`, o con `cuotasPorOperador` o
    `cuotasPorClase` sumando más de 100%, **el sistema debe** responder
    HTTP 400 con un mensaje descriptivo y no persistir el cambio.
  - Tipo de verificación: unit

- **R14 - Procesar resultado en DataAgro (US-06):**
  - **Cuando** el frontend envía `POST /api/distribucion/procesar-en-dataagro`
    con un array `sap` no vacío, **el sistema debe** construir el
    `DataSet` esperado por `ICupoManager.AltaMasivaSugerenciaCuposV2`
    (convirtiendo `fechaSugerida` de `"dd/MM/yyyy"` a `OADate` y
    manteniendo `contratoSAP` como texto), invocar el método y devolver
    `{ resultado: true, resume, mensaje }`.
  - Tipo de verificación: unit

- **R15 - Procesar en DataAgro: validaciones y errores (US-06):**
  - **Cuando** el array `sap` viene vacío o nulo, **el sistema debe**
    responder `{ resultado: false, error: "El array 'sap' no puede
    estar vacío" }` sin invocar `AltaMasivaSugerenciaCuposV2`; si la
    llamada al manager lanza una excepción, **el sistema debe**
    responder `{ resultado: false, error: "Error al procesar: <mensaje>" }`
    sin propagar una excepción HTTP 500 sin manejar.
  - Tipo de verificación: unit

- **R16 - Sin CORS, URLs inyectadas via Razor:**
  - **Cuando** se renderiza `Views/Distribucion/Index.cshtml`, **el
    sistema debe** inyectar las URLs de los endpoints (`API_IMPORTAR`,
    `API_CALCULAR`, `API_CALCULAR_MD`, `API_CONFIG_GET`,
    `API_CONFIG_PUT`, `API_PROCESAR_DATAAGRO`) vía `@Url.Action(...)`
    en lugar de hardcodearlas, y no requerir ninguna configuración de
    CORS porque todas las llamadas `fetch()` son del mismo origen.
  - Tipo de verificación: manual

- **R17 - Paridad visual con el POC y sin estilos inline:**
  - **Cuando** se compara la Vista Razor renderizada contra el POC
    `distribuidor_cupos.html`, **el sistema debe** verse idéntico
    (mismo layout, colores e imagen de fondo del header ahora servida
    como archivo estático en `Content/Images/`), sin atributos
    `style="..."` inline en el markup ni generados dinámicamente por
    JS — todo el estilo debe vivir en `distribucion-cupos.css` vía
    clases utilitarias (siguiendo
    `.github/instructions/instructions.frontend-template.md`).
  - Tipo de verificación: manual
