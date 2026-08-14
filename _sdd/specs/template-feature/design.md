# Technical Design - <Feature Name>

## 1. Mapa de Impacto Técnico

### 📂 Entidades y Datos
- **Entidad/DTO:** `Entities/<Nombre>.cs` o `Models/<Nombre>.cs`
- **Cambio en el DbContext / ORM:** `Repository/<Context>.cs` o la capa de acceso
  correspondiente.
  (solo si aplica; migraciones de EF requieren aprobación humana, ver AGENTS.md)

### 📂 Lógica de Negocio
- **Servicio / Manager / UseCase:** `Business/<Nombre>Manager.cs` o
  `Application/<Nombre>Service.cs`.
- **Contratos de interfaz:** `Interfaces/I<Nombre>Manager.cs` / `Contracts`.
- **Patrón de inyección de dependencias**: constructor injection, no new
  instancias directas dentro de la capa de negocio.

### 📂 Acceso a Datos
- **Repositorio / DAO:** uso de EF, Dapper, ADO.NET o `IRepositorio` según la
  tecnología adoptada por el proyecto.
- **Abstracción:** evitar lógica de persistencia mezclada con la capa web o UI.

### 📂 Web / Frontend
- **MVC / Razor / ASP.NET Core:** `Controllers/<Nombre>Controller.cs`,
  `Views/<Nombre>/<Accion>.cshtml` o `Pages/...`.
- **Angular / SPA:** `components`, `services`, `models`, `guards`, `interceptors`.
- **Legacy jQuery/Kendo**: mantener patrones actuales del proyecto; si se
  usa, documentar la ruta de serialización, AJAX y validación.

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
- Tests unitarios y de integración según el stack del proyecto: NUnit/XUnit/
  MSTest para .NET, además de tests de UI o E2E para frontend si aplica.
- Usar mocks / stubs para capas externas (`IRepositorio`, `HttpClient`,
  `IService`, `DataContext`) y mantener los tests enfocados en la lógica.
- Flujos de UI (Kendo, MVC legacy, Razor, Angular) que no se puedan cubrir
  automáticamente se documentan como requisito tipo `manual` en
  `requirements.md`.
- Registrar en `feature.json` la trazabilidad R<n> → tests y QA manual.
