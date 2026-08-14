---
applyTo: "**/*.html,**/*.cshtml"
---

# Guía para nuevas pantallas funcionales (mockups HTML)

Para construir una pantalla nueva en el estilo visual de DataAgro, seguir las
instrucciones detalladas en:

- **Template base:** [`.github/instructions/template_dataagro.html`](template_dataagro.html)
- **Manual de uso:** [`.github/instructions/template_dataagro_instrucciones.md`](template_dataagro_instrucciones.md)

## Pasos resumidos

1. Copiar `template_dataagro.html` con el nombre de la pantalla nueva.
2. Seguir el proceso del **§10 "Instrucción resumida"** de `template_dataagro_instrucciones.md`.
3. Usar únicamente los componentes HTML que la pantalla realmente necesita — no arrastrar bloques de ejemplo sobrantes.
4. Respetar las reglas de estilos del **§ "Reglas de uso de estilos"**:
   - No renombrar clases `.da-*` ni clases nativas de `Site.css`.
   - Estilos nuevos → bloque `<style id="estilos-pantalla">`, prefijo `pant-`, comentario `/* NUEVO */`.
5. Insertar datos en `innerHTML` siempre con `esc()` (ver §4 de las instrucciones).
6. Eliminar `<div id="panel-referencia">` antes de entregar.
7. Verificar el checklist del §9 antes de dar por terminada la pantalla.
