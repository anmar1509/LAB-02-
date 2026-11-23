# LAB-02 — Trazabilidad en MongoDB Community

**Materia:** Gestión de Seguridad y Desempeño de Bases de Datos  
**Unidad:** III — Auditoría, trazabilidad y gobierno práctico de datos  
**Temas:** 3.1 Auditoría de bases de datos · 3.2 Bitácoras y trazabilidad  
**Nombre:** Ana Maribel Felipe Rodriguez 

---

## 1. Despliegue y Versión del Servidor

* **Motor y edición:** MongoDB 7 Community Edition (verificado mediante `db.adminCommand({ buildInfo: 1 }).modules` con resultado `[]`).
* **Versión exacta:** `7.0.x` (sustituir por el resultado exacto de `db.version()` en tu equipo, p. ej. `7.0.43`).
* **Plataforma y método de despliegue (Anexo A):** Instalación nativa en **Windows** mediante instalador MSI, ejecutándose como servicio del sistema (`MongoDB`).
* **Configuración de seguridad:** Control de acceso habilitado en el archivo `mongod.cfg` (`security.authorization: enabled`) y administración del servicio desde la aplicación *Servicios* / PowerShell (`Restart-Service MongoDB`).
* **Herramientas cliente:** `mongosh` y MongoDB Compass conectando localmente a `127.0.0.1:27017` sobre la base de datos `comercial`.

---

## 2. Objetivo del Laboratorio

Configurar e interpretar los mecanismos de trazabilidad nativos disponibles en **MongoDB Community Edition** —el perfilador de base de datos (`system.profile`) y la bitácora estructurada del servidor (`mongod`)—, delimitar con precisión técnica qué eventos auditables cubren y cuáles no (como los cambios de privilegios o el estado previo de los documentos), aplicar un control compensatorio mediante instantáneas periódicas y comparar su alcance frente a PostgreSQL y MongoDB Enterprise/Atlas.

