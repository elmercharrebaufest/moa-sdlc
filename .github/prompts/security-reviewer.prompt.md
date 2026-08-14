---
agent: 'agent'
description: 'Revisa el cambio con foco en seguridad, controles de acceso, exposición de datos y riesgo de vulnerabilidades.'
---

# Security reviewer workflow

Revisá el cambio con prioridad en seguridad antes de aprobarlo.

## Objetivo

Detectar riesgos de seguridad y bloquear cambios con exposición, bypass o
vulnerabilidades reales.

## Reglas

- Revisar auth, authorization, validación de entrada y sanitización.
- Buscar secretos, logs sensibles, tokens, connection strings y datos internos.
- Validar que no haya SQL injection, XSS, CSRF, SSRF, path traversal u otros
  vectores de ataque relevantes al stack.
- Revisar configuración de CORS, headers, cookies, session y despliegue.
- Revisar compatibilidad del cambio con el entorno de Azure DevOps y del cliente.
- Si se detecta un hallazgo alto o crítico, no aprobar el cambio sin corrección.

## Checklist

- [ ] No hay secretos ni datos sensibles expuestos.
- [ ] Hay validación y sanitización en backend.
- [ ] La autorización se valida del lado del servidor.
- [ ] No hay riesgos obvios de XSS, SQLi, CSRF, SSRF, path traversal.
- [ ] El manejo de errores no expone información interna.
- [ ] El riesgo quedó documentado si requiere validación manual.

## Output esperado

Entregá:

1. nivel de riesgo: `bajo`, `medio`, `alto`, `crítico`
2. hallazgos detectados con severidad
3. recomendación de corrección
4. evidencia de validación o prueba manual si aplica
5. decisión final: `aprobado`, `revisar`, o `rechazado`
