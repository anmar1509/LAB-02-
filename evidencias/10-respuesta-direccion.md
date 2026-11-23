# Actividad 10 — Respuesta a Dirección

## 1. ¿Quién cambió el RFC de María García en la Actividad 2? ¿Y en la Actividad 3?

* **En la Actividad 2:** **No hay evidencia con esta edición.**
  * **Motivo técnico:** Al momento del incidente el perfilador de base de datos estaba apagado (`was: 0`), por lo que no se registró ninguna operación en `system.profile`. Aunque la bitácora de `mongod` registró una conexión autenticada del usuario `app_ventas` desde `app-ventas-web` a esa hora, la bitácora del servidor no registra las sentencias CRUD ejecutadas, por lo que no es posible vincular técnicamente la modificación del RFC con ese usuario.
* **En la Actividad 3:** El usuario **`app_ventas@comercial`**.
  * **Evidencia:** Con el perfilador activo en nivel 2 (`db.setProfilingLevel(2)`), la colección `comercial.system.profile` registró el documento de la operación `op: "update"` sobre `ns: "comercial.clientes"` con filtro `{ _id: 3 }` y actualización `{ $set: { rfc: "GAMA780912XYZ" } }`, atribuido explícitamente en el campo `user` a `app_ventas@comercial`.

---

## 2. ¿Cuándo y desde qué aplicación?

* **En la Actividad 2:** **No hay evidencia con esta edición** respecto al momento exacto de ejecución ni a la aplicación que efectuó el cambio de datos, debido a que el perfilador estaba desactivado (`was: 0`). Solo existe evidencia circunstancial en la bitácora de `mongod` de un inicio de sesión exitoso con `appName: "app-ventas-web"`, sin rastro de la consulta de actualización.
* **En la Actividad 3:**
  * **Cuándo:** En la marca de tiempo registrada en el campo **`ts`** del documento en `system.profile` (sustituir por la fecha y hora exacta `ISODate(...)` de tu salida en la Actividad 3; en la guía de referencia: `2026-10-25T22:52:45.909Z`).
  * **Desde qué aplicación:** Desde **`app-ventas-web`**, según consta en el campo `appName` del registro en `system.profile` (y desde el cliente `127.0.0.1`).

---

## 3. ¿Qué valor tenía antes?

* **No hay evidencia con esta edición.**
* **Motivo técnico:** MongoDB Community Edition no cuenta con auditoría nativa ni con disparadores (*triggers*) a nivel de base de datos que capturen el estado previo de los documentos antes de una modificación. El perfilador (`system.profile`) únicamente almacena los parámetros de la instrucción recibida en el campo `command` (`command.q` para el filtro y `command.u` para el nuevo valor asignado en `$set`), pero no guarda una copia del documento anterior ni el valor previo del campo `rfc`.

---

## 4. ¿Cambió alguien los privilegios de `app_ventas`?

* **Evidencia directa del evento y su autor:** **No hay evidencia con esta edición.**
  * **Motivo técnico:** La auditoría nativa de eventos de autorización y gestión de roles (como `grantRolesToUser` y `revokeRolesFromUser`) es una función exclusiva de MongoDB Enterprise y MongoDB Atlas. En MongoDB Community, ni el perfilador en nivel 2 ni la bitácora del servidor (`mongod`) registran la ejecución de comandos administrativos de asignación o revocación de roles.
* **Evidencia mediante control compensatorio (`auditoria.instantaneas_privilegios`):** Al comparar las instantáneas periódicas tomadas manualmente con `db.getUsers()`, se detectó que entre dos capturas se agregó temporalmente el rol `dbAdmin` sobre la base `comercial` al usuario `app_ventas` y posteriormente fue revocado, restableciendo su rol original `readWrite`. Sin embargo, este control compensatorio solo demuestra que el cambio ocurrió dentro de una ventana de tiempo entre dos instantáneas, pero **no permite determinar quién ejecutó el cambio ni el momento exacto en que sucedió**.