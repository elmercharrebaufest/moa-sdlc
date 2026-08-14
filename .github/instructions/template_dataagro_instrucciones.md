# Instrucciones de uso del template DataAgro
> **Audiencia**: IA generadora de modelos funcionales (mockups HTML interactivos)  
> **Archivo base**: `template_dataagro.html`  
> **Propósito**: Construir pantallas funcionales para integración en ASP.NET MVC con `_Layout.cshtml`, con el diseño visual de DataAgro.

---

## 1. ¿Qué es este template?

Es un archivo HTML que puede usarse de dos formas:

### 1A. Como vista Razor en ASP.NET MVC (RECOMENDADO)
Crear un archivo `.cshtml` que use el **shared layout** y embeba el template:
```razor
@{
    ViewBag.Title = "Nombre de la Pantalla";
}

@section Styles {
    @Styles.Render("~/Content/mi-pantalla")
}

<div class="Titulo">
    <h5>Nombre de la Pantalla</h5>
</div>

<div class="body-content">
    <!-- Contenido del template adaptado aquí -->
</div>

@section Scripts {
    <!-- Scripts específicos de la pantalla -->
}
```

**Ventajas:**
- ✅ Hereda navbar, autenticación, permisos del `_Layout.cshtml`
- ✅ Mantiene consistencia visual con el resto del app
- ✅ Acceso a `@Url.Action()`, helpers, `ViewBag`, etc.
- ✅ CSS/JS globales del sitio cargados automáticamente

### 1B. Como HTML standalone (SOLO para prototipos iniciales)
Use `template_dataagro.html` directamente en el navegador sin backend — úselo **solo** para validación rápida con usuarios. **Antes de integrar en el app, convierta a vista Razor siguiendo el patrón 1A.**

---

## El template incluye:

- Bootstrap 3.4 + Font Awesome 4.7 (las mismas librerías que el app real)
- El CSS de `Site.css` y `SiteResponsive.css` embebido → las clases nativas del app ya están disponibles
- Componentes propios con prefijo `.da-*` para elementos nuevos (cards, tabs, stats, badges, modales)
- JavaScript utilitario listo (tabs, modales, filtros, export Excel, helpers de formato)

**Flujo de trabajo esperado:**
1. Copiar y adaptar el contenido del `template_dataagro.html` según la necesidad
2. Crear una vista `.cshtml` en `WebDataAgro/Views/MiControlador/Index.cshtml` que use `_Layout.cshtml`
3. Reemplazar todos los marcadores `[TODO]`
4. Implementar la lógica de datos en JS
5. El bloque `#panel-referencia` debe estar en el template pero NO en la vista final (eliminar antes de integrar)

---

## ⚠️ Reglas de uso de estilos (obligatorias)

### 1 — Solo incluir los estilos que el modelo usa
El template es un catálogo de opciones. Al generar una pantalla, **copiar únicamente los componentes HTML que esa pantalla realmente utiliza**. Si el modelo no tiene tabs, no agregar el bloque de tabs. Si no hay stats cards, no incluir esas secciones. No arrastrar markup de ejemplo que no cumpla ninguna función en el modelo entregado.

### 2 — No cambiar nombres de clases existentes
Las clases `.da-*` y las clases nativas de `Site.css` son **fijas e inamovibles**. Está prohibido:
- Renombrar: `da-btn-verde` → ~~`da-boton-verde`~~
- Crear alias: `da-card-blue` en lugar de `da-card-header--azul`
- Modificar las propiedades CSS de clases existentes dentro del `<style>` del modelo

Si el nombre de una clase no es correcto para el caso de uso, elegir la clase existente más apropiada del catálogo, no inventar una nueva con nombre similar.

### 3 — Si se necesita un estilo nuevo, declararlo explícitamente
Cuando ninguna clase del template cubre un requisito visual, el estilo nuevo debe:

1. Ir en un bloque `<style id="estilos-pantalla">` **separado** (nunca mezclarse con `<style id="site-css">` ni con el bloque de componentes `.da-*`)
2. Usar el prefijo `pant-` para diferenciarlo de los estilos del template
3. Llevar el comentario `/* NUEVO */` para identificarlo como adición propia de esta pantalla

