---
applyTo: "**/*.{cs,cshtml,razor,ts,tsx,js,html,csproj,sln,yml,yaml,json,xml}"
---

# Security review instructions for Copilot

Usar esta guía para revisar cualquier cambio con foco en seguridad, control de
acceso, integridad y riesgo de exposición de datos. Es un gate obligatorio antes
de aprobar cambios de backend, frontend, APIs, autenticación, autorización,
configuración de infraestructura o despliegue.

## Checklist de seguridad

- Verificar que no haya exposición accidental de secretos, tokens, claves,
  connection strings, credenciales o datos sensibles en código, logs, mensajes,
  traces, UI, comentarios o archivos temporales.
- Revisar autenticación y autorización: endpoints públicos, bypass, falta de
  verificaciones en backend, roles insuficientes y privilegios no esperados.
- Revisar validación de entrada: SQL injection, command injection, XSS,
  deserialización insegura, path traversal, SSRF, open redirects y abuso de
  rutas o query params.
- Revisar manejo de errores: no filtrar información interna, no exponer stack
  traces, no dejar logs con detalles sensibles ni errores demasiado verbosos.
- Revisar control de sesión y tokens: expiración, refresh, rotación, expiración
  de cookies, CSRF, anti-forgery y validación de origen.
- Revisar serialización y parsing: JSON deserialization insegura, tipos
  dinámicos, uso de `unsafe`, `Eval`, `innerHTML`, DOM injection o abuso de
  expresiones evaluadas.
- Revisar configuración y despliegue: CORS demasiado amplio, headers inseguros,
  debug habilitado, endpoints de diagnóstico expuestos, variables de ambiente
  mal administradas y entornos incorrectos.
- Revisar cumplimiento mínimo de arquitectura segura: separación de roles,
  privilegios mínimos, validación en backend, no confiar en cliente como control
  de seguridad.

## Riesgos específicos por stack

### .NET / ASP.NET / MVC / Razor

- No confiar solo en validación del cliente.
- Revisar `[Authorize]`, claims, roles, policies, antiforgery y configuración de
  cookies/auth.
- Verificar que no se expongan endpoints sin control ni archivos sensibles.
- Evitar concatenar SQL con entrada del usuario.

### Angular / frontend JS

- No usar `innerHTML`, `dangerouslySetInnerHTML`, `eval`, `Function(...)` ni
  manipulación directa de DOM con contenido no sanitizado.
- Validar que los tokens se manejen en almacenamiento seguro y no por default en
  localStorage cuando el proyecto lo prohibe.
- Revisar interceptors, guards y rutas protegidas.

### Azure DevOps / CI/CD

- No incluir secretos en pipeline YAML ni en variables compartidas visibles.
- Revisar que los despliegues no usen permisos excesivos ni entornos no
  autorizados.
- Validar que los jobs de build/test no ejecuten scripts con permisos de alto
  riesgo.

## Revisión obligatoria antes de aprobar

Antes de aprobar un cambio, responder estas preguntas:

1. ¿Se trata de una exposición de secretos o datos sensibles?
2. ¿La autenticación/autorización se valida en backend?
3. ¿El input del usuario se valida y sanitiza?
4. ¿Hay algún vector de XSS/SQLi/CSRF/SSRF o abuso de rutas?
5. ¿El manejo de errores no expone información interna?
6. ¿Hay una posibilidad real de bypass de controles?
7. ¿Se documentó el riesgo y se dejó evidencia de validación/manual QA?

Si alguna respuesta es sí o hay dudas razonables, el cambio debe quedar en
estado de `revisar` o `rechazado` y no aprobarse sin evidencia adicional.

## Resultado esperado

Entregar un resumen con:

- riesgo general: `bajo`, `medio`, `alto`, `crítico`
- hallazgos concretos con archivo, patrón y recomendación
- acciones correctivas inmediatas
- si aplica, una nota de validación manual o de prueba requerida

No aprobar un cambio con hallazgos de seguridad alta o crítica sin evidencia de
corrección y revisión humana explícita.
