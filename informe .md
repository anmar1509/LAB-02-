# Informe Técnico — LAB-02: Trazabilidad en MongoDB Community

## 1. Despliegue, edición y limitación
* **Despliegue y versión:** Servidor MongoDB 7 Community Edition desplegado de forma nativa en Windows (servicio `MongoDB`, con control de acceso habilitado mediante `security.authorization: enabled` en `mongod.cfg`).
* **Verificación de edición:** Al ejecutar `db.adminCommand({ buildInfo: 1 }).modules` en `mongosh`, el arreglo devuelto fue vacío (`[]`), confirmando que se trata de **MongoDB Community Edition** (en la edición Enterprise este arreglo incluiría `"enterprise"`).
* **Limitación técnica verificada:** MongoDB Community Edition **no incluye auditoría nativa** (`--auditDestination`, auditoría de DDL, autorización o CRUD). Los únicos mecanismos integrados disponibles son el perfilador de base de datos (`system.profile`) y la bitácora del servidor (`mongod`), los cuales no sustituyen a un sistema de auditoría de seguridad.

---

## 2. Perfilador (Database Profiler)
* **Configuración:** Se gestiona por base de datos mediante `db.setProfilingLevel()` (nivel `0` apagado, `1` operaciones lentas o filtradas, `2` todas las operaciones) o de forma global y persistente mediante `operationProfiling.mode` en el archivo de configuración. Para reducir el ruido y registrar solo modificaciones de datos, se aplicó el filtro `{ op: { $in: ["insert", "update", "remove"] } }` en nivel `1`.
* **Interpretación:** Cada documento en la colección limitada (`capped`) `comercial.system.profile` permite identificar el tipo de operación (`op`), la colección (`ns`), el usuario autenticado (`user`), la aplicación (`appName`), la dirección de origen (`client`), la marca de tiempo (`ts`) y los parámetros de la instrucción (`command`). Sin embargo, **no registra el estado anterior de los documentos** antes de una actualización o eliminación.
* **Costo operativo:** El perfilador fue diseñado para diagnóstico de rendimiento, no para auditoría. Activar el nivel `2` en producción introduce sobrecarga de escritura y degradación de desempeño al registrar cada consulta (`find`), además de consumir espacio fijo (1 MB por defecto) que provoca la sobrescritura automática de los registros más antiguos.

---

## 3. Bitácora de `mongod`
* **Ubicación:** Se verifica mediante `db.serverCmdLineOpts().parsed` (`systemLog.path` cuando se configura salida a archivo) y se consulta en memoria desde `mongosh` mediante `db.adminCommand({ getLog: "global" })`, que devuelve los últimos 1024 eventos en formato JSON estructurado (`t`, `s`, `c`, `id`, `ctx`, `msg`, `attr`).
* **Accesos:** Bajo el componente `ACCESS`, la bitácora distingue entre autenticaciones exitosas (`"Successfully authenticated"`, `result: 0`) e intentos fallidos (`"Failed to authenticate"`), detallando en `attr.error` si el fallo ocurrió por contraseña incorrecta (`AuthenticationFailed: SCRAM authentication failed`) o por usuario inexistente (`UserNotFound`), así como el cliente y el `appName`.
* **Cambios de configuración:** Cuando se modifica el perfilador, la bitácora registra el evento `"Profiler settings changed"` con los valores anterior (`from`) y nuevo (`to`). Aunque esa línea no contiene el nombre de usuario, permite atribuir el cambio correlacionando el identificador de conexión (`ctx`, por ejemplo `conn31`) con el evento `"Successfully authenticated"` de esa misma conexión dentro del mismo ciclo de ejecución del servidor.

---

## 4. Privilegios: evidencia disponible, control compensatorio y su límite
* **Evidencia disponible:** En MongoDB Community, ni el perfilador en nivel `2` ni la bitácora de `mongod` registran la ejecución de comandos administrativos de gestión de roles (`grantRolesToUser` o `revokeRolesFromUser`).
* **Control compensatorio aplicado:** Se implementó una colección de instantáneas periódicas (`auditoria.instantaneas_privilegios`) donde el DBA almacena el resultado de `db.getUsers()` junto con la fecha (`momento`) y el usuario que tomó la captura (`tomada_por`).
* **Límite del control:** Al comparar dos instantáneas consecutivas se logró detectar que al usuario `app_ventas` se le asignó temporalmente el rol `dbAdmin` sobre `comercial` y luego se le revocó. No obstante, este control **no equivale a auditoría**: demuestra *qué* cambió y en *qué ventana de tiempo*, pero **no registra quién ejecutó el cambio de privilegios ni el instante exacto en que ocurrió**.

---

## 5. Conservación y exportación de evidencias
* **Riesgos observados:**
  1. **Volatilidad tras reinicio:** Al reiniciar el servicio `MongoDB`, el nivel del perfilador configurado en tiempo de ejecución regresa a `was: 0` y el búfer en memoria de `getLog` se vacía por completo.
  2. **Sobrescritura circular:** `system.profile` es una colección *capped* de 1 MB por defecto; al llenarse, sobrescribe los documentos más antiguos sin intervención humana.
  3. **Búfer limitado:** `getLog` solo conserva en RAM los últimos 1024 eventos.
* **Exportación realizada:** Antes de perder los registros, se exportó la colección `system.profile` a `evidencias/system-profile.json` y la salida de la bitácora a `evidencias/bitacora-mongod.log`, generando sus huellas digitales SHA-256 en `evidencias/hashes.sha256` para garantizar su integridad y cadena de custodia.