**Ejemplo:**
```html
<style id="estilos-pantalla">
  /* NUEVO */
  .pant-semaforo { display: inline-block; width: 12px; height: 12px; border-radius: 50%; }
  .pant-semaforo--verde { background: #179e2b; }
  .pant-semaforo--rojo  { background: #d00707; }
</style>
```

Este bloque debe agregarse **después** del bloque `<style id="site-responsive-css">` y **antes** del `<body>`.

---

## 2. Paleta de colores y tokens de diseño

| Token CSS          | Valor     | Uso                                        |
|--------------------|-----------|--------------------------------------------|
| `--da-verde`       | `#017940` | Navbar, bordes de sección, botones primarios |
| `--da-verde-drk`   | `#046135` | Hover de botón verde                       |
| `--da-azul`        | `#26337B` | Botones secundarios, totalizadores, headers |
| `--da-azul-drk`    | `#151f52` | Hover de botón azul                        |
| `--da-amarillo`    | `#ffc100` | Estado "Pendiente"                         |
| `--da-rojo`        | `#d00707` | Estado "Error / Anulado"                   |
| `--da-fondo`       | `#f0f0f0` | Background del body                        |
| `--da-panel`       | `#ffffff` | Fondo de cards y paneles                   |
| `--da-borde`       | `#dedede` | Bordes de inputs y contenedores            |
| `--da-muted`       | `#787878` | Texto secundario                           |

---

## 3. Componentes disponibles

### 3.1 Encabezado de página
```html
<div class="da-titulo">
  <h4>Nombre de la pantalla</h4>
</div>
<br />
```

### 3.2 Barra de filtros desplegable
```html
<div class="da-filtro-barra" data-toggle="collapse" data-target="#filtrar"
     onclick="toggleFiltroLabel(this)">
  <div class="da-btn da-btn-out" style="display:inline-flex">
    <span class="da-filtro-lbl">Filtro</span>
    &nbsp;<strong><span class="da-filtro-toggle">Mostrar</span></strong>
  </div>
</div>

<div id="filtrar" class="collapse">
  <div class="da-panel-filtros">
    <div class="row da-fila-filtro">
      <div class="col-md-2"><label>Etiqueta</label></div>
      <div class="col-md-4">
        <input type="date" class="form-control" id="MiCampo" />
      </div>
      <!-- repetir pares col-md-2 + col-md-4 para más campos -->
    </div>
    <div class="row">
      <div class="col-md-12 da-r">
        <button class="da-btn da-btn-verde" onclick="buscar()">
          <i class="fa fa-search"></i> Buscar
        </button>
      </div>
    </div>
  </div>
</div>
```
**Tipos de input soportados:** `text`, `date`, `number`, `select` (con `class="form-control"`).

### 3.3 Tarjeta de sección / paso
```html
<div class="da-card" id="card-nombre">
  <div class="da-card-header da-card-header--verde">
    <i class="fa fa-list"></i> Título del paso
  </div>
  <div class="da-card-body">
    <!-- contenido -->
  </div>
</div>
```
| Clase de header        | Color de fondo | Cuándo usar                         |
|------------------------|---------------|-------------------------------------|
| `da-card-header--verde`  | `#017940`    | Paso principal, resultado, carga    |
| `da-card-header--azul`   | `#26337B`    | Configuración, parámetros, secundario |
| `da-card-header--amarillo`| `#E6A800`   | Alertas, advertencias               |

Para **ocultar/mostrar** una card en el flujo: agregar/quitar clase `da-hidden`.

### 3.4 Stats cards (KPIs de resumen)
```html
<div class="da-stats-grid">
  <div class="da-stat-card da-stat--verde">
    <div class="da-stat-lbl">Etiqueta</div>
    <div class="da-stat-val" id="stat-mi-id">0</div>
    <div class="da-stat-sub">unidad</div>
  </div>
</div>
```
| Variante           | Fondo      | Color valor | Usar para         |
|--------------------|------------|-------------|-------------------|
| `da-stat--verde`   | verde claro| `#017940`   | totales positivos |
| `da-stat--azul`    | azul claro | `#26337B`   | cantidades        |
| `da-stat--amarillo`| amarillo   | `#B08000`   | pendientes        |
| `da-stat--rojo`    | rojo claro | `#d00707`   | errores           |

