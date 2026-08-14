# SonarQube quality review skill

## Propósito

Usar esta guía para revisar y calificar el código agregado antes de que pase por
análisis estático o SonarQube.

## Enfoque

- Seguridad: autenticación, autorización, validación, entrada del usuario,
  secretos y manejo de errores.
- Fiabilidad: nulos, excepciones, concurrencia, recursos y condiciones de borde.
- Mantenibilidad: complejidad, deuda técnica, nombres, duplicación y acoplamiento.
- Rendimiento: consultas costosas, loops, serialización, uso de memoria.
- Tests: cobertura razonable y casos de falla documentados.

## Criterio de salida

La revisión solo debe ser aprobada si:

- no hay bugs obvios ni riesgos de seguridad;
- la cobertura es suficiente para la lógica nueva;
- la implementación se alinea con las reglas del proyecto y del stack elegido;
- se documentan claramente los riesgos o pendientes manuales.

Este artefacto actúa como pre-gate de calidad antes del análisis estático del
equipo o SonarQube.