---

## 6. Comparación de trazabilidad (Resumen Actividad 9)
* **PostgreSQL (bitácora + disparadores):** Cubre de forma nativa y observada accesos exitosos/fallidos, cambios de privilegios (`GRANT`/`REVOKE`), DDL, modificaciones DML y captura del valor anterior del dato mediante registros `OLD` en triggers.
* **MongoDB Community (perfilador + bitácora):** Cubre accesos exitosos/fallidos y modificaciones de datos (solo si el perfilador está activo y sin valor anterior), cubre parcialmente cambios de estructura y de configuración (requiriendo correlación por `ctx` y con riesgo de sobrescritura), y **no cubre** cambios de privilegios de forma nativa.
* **MongoDB Enterprise / Atlas (auditoría nativa):** Según la documentación oficial (`REF-U3-04`), añade registro nativo y persistente para eventos de autenticación/autorización (incluyendo cambios de roles), cambios de esquema (DDL) y operaciones CRUD (`auditAuthorizationSuccess: true`), aunque tampoco captura el estado anterior de los documentos modificados.

---

## 7. Recomendación
* **Hechos verificados en este laboratorio:**
  * MongoDB Community Edition carece de auditoría nativa (`modules: []`).
  * Ni `system.profile` ni la bitácora de `mongod` registran comandos de asignación o revocación de privilegios (`grantRolesToUser` / `revokeRolesFromUser`).
  * El control compensatorio mediante instantáneas periódicas detecta diferencias de roles entre dos puntos en el tiempo, pero no identifica al autor ni la hora exacta del cambio.
* **Supuestos de negocio y operación:**
  * La organización requiere cumplir con controles formales de auditoría y atribución individual sobre cambios de privilegios en producción (como se plantea en el escenario de Dirección y auditoría interna).
  * La organización cuenta con presupuesto para licenciamiento comercial o migración a servicios administrados en la nube, y los cambios de permisos pueden centralizarse mediante flujos controlados.
* **Recomendación técnica:** Si auditar cambios de privilegios con atribución de autor y marca de tiempo exacta es un requisito obligatorio de cumplimiento, se recomienda **migrar a MongoDB Enterprise Edition o MongoDB Atlas (clústeres M10 o superiores)** para habilitar la **auditoría nativa** dirigida a un repositorio externo inmutable (SIEM/syslog o archivo protegido). De forma complementaria, independientemente de la edición, se debe restringir el rol `userAdmin` / `root` a una única cuenta administrativa controlada y canalizar cualquier cambio de roles a través de scripts aprobados y versionados.

---

# Preguntas de Reflexión

1. **¿Por qué el perfilador no debe presentarse como un mecanismo de auditoría de seguridad?**
   Porque fue diseñado para analizar el desempeño de consultas, no para garantizar trazabilidad de seguridad: está apagado por defecto (`was: 0`), su activación en tiempo de ejecución no sobrevive a un reinicio del servicio, utiliza una colección *capped* (1 MB por defecto) que sobrescribe automáticamente los registros antiguos al llenarse, cualquier usuario con privilegios administrativos sobre la base puede desactivarlo o eliminar `system.profile` (`drop()`), y no registra eventos críticos de seguridad como cambios de privilegios.

2. **¿Por qué el perfilador sí aporta trazabilidad aunque no sea auditoría?**
   Porque, mientras está activo y antes de que la colección se sobrescriba, registra información operativa estructurada por cada operación ejecutada: qué colección se afectó (`ns`), qué instrucción se envió (`command`), qué usuario autenticado la ejecutó (`user`), desde qué aplicación (`appName`) y dirección IP (`client`) se conectó, y en qué momento exacto ocurrió (`ts`), permitiendo reconstruir incidentes de modificación o consulta de datos.

3. **¿Qué riesgo introduce el nivel 2 al registrar filtros de consultas que pueden contener datos personales?**
   En nivel `2`, el perfilador guarda el documento completo de cada consulta (`op: "query"`), incluyendo los valores literales usados en los filtros (por ejemplo, nombres, correos electrónicos, teléfonos o RFC de clientes). Esto expone información personal sensible (PII) dentro de `system.profile` (y en `system-profile.json` al exportarse), ampliando la superficie de exposición de datos ante cualquier usuario o analista que tenga permisos de lectura sobre los registros de monitoreo.

4. **¿Qué diferencia hay entre detectar un cambio y atribuirlo a alguien?**
   **Detectar un cambio** significa comprobar que el estado de los datos o de los permisos es distinto respecto a un estado previo (por ejemplo, ver que el RFC cambió o que `app_ventas` obtuvo `dbAdmin` entre dos instantáneas). **Atribuirlo a alguien** implica contar con evidencia íntegra que vincule de forma inequívoca esa acción específica con una identidad autenticada (`user`), un origen (`client`/`appName`) y una marca de tiempo exacta (`ts`). Sin atribución, se conoce el impacto del incidente, pero no es posible establecer responsabilidades individuales.

5. **¿Por qué reconocer una limitación técnica es parte del trabajo profesional del DBA?**
   Porque la seguridad y el gobierno de datos dependen de conocer con exactitud qué controles existen y dónde termina su cobertura. Afirmar que una herramienta cubre un requisito cuando técnicamente no lo hace genera una falsa sensación de seguridad, invalida la evidencia en una auditoría formal e impide diseñar controles compensatorios adecuados o justificar ante Dirección la inversión en la arquitectura correcta.