### 3.5 Tabs
```html
<div class="da-tabs-bar">
  <div class="da-tab active" onclick="switchTab('tab-uno', this)">Pestaña 1</div>
  <div class="da-tab"        onclick="switchTab('tab-dos', this)">Pestaña 2</div>
</div>
<div id="tab-uno" class="da-tab-pane active"> <!-- contenido --> </div>
<div id="tab-dos" class="da-tab-pane">        <!-- contenido --> </div>
```
`switchTab(tabId, elemento)` ya está implementado en el template.

### 3.6 Tabla con header verde
```html
<div class="da-table-wrap">
  <table class="da-table" id="mi-tabla">
    <thead>
      <tr>
        <th>Columna texto</th>
        <th class="r">Columna numérica</th>
        <th>Estado</th>
      </tr>
    </thead>
    <tbody id="tbody-mi-tabla">
      <!-- filas generadas por JS -->
    </tbody>
  </table>
</div>
```
Clases de celda: `td.r` (alinear derecha), `td.bold` (negrita), `td.muted` (gris chico).  
Para header azul: agregar clase `da-table--azul` a la tabla.

**Patrón para generar filas en JS:**
```javascript
function renderTabla(datos) {
  const tbody = document.getElementById('tbody-mi-tabla');
  tbody.innerHTML = datos.map(d => `
    <tr>
      <td class="bold">${esc(d.contrato)}</td>
      <td class="r">${fmtNum(d.toneladas)}</td>
      <td>${badgeEstado(d.estado)}</td>
    </tr>
  `).join('');
}
```

### 3.7 Badges de estado
```html
<!-- Template (da-badge): -->
<span class="da-badge da-badge--pendiente">Pendiente</span>
<span class="da-badge da-badge--confirmado">Confirmado</span>
<span class="da-badge da-badge--finalizado">Finalizado</span>
<span class="da-badge da-badge--error">Error</span>
<span class="da-badge da-badge--gris">Anulado</span>
<span class="da-badge da-badge--azul">Info</span>
<span class="da-badge da-badge--violeta">Reconfirmar</span>
```
O usar el helper JS: `badgeEstado('Confirmado')` → devuelve el HTML del badge correcto.

### 3.8 Botones
```html
<!-- Primario (verde) -->
<button class="da-btn da-btn-verde" onclick="miFuncion()">
  <i class="fa fa-play"></i> Ejecutar
</button>

<!-- Secundario (azul) -->
<button class="da-btn da-btn-azul"><i class="fa fa-cog"></i> Configurar</button>

<!-- Outline -->
<button class="da-btn da-btn-out"><i class="fa fa-times"></i> Cancelar</button>

<!-- Excel export -->
<button class="da-btn da-btn-out da-btn-excel" onclick="exportExcel()">
  <i class="fa fa-file-excel-o"></i> Exportar Excel
</button>
```
Agrupar botones en una fila: envolverlos en `<div class="da-btn-row">`.

También disponibles las clases nativas del app (de `Site.css`, ya embebido):
- `.btnAzul` → `<span class="btnAzul">Crear Negocio</span>`
- `.btnVerde` → `<span class="btnVerde">Guardar</span>`
- `.formulario-footer-cancelar` → link cancelar azul

### 3.9 Modal
```html
<div id="modal-mi-modal" class="da-modal-overlay" style="display:none"
     onclick="cerrarModalClick(event, 'modal-mi-modal')">
  <div class="da-modal-box">
    <div class="da-modal-header da-card-header--verde">
      <span>Título del modal</span>
      <button class="da-modal-close" onclick="cerrarModal('modal-mi-modal')">✕</button>
    </div>
    <div class="da-modal-body">
      <!-- contenido del modal -->
    </div>
    <div class="da-modal-footer">
      <button class="da-btn da-btn-verde" onclick="confirmar()">Confirmar</button>
      <button class="da-btn da-btn-out" onclick="cerrarModal('modal-mi-modal')">Cancelar</button>
    </div>
  </div>
</div>
```
Abrir desde JS: `abrirModal('modal-mi-modal')` — función ya implementada.

### 3.10 Totalizadores de materiales
```html
<div class="da-totales">
  <div class="da-total-item">
    <span class="da-total-lbl">Soja</span>
    <strong><span class="da-total-val" id="tot-soja">0</span> TN.</strong>
  </div>
  <div class="da-total-item">
    <span class="da-total-lbl">Maíz</span>
    <strong><span class="da-total-val" id="tot-maiz">0</span> TN.</strong>
  </div>
</div>
```

