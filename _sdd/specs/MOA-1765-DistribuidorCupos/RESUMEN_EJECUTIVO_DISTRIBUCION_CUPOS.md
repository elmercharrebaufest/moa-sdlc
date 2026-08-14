# Resumen Ejecutivo — Módulo Distribución de Cupos SL

**Fecha:** 01/07/2026  
**Proyecto:** DataAgro — Distribución de Cupos Planta San Lorenzo

---

## Objetivo

Migrar la herramienta `distribuidor_cupos.html` (actualmente standalone y 100% client-side) a una funcionalidad nativa dentro de la aplicación DataAgro, respetando su arquitectura en capas y exponiéndola como módulo MVC con lógica de negocio en el backend.

---

## Proceso a ejecutar

El trabajo se organiza en **6 Historias de Usuario** que cubren el ciclo completo de la distribución diaria de cupos:

| ID | Descripción | Endpoint |
|---|---|---|
| US-01 | Importar y parsear archivo SAP ("A Recibir") | `POST /api/contratos/importar` |
| US-02 | Calcular distribución de cupos para un día | `POST /api/distribucion/calcular` |
| US-03 | Calcular distribución multi-día (hasta 7 días) | `POST /api/distribucion/calcular-multi-dia` |
| US-04 | Exportar resultado a Excel para SAP | Frontend (SheetJS) |
| US-05 | Leer y persistir configuración de parámetros de planta | `GET/PUT /api/configuracion/distribucion` |
| US-06 | Procesar el resultado directamente en DataAgro (sin pasar por SAP/Excel) | `POST /api/distribucion/procesar-en-dataagro` |

El flujo operativo es: el operador sube el archivo SAP → el sistema parsea los contratos → ejecuta el algoritmo de distribución respetando topes por CUIT (30%), cuotas por tipo de operador y por clase → entrega el resultado visual en pantalla y, según lo necesite el operador, genera el Excel listo para importar en SAP (US-04) **o** lo procesa directamente en DataAgro con un botón dedicado (US-06), reutilizando la lógica de alta masiva ya existente (`AltaMasivaSugerenciaCuposV2`).

---

## Módulos impactados

| Proyecto | Cambios |
|---|---|
| **WebDataAgro** | 4 controllers nuevos (`DistribucionController`, `ContratosApiController`, `DistribucionApiController`, `ConfiguracionDistribucionController`) + 1 Razor View (`Views/Distribucion/Index.cshtml`) + CSS y JS en bundles + nueva entrada de menú "Distribuidor Cupos" en `Views/Shared/_Layout.cshtml` (dropdown "Cupos"). Incluye la **conversión completa del CSS y los estilos inline de la maqueta** (`Content/distribucion-cupos.css` con variables prefijadas y clases utilitarias, sin `style=""` inline ni imágenes en base64) a los estándares de la solución |
| **Molinos.DataAgro.Business** | 3 managers nuevos: `SapArchivoParserManager`, `DistribucionCuposManager`, `ConfiguracionDistribucionManager`. Se reutiliza el manager existente `CupoManager` (método `AltaMasivaSugerenciaCuposV2`) para US-06 |
| **Molinos.DataAgro.Interfaces** | 3 interfaces nuevas: `ISapArchivoParserManager`, `IDistribucionCuposManager`, `IConfiguracionDistribucionManager` |
| **Molinos.DataAgro.Entities** | Carpeta `Dto/Distribucion/` con DTOs nuevos (incluye `ProcesarEnDataAgroRequestDto` de US-06) + entity `ConfiguracionDistribucionPlanta` |
| **Base de Datos** | 1 tabla nueva: `ConfiguracionesDistribucionPlanta` |

No se modifica ningún módulo existente de la solución. **No se crea ningún permiso nuevo**: la entrada de menú y los endpoints reutilizan el permiso ya existente `SugerenciaDeCupos`, evitando alta en el ABM de Roles/Permisos. El menú horizontal de DataAgro está hardcodeado en Razor (no hay tabla SQL de menú), por lo que agregar la opción implica editar `_Layout.cshtml` y desplegar.

---

## Cómo integrar un HTML maqueta en una aplicación con capas

Cuando se tiene un prototipo HTML autónomo (con lógica en JavaScript), el proceso de integración a una arquitectura en capas sigue estos pasos: **(1)** identificar toda la lógica de negocio embebida en el JS (parseo, cálculos, validaciones) y moverla al backend como clases `Manager` en la capa de Business; **(2)** definir los contratos de entrada/salida como DTOs en la capa de Entities y las interfaces correspondientes en la capa de Interfaces; **(3)** crear los Controllers en la capa Web que exponen los endpoints REST, conectados al Manager vía inyección de dependencias (Autofac); **(4)** convertir el HTML en una Razor View, extrayendo el CSS a `Content/` y el JavaScript de UI a `Scripts/`, reemplazando las URLs hardcodeadas por `@Url.Action()`. El resultado es que el HTML original pasa a ser solo la capa de presentación, sin lógica, y toda la operación queda dentro del ciclo de vida de la aplicación (autenticación, logging, pruebas unitarias).

**Conversión de CSS y estilos inline:** la maqueta trae un bloque `<style>` propio (variables `:root`, ~90 reglas) más **~102 atributos `style="..."` inline**, la mitad de ellos generados dinámicamente por JavaScript al construir tablas y tarjetas. Además, el fondo del header está embebido como imagen en base64 dentro del CSS. La conversión implica: (1) mover el CSS a `Content/distribucion-cupos.css` con las variables prefijadas para evitar colisiones futuras; (2) extraer la imagen base64 a un archivo estático en `Content/Images/`; (3) reemplazar todos los `style=""` inline —tanto los del markup como los que arma el JS— por clases utilitarias reusables (`.mt-14`, `.d-flex-center`, `.text-center`, etc.) definidas en ese mismo CSS. El resultado visual debe ser idéntico al POC, validado lado a lado por el Analista Funcional.

---

## Estimación

La estimación detallada (por fase, con y sin asistencia de IA) está en el documento técnico. Resumen ejecutivo por rol:

| Rol / Actividad | Sin IA | **Con IA** | Ahorro |
|---|---|---|---|
| Desarrollo (Dev) | 44.5 h | **13.75 h** | 69% |
| QA (funcional + regresión) | 21 h | **13 h** | 38% |
| Analista Funcional (UAT) | 8 h | **8 h** | 0% |
| Despliegues (QA + PROD) | 8 h | **8 h** | 0% |
| **Total** | **81.5 h (~10.2 días hábiles)** | **~42.75 h (~5.3 días hábiles)** | **48%** |

> El ahorro de IA es alto en desarrollo (69%) pero bajo en QA/Funcional/Despliegues (0-38%), porque son actividades humanas y operativas. **Duración calendario estimada: 3 a 4 semanas**, considerando dedicación parcial del equipo (no full-time) y los tiempos de coordinación de ventanas de despliegue a QA y PROD.

---

*Documento de referencia técnica: `Docs/Reqs/plan-distribucionCuposSL-v2.md` · `Docs/Reqs/UserStories_API_Cupos.md`*
