---
agent: 'agent'
description: 'Revisa la calidad del código cambiado y valida que cumpla los estándares de seguridad, mantenimiento y prueba antes de aprobarlo.'
---

# Quality gate review

Revisá el diff o los archivos modificados en el branch actual.

## Objetivo

Validar que el código cumpla con la arquitectura, pruebas y estándares de 
calidad del proyecto antes de considerarlo listo para merge o aprobación.

## Checklist obligatoria

- [ ] El cambio resuelve el requisito sin introducir regresiones.
- [ ] Hay pruebas automatizadas o validación manual documentada para la lógica nueva.
- [ ] No hay riesgos de seguridad evidentes (SQLi, XSS, CSRF, secret exposure, auth bypass).
- [ ] No hay manejo de errores inapropiado ni `catch` vacío.
- [ ] El código es legible, mantenible y consistente con el patrón del proyecto.
- [ ] No hay duplicación innecesaria ni complejidad artificial.
- [ ] El impacto en frontend/backend está claramente definido.
- [ ] Los cambios de base de datos, APIs o contratos fueron revisados con cuidado.

## Output esperado

Entregá un resumen con:

1. Estado general: `aprobado`, `revisar`, o `rechazado`
2. Hallazgos de seguridad y calidad con severidad (`Alta`, `Media`, `Baja`)
3. Evidencia concreta del archivo/línea o del patrón detectado
4. Recomendaciones concretas y priorizadas
5. Lista de pruebas que faltan o deben ejecutarse

Si hay un hallazgo de severidad alta o crítica, no lo desestimes como ruido;
debe quedar como bloqueo explicativo para aprobación.