### 3.11 Puntos de color por material
```html
<span class="da-dot da-dot--soja"></span>Soja
<span class="da-dot da-dot--maiz"></span>Maíz
<span class="da-dot da-dot--trigo"></span>Trigo
<span class="da-dot da-dot--otro"></span>Otro
```

---

## 4. Funciones JS ya implementadas

| Función                              | Descripción                                                    |
|--------------------------------------|----------------------------------------------------------------|
| `toggleFiltroLabel(barEl)`           | Alterna "Mostrar" / "Ocultar" en la barra de filtros           |
| `switchTab(tabId, clickedEl)`        | Cambia pestaña activa en un grupo de tabs                      |
| `abrirModal(id)`                     | Abre un modal por su id                                        |
| `cerrarModal(id)`                    | Cierra un modal por su id                                      |
| `cerrarModalClick(event, id)`        | Cierra modal al hacer clic en el overlay                       |
| `filtrarTabla(tbodyId, query)`       | Filtra filas de una tabla por texto                            |
| `pasarAPaso2()`                      | Oculta card-paso1, muestra card-paso2                          |
| `volverAPaso1()`                     | Oculta card-paso2, muestra card-paso1                          |
| `mostrarResultado(data)`             | Oculta paso2, muestra stats-section y card-resultado           |
| `limpiarFiltros()`                   | Limpia los 4 campos de filtro del template base                |
| `exportExcel()`                      | Exporta la variable `filas` a `.xlsx` via SheetJS              |
| `fmtNum(n)`                          | Formatea número con separador de miles `es-AR`                 |
| `fmtFecha(d)`                        | Formatea `Date` → `dd/mm/yyyy`                                 |
| `esc(s)`                             | Escapa HTML para inserción segura en `innerHTML`               |
| `badgeEstado(estado)`                | Devuelve HTML del badge `da-badge` correspondiente al estado   |

**Siempre usar `esc()` al insertar strings de datos en `innerHTML`** para evitar XSS.

---

## 5. Patrón de flujo en múltiples pasos

El template implementa el patrón "wizard": una card por paso, visible de a una.

```
[card-paso1] → pasarAPaso2() → [card-paso2] → ejecutar() → mostrarResultado() → [card-resultado]
                              ← volverAPaso1() ←
```

Para más de 2 pasos, duplicar las cards y crear las funciones de navegación siguiendo el mismo patrón:
```javascript
function pasarAPaso3() {
  document.getElementById('card-paso2').classList.add('da-hidden');
  document.getElementById('card-paso3').classList.remove('da-hidden');
}
```

---

## 6. Cómo agregar más tabs al resultado

```javascript
// En renderResults(), después de crear da-tabs-bar:
// 1. Agregar div en la barra:
tabsBar.innerHTML += `
  <div class="da-tab" onclick="switchTab('tab-nuevo', this)">Nueva pestaña</div>
`;

// 2. Agregar panel después del último da-tab-pane existente:
const pane = document.createElement('div');
pane.id = 'tab-nuevo';
pane.className = 'da-tab-pane';
pane.innerHTML = '...contenido...';
tabsBar.parentNode.appendChild(pane);
```

---

## 7. Export a Excel

La función `exportExcel()` usa SheetJS. Personalizar el contenido de `filas`:

```javascript
function exportExcel() {
  const filas = [
    ['Contrato', 'CUIT', 'Material', 'Toneladas', 'Estado']  // encabezado
  ];
  datos.forEach(d => {
    filas.push([d.contrato, d.cuit, d.material, d.toneladas, d.estado]);
  });

  const ws = XLSX.utils.aoa_to_sheet(filas);
  // Ancho de columnas (opcional):
  ws['!cols'] = [{wch:12},{wch:13},{wch:10},{wch:12},{wch:12}];

  const wb = XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(wb, ws, 'Resultado');
  const hoy = new Date().toISOString().slice(0,10).replace(/-/g,'');
  XLSX.writeFile(wb, `resultado_${hoy}.xlsx`);
}
```

---

## 8. Clases nativas del app disponibles en el template

Estas clases de `Site.css` están embebidas y pueden usarse directamente:

