# Technical Design - <Feature Name>

## 1. Mapa de Impacto Técnico

### 📂 Entidades y Datos
- **Entidad/DTO:** `Molinos.DataAgro.Entities/<Nombre>.cs`
- **Cambio en el DbContext:** `Molinos.DataAgro.Repository/DataAgroDbContext.cs`
  (solo si aplica; migraciones de EF requieren aprobación humana, ver AGENTS.md)

### 📂 Lógica de Negocio
- **Manager:** `Molinos.DataAgro.Business/Managers/<Nombre>Manager.cs`
  (implementa `I<Nombre>Manager` en `Molinos.DataAgro.Interfaces`,
  auto-registrado por Autofac)

### 📂 Acceso a Datos
- **Repositorio:** uso de `IRepositorio` / `RepositorioEF`
  (`Molinos.DataAgro.Repository`)

### 📂 Web (MVC + Kendo UI)
- **Controlador:** `WebDataAgro/Controllers/<Nombre>Controller.cs`
  (Acción: `<Verbo> /<Controller>/<Action>`)
- **Vista:** `WebDataAgro/Views/<Nombre>/<Accion>.cshtml`
  (Kendo Grid / jQuery AJAX si aplica)

## 2. Contrato de Datos

### Modelo de entrada (ViewModel / parámetros de acción)
```json
{
  "campo1": "tipo",
  "campo2": "tipo"
}
```

### Modelo de salida
```json
{
  "campo1": "tipo"
}
```

## 3. Estrategia de Pruebas
- Tests NUnit en `Molinos.DataAgro.Test`, con Moq para mockear
  `Manager`/`IRepositorio` según el patrón existente en el proyecto.
- Flujos de UI (Kendo/vistas MVC) que no se puedan cubrir con NUnit se
  documentan como requisito tipo `manual` en `requirements.md`.
