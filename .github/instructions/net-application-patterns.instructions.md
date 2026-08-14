---
applyTo: "**/*.cs,**/*.csproj,**/*.sln,**/*.cshtml,**/*.razor,**/*.ts,**/*.tsx,**/*.js,**/*.html"
---

# .NET application patterns for Copilot agents

Aplica a proyectos .NET Framework, .NET Core, ASP.NET MVC, ASP.NET Core MVC,
Razor, Angular, SPA y mix de frontend legado.

## Principios generales

- Respetar la arquitectura existente del proyecto: capas, inyección de
  dependencias, servicio/manager/repository, separación de responsabilidades.
- Para .NET Framework legacy: evitar introducir patrones modernos cuando la base
  ya usa `Web.config`, `Global.asax`, `HttpContext`, MVC 5 o Kendo/jQuery.
- Para .NET actual: preferir `IServiceCollection`, `AddScoped/AddTransient`,
  `async`/`await`, `Task`, `CancellationToken`, `ILogger`, configuraciones por
  entorno y `nullable` si el proyecto lo permite.
- Mantener el cambio mínimo necesario para resolver la historia o ticket.
- Si el proyecto usa una capa de negocio, no mezclar lógica de acceso a datos con
  la lógica de presentación ni con la UI.
- Usar principios de Clean Code sin dogmatismo: nombres claros, funciones con
  responsabilidad única, evitación de duplicación significativa, y métodos
  legibles, pero sin introducir complejidad artificial ni refactors grandes por
  estilo.
- Usar patrones de diseño y arquitectura cuando aporten claridad, testabilidad y
  mantenimiento: Factory, Strategy, Repository, Unit of Work, Adapter, Decorator,
  CQRS/Command-Query cuando sea razonable, y una orientación a Clean Architecture
  cuando la complejidad lo justifique. No forzar patrones innecesarios en casos
  simples.
- Favorecer una arquitectura en capas con dependencia de abstracciones hacia
  adentro: entidades y reglas del dominio no dependen de infraestructura ni de
  UI; infraestructura, persistencia y presentación se adaptan a los casos de uso.

## Reglas de negocio y calidad

- Generar tests para toda lógica de negocio nueva o modificada.
- Cubrir casos principales y edge cases: validación nula, entradas vacías,
  valores inválidos, errores de infraestructura, casos borde de concurrencia.
- Usar `try/catch` solo alrededor de puntos de integración o llamadas externas;
  no ocultar errores ni convertir excepciones en respuestas ambiguas.
- No guardar secretos, tokens, strings de conexión ni credenciales en código,
  archivos temporales, logs o mensajes de error.
- Validar autenticación/autorización antes de procesar operaciones sensibles.
- En ASP.NET MVC / Razor / ASP.NET Core, usar antiforgery, model binding,
  sanitización y validación de entrada según el patrón del proyecto.

## Frontend guidance

- Si el proyecto es legacy MVC/Kendo: seguir el estilo actual, evitar reescribir
  a un enfoque moderno cuando el patrón existente es estable.
- Si el proyecto usa Angular o SPA: mantener servicios, models y componentes
  separados; preferir pipe/guard/interceptor reutilizables y validación en
  cliente solo como UX, nunca como seguridad.
- En cualquier frontend, no confiar solo en validación del cliente para seguridad;
  validar siempre en backend.

## Data access and APIs

- No construir SQL concatenado con entrada del usuario; usar parámetros,
  consultas parametrizadas o ORMs seguros.
- Evitar N+1 queries, loops masivos, materialización innecesaria de colecciones
  grandes y conversiones repetidas de tipos.
- En APIs, definir contratos claros con request/response y errors consistentes.
- En apps legacy, mantener compatibilidad con el contrato actual, firma de
  controladores y modelo de rutas si no hay aprobación para romperlo.

## Code review gate before completion

El cambio no debe considerarse listo si:

- presenta warnings de seguridad, NRE, nullability, desbordes, lógica no testeada,
  o complejidad innecesaria;
- repite lógica o introduce deuda técnica sin justificación;
- no documenta los supuestos de diseño, restricciones del stack o casos manuales;
- deja tests faltantes o sin explicación de cobertura.

Cuando un cambio involucra UI legacy o Angular, priorizar compatibilidad visual,
patrones del proyecto y comportamiento no regresivo.
