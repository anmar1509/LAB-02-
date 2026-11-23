# Actividad 1 — Lo que no existe en esta edición

1. **¿Qué edición de MongoDB estás usando? ¿Qué evidencia lo demuestra?**
   Estoy utilizando **MongoDB 8.0.5 Community Edition**. La evidencia técnica es el resultado de `db.adminCommand({ buildInfo: 1 }).modules`, que devuelve un arreglo vacío (`[]`); en MongoDB Enterprise ese arreglo incluiría `"enterprise"`.

2. **¿A dónde escribe tu servidor su bitácora? ¿Cómo lo sabes?**
   La escribe en el archivo `C:\Program Files\MongoDB\Server\8.0\log\mongod.log` con modo `destination: 'file'` y `logAppend: true`. Lo sé mediante la sección `systemLog` de `db.serverCmdLineOpts().parsed`.

3. **Según REF-U3-04, ¿qué cuatro categorías de eventos registraría la auditoría nativa?**
   * Cambios de esquema (DDL).
   * Eventos de conjuntos de réplicas (*replica sets*) y clústeres fragmentados (*sharded clusters*).
   * Eventos de autenticación y autorización.
   * Operaciones CRUD (requiere habilitar `auditAuthorizationSuccess: true`).

4. **¿Qué tendría que evaluar una organización antes de licenciar Enterprise o migrar a Atlas solo para obtener auditoría?**
   * **Costo-beneficio:** El gasto recurrente de licencias Enterprise o clústeres dedicados (M10+) en Atlas frente al valor de los datos protegidos.
   * **Cumplimiento normativo:** Si regulaciones externas (PCI-DSS, HIPAA, GDPR, etc.) exigen pistas de auditoría completas e inmutables.
   * **Suficiencia de controles compensatorios:** Si la bitácora de `mongod`, el perfilador filtrado y las instantáneas periódicas de privilegios cubren los riesgos operativos actuales.
   * **Impacto operativo:** Almacenamiento requerido para los registros de auditoría y sobrecarga de rendimiento al auditar operaciones de lectura/escritura.