# Actividad 4 — Reducir el volumen con un filtro

## Preguntas

1. **¿Qué perdiste de evidencia al aplicar el filtro? ¿Qué ganaste?**
   * **Qué se perdió:** Se perdió la trazabilidad de todas las operaciones de lectura (`query`, como consultas `find` sobre datos personales o RFC) y de los comandos administrativos o de agregación (`command`), ya que el filtro únicamente captura escrituras (`insert`, `update`, `remove`). Además, al definir una expresión `filter`, MongoDB ignora los parámetros `slowms` y `sampleRate`, por lo que tampoco se registran consultas de lectura lentas.
   * **Qué se ganó:** Se redujo drásticamente el volumen de documentos escritos en la colección limitada `system.profile` (1 MB por defecto) y la sobrecarga de E/S en disco del servidor. Esto evita que las consultas cotidianas de lectura sobrescriban rápidamente los registros de modificaciones críticas (`insert`, `update`, `remove`), prolongando la retención de la evidencia de cambios de datos.

2. **¿Qué filtro usarías si solo interesaran operaciones sobre la colección `clientes`?**
   Usaría un filtro sobre el campo `ns` (espacio de nombres completo `base.coleccion`) configurado en nivel 1:
   ```javascript
   db.setProfilingLevel(1, { filter: { ns: "comercial.clientes" } })

   