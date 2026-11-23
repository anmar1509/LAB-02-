# Actividad 6 — Correlacionar un cambio de configuración con su autor

## Preguntas

1. **¿Qué usuario cambió la configuración del perfilador? ¿Qué dos eventos lo demuestran?**
   El usuario fue **`admin`** (autenticado en la base `admin`). Lo demuestran dos eventos de la bitácora de `mongod` vinculados mediante el mismo identificador de conexión (`ctx: 'conn28'`):
   * **Primer evento (`ACCESS` — `Successfully authenticated`):** Registra que la conexión `conn28` desde `127.0.0.1` se autenticó exitosamente con `user: 'admin'` y `db: 'admin'` antes de la modificación.
   * **Segundo evento (`COMMAND` — `Profiler settings changed`):** Registra que esa misma conexión `conn28` ejecutó los cambios de configuración del perfilador en la base `comercial` (incluyendo el paso de `level: 0` a `level: 1` con filtro).

2. **¿Qué condición debe cumplirse para que esta correlación sea válida? (Piensa en qué pasa si el servidor se reinicia y los números de conexión vuelven a empezar.)**
   La correlación únicamente es válida dentro de un **mismo ciclo de vida continuo del proceso `mongod` (sin reinicios entre ambos eventos)** y verificando que el evento de autenticación sea **anterior y cronológicamente el más cercano** al evento de cambio en la línea de tiempo (`t`). Si el servidor `mongod` se reinicia, el contador interno de conexiones vuelve a empezar desde `conn1`, por lo que un mismo `ctx` en diferentes arranques pertenecería a sesiones y usuarios completamente distintos.

3. **¿Por qué es relevante que quede registrado un cambio de configuración del propio mecanismo de registro?**
   Porque protege la **integridad y confiabilidad de la pista de trazabilidad**. Si un usuario con privilegios o un atacante apaga el perfilador (`setProfilingLevel(0)`) o altera sus filtros antes de realizar una acción maliciosa para encubrir sus huellas, este registro permite detectar la manipulación, identificar quién deshabilitó la vigilancia y determinar en qué ventana de tiempo hubo ceguera operativa.