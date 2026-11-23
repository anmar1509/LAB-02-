# Actividad 3 — Activar el perfilador y registrar operaciones

## 9.1 Interpretar los documentos

| # | op | Usuario | Cliente | Aplicación | Momento | Qué hizo | Documentos afectados |
|---|---|---|---|---|---|---|---|
| 1 | `query` | `app_ventas@comercial` | `127.0.0.1` | `app-ventas-web` | `2026-11-23T03:37:22.495Z` | Consultó `nombre` y `ciudad` de los clientes donde `ciudad: 'Culiacán'` (`find`) | 5 examinados, 3 devueltos (`nreturned: 3`) |
| 2 | `update` | `app_ventas@comercial` | `127.0.0.1` | `app-ventas-web` | `2026-11-23T03:37:27.828Z` | Actualizó el campo `rfc` a `'GAMA780912XYZ'` del cliente con `_id: 3` (`updateOne`) | 1 examinado, 1 modificado (`nModified: 1`) |
| 3 | `remove` | `app_ventas@comercial` | `127.0.0.1` | `app-ventas-web` | `2026-11-23T03:37:32.002Z` | Eliminó el documento del cliente con `_id: 2` (`deleteOne`) | 1 examinado, 1 eliminado (`ndeleted: 1`) |

## Preguntas

1. **¿Puedes saber cuál era el RFC antes del cambio con la información de `system.profile`?**
   No hay evidencia con esta edición del valor anterior del RFC. La colección `system.profile` únicamente guarda el comando de actualización enviado (`{ '$set': { rfc: 'GAMA780912XYZ' } }`) y el contador de documentos modificados (`nModified: 1`), pero no captura el estado previo del documento antes de sobrescribirlo.

2. **¿Puedes saber qué datos tenía el documento eliminado?**
   No hay evidencia con esta edición del contenido que tenía el documento antes de ser borrado. El registro de `op: 'remove'` en `system.profile` solo conserva el filtro utilizado para eliminar (`q: { _id: 2 }, limit: 1`) y confirma que se eliminó 1 registro (`ndeleted: 1`), sin respaldar sus campos previos.

3. **A diferencia de `log_statement = 'mod'` en PostgreSQL, el nivel 2 sí registró la consulta. ¿Qué ventaja y qué riesgo tiene eso?**
   * **Ventaja:** Permite obtener trazabilidad completa de lectura sobre datos sensibles (saber qué usuario o aplicación consultó información confidencial como RFC o teléfonos, qué filtros usó y cuándo ocurrió), además de facilitar el análisis de rendimiento de consultas lentas.
   * **Riesgo:** Genera una fuerte degradación del rendimiento en el servidor al tener que escribir en disco cada operación `find` y provoca que la colección limitada `system.profile` (1 MB por defecto) se llene y sobrescriba la evidencia crítica en cuestión de segundos; además, expone valores sensibles incluidos en los filtros de búsqueda en texto claro dentro del perfilador.

4. **¿Por qué el campo `user` incluye `@comercial`?**
   Porque en MongoDB la identidad completa de un usuario está compuesta por su nombre y la base de datos de autenticación donde fue definido (`usuario@base_de_datos`). Como el usuario `app_ventas` se creó dentro de la base de datos `comercial`, el servidor lo identifica inequívocamente como `app_ventas@comercial`.