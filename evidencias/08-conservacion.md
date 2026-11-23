# Actividad 8 — Conservación y Exportación de Evidencias

## 1. ¿Qué ocurrió con el nivel del perfilador y con los registros de `system.profile` tras reiniciar el servicio de MongoDB?

* **Nivel del perfilador (`db.getProfilingStatus()`):** Regresó a `was: 0` (apagado). Esto ocurre porque la configuración aplicada en tiempo de ejecución mediante `db.setProfilingLevel(2)` vive únicamente en la memoria del proceso `mongod` y no se persiste automáticamente en el archivo de configuración (`mongod.conf` / `mongod.cfg`). Si no se reactiva tras un reinicio, todas las operaciones posteriores dejan de registrarse.
* **Registros almacenados (`system.profile`):** Los 3 documentos registrados antes del reinicio permanecieron intactos (`countDocuments() == 3`), ya que `system.profile` es una colección almacenada en disco dentro de la base de datos `comercial`.

---

## 2. ¿Cómo puede perderse la evidencia del perfilador y de la bitácora sin que nadie la borre intencionalmente?

* **Sobrescritura circular en `system.profile` (colección *capped*):** Al ser una colección de tamaño fijo (por defecto 1 MB), cuando recibe un volumen alto de consultas y alcanza su límite de capacidad, MongoDB sobrescribe automáticamente los documentos más antiguos con los más nuevos (comportamiento FIFO).
* **Desbordamiento del búfer en memoria (`getLog: "global"`):** El comando administrativo mantiene en RAM únicamente los últimos **1024 eventos** del servidor. En cuanto ocurren nuevos eventos de conexión o sistema, los registros anteriores son expulsados de la memoria.
* **Reinicio del servicio `mongod`:** Al detener o reiniciar el servidor, el búfer en memoria de `getLog` se vacía por completo (perdiendo los eventos previos al reinicio si no se configuró un archivo de salida en disco mediante `systemLog.path`), y el perfilador vuelve a nivel `0`, dejando sin auditar las operaciones futuras.

---

## 3. ¿Para qué sirve calcular y conservar el archivo `hashes.sha256` junto con las exportaciones?

* **Garantía de integridad forense:** El algoritmo SHA-256 genera una huella digital única de 64 caracteres hexadecimales para `system-profile.json` y `bitacora-mongod.log`. Si alguien altera aunque sea un solo carácter, espacio o marca de tiempo en cualquiera de los archivos exportados, el hash cambiará por completo al volver a calcularse.
* **Cadena de custodia:** Permite demostrar en una auditoría o investigación de incidentes que las evidencias extraídas de la base de datos son auténticas y no han sido manipuladas desde el momento exacto de su recolección.