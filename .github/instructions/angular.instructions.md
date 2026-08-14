---
applyTo: "**/*.ts,**/*.tsx,**/*.js,**/*.html,**/*.scss,**/*.css"
---

# Angular and SPA guidance

Aplicar cuando el frontend use Angular, componentes, services, RxJS y APIs.

## Principios

- Mantener arquitectura por feature o por dominio cuando la app lo requiera.
- Separar componentes, servicios, modelos, guards, interceptors y state.
- No mezclar lógica de UI con acceso a datos ni reglas de negocio.
- Usar observables y RxJS con criterio; no crear pipelines complejas sin
  necesidad.

## Seguridad y validación

- Validar inputs en backend; la validación del cliente es UX, no seguridad.
- Sanitizar datos y evitar `innerHTML` directo en templates o lógica de UI.
- Tratar errores de API de forma consistente y no exponer detalles sensibles en
  la UI.

## Calidad

- Tests unitarios para servicios, pipes, guards y lógica de componentes.
- Usar mocks para HTTP y state; validar casos de éxito, error y edge cases.
- Mantener el componente simple y reutilizable, con estilos y lógica bien
  separados.
