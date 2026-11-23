# Actividad 7 — Cambios de privilegios: el límite más importante

## 1. Análisis del control compensatorio (Instantáneas)

| Pregunta | ¿La instantánea responde? |
| :--- | :--- |
| ¿Cambiaron los privilegios de `app_ventas`? | Sí |
| ¿Qué rol se agregó? | Sí (`dbAdmin` en la base `comercial`) |
| ¿Quién lo agregó? | No |
| ¿Cuándo exactamente? | No, solo que ocurrió entre dos instantáneas |

---

## 2. Respuestas a las preguntas de la actividad

### 1. ¿Por qué un cambio de privilegios es uno de los eventos auditables más importantes?
Porque representa una **escalada de privilegios** o una modificación directa en la superficie de ataque del sistema. Si un usuario malicioso o una cuenta comprometida obtiene roles administrativos (como `dbAdmin` o `root`), puede acceder a datos confidenciales, alterar esquemas, eliminar colecciones o incluso modificar y desactivar los propios mecanismos de registro (`system.profile`) para encubrir sus acciones.

### 2. ¿Con qué frecuencia tomarías instantáneas? ¿Qué factores influyen?
En un entorno productivo crítico se recomienda tomarlas **cada hora o de forma diaria** (automatizadas mediante un *cron job* o tarea programada), además de ejecutarlas **bajo demanda** inmediatamente antes y después de cualquier ventana de mantenimiento o cambio autorizado.

**Factores que influyen:**
* La criticidad y sensibilidad de la información almacenada.
* La frecuencia con la que cambian los usuarios y roles en la organización.
* Los requisitos normativos o de cumplimiento aplicable.
* La ventana máxima de exposición que el negocio puede tolerar sin detectar una alteración no autorizada.

### 3. ¿Quién debería poder escribir en `auditoria.instantaneas_privilegios`?
Únicamente una **cuenta de servicio automatizada** dedicada exclusivamente a ejecutar la tarea de respaldo de instantáneas, configurada bajo el principio de mínimo privilegio (solo con permisos de inserción `insert` en `auditoria.instantaneas_privilegios` y lectura de usuarios `viewUser` en las bases auditadas). Ningún usuario de aplicación (`app_ventas`), desarrollador ni cuenta operativa cotidiana debe tener permisos de escritura, modificación (`update`) o borrado (`delete`) sobre esta colección para preservar la integridad de la evidencia.

### 4. Redacta en tres líneas cómo comunicarías esta limitación a Dirección.
> MongoDB Community Edition carece de auditoría nativa, por lo que no deja rastro nativo ni inmediato sobre quién modifica permisos de usuarios o el momento exacto en que ocurre. Como control compensatorio, hemos implementado instantáneas periódicas automatizadas que permiten detectar qué privilegios cambiaron entre dos puntos en el tiempo. Sin embargo, este mecanismo no identifica al autor del cambio ni el instante preciso en que sucedió; obtener trazabilidad completa requeriría migrar a MongoDB Enterprise o Atlas.