| Clase                        | Descripción                                         |
|------------------------------|-----------------------------------------------------|
| `.btnAzul`                   | Botón azul `#26337B`, radio 5px                     |
| `.btnVerde`                  | Botón verde `#017940`                               |
| `.btnagregar`                | Igual a `.btnAzul`, con margin-bottom               |
| `.formulario-footer-cancelar`| Link cancelar azul, subrayado en hover              |
| `.float-left` / `.float-right`| Floats de layout                                   |
| `.noPadding`                 | Quita padding lateral                               |
| `.noPaddingLeft`             | Quita padding izquierdo                             |
| `.noPaddingRight`            | Quita padding derecho                               |
| `.buscar-nomb`               | Texto azul bold mayúsculas (resultado de búsqueda)  |
| `.cant-notif`                | Badge circular rojo (cantidad)                      |
| `.opacidad`                  | Animación fade-in 0.4s                              |
| `.alertBox`                  | Alerta verde DataAgro (fondo `#017940`, texto blanco)|
| `.puntero`                   | `cursor: pointer`                                   |
| `.page-footer`               | Pie de página con borde superior verde              |

---

## 9. Checklist antes de entregar el modelo

**Contenido**
- [ ] Todos los `[TODO]` reemplazados
- [ ] `<title>` actualizado con el nombre de la pantalla
- [ ] El bloque `<div id="panel-referencia">` eliminado del HTML
- [ ] Solo están presentes los componentes HTML que el modelo realmente usa (sin bloques de ejemplo sobrantes)
- [ ] Las funciones `buscar()`, `ejecutar()`, `confirmarModal()` implementadas (o eliminadas si no aplican)
- [ ] `limpiarFiltros()` actualizada con los IDs reales de los filtros
- [ ] `exportExcel()` actualizada con las columnas del proceso
- [ ] Toda inserción de datos en `innerHTML` usa `esc()` o `fmtNum()`/`fmtFecha()`
- [ ] Los IDs de stats (`stat-contratos`, etc.) actualizados y seteados en `mostrarResultado()`

**Estilos**
- [ ] No se renombró ni modificó ninguna clase `.da-*` ni clase nativa de `Site.css`
- [ ] Si se crearon estilos nuevos: están en `<style id="estilos-pantalla">`, usan prefijo `pant-`, y tienen comentario `/* NUEVO */`
- [ ] No hay clases nuevas con nombres similares a los existentes (ej: `da-boton-verde`, `da-card-blue`)

---

## 10. Instrucción resumida para generar una pantalla nueva

Cuando el usuario describa una pantalla, seguir este proceso:

1. **Crear vista Razor** en `WebDataAgro/Views/MiControlador/Index.cshtml` (o el controlador correspondiente).
2. **Usar `_Layout.cshtml`** — NO usar `Layout = null`. La vista hereda navbar, estilos globales y scripts del layout.
3. **Estructura base:**
   ```razor
   @{
       ViewBag.Title = "Nombre de Pantalla";
   }

   @section Styles {
       @Styles.Render("~/Content/mi-pantalla-css")
   }

   <div class="Titulo">
       <h5>Nombre de la Pantalla</h5>
   </div>

   <div class="body-content">
       <!-- Contenido adaptado del template aquí -->
   </div>

   @section Scripts {
       <script src="~/Scripts/Controlador/mi-pantalla.js"></script>
   }
   ```
4. **Definir la estructura**: ¿cuántos pasos tiene el flujo? ¿hay filtros? ¿resultado en tabla? — **Solo conservar los bloques HTML que apliquen; eliminar los demás.**
5. **Completar el encabezado**: título, scripts a agregar si hay lógica adicional.
6. **Adaptar los filtros**: campos de fecha, material, centro u otros según la pantalla.
7. **Adaptar las cards de pasos**: headers (`--verde` o `--azul`), títulos, campos.
8. **Definir las columnas de la tabla de resultado** según los datos del proceso.
9. **Definir las stats cards**: qué KPIs mostrar (total, pendientes, errores, etc.).
10. **Implementar la lógica en JS**: `buscar()`, `ejecutar()`, `mostrarResultado()`.
11. **Implementar `exportExcel()`** con las columnas del proceso.
12. **Verificar estilos**: 
    - Hereda colores y fonts de `Site.css` (NO redefinir)
    - No hay clases renombradas; los estilos nuevos (si los hay) están en un CSS separado con prefijo `pant-` y marcados `/* NUEVO */`
    - El padding de `.body-content` es automático (no duplicar espacios)
