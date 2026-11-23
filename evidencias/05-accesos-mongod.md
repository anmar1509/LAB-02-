# Actividad 5 — Accesos en la bitácora de `mongod`

## Preguntas

1. **¿Qué campo distingue un acceso fallido de uno exitoso?**
   Se distinguen principalmente por tres campos del documento JSON registrado en la bitácora:
   * **`msg`:** Muestra `"Failed to authenticate"` cuando el acceso falla, frente a `"Successfully authenticated"` cuando es exitoso.
   * **`id`:** El identificador numérico del evento es `5286307` para autenticación fallida y `5286306` para autenticación exitosa.
   * **`attr.result`:** Devuelve `0` en accesos exitosos, mientras que en los fallidos devuelve un código de error distinto de cero (`18` para contraseña incorrecta y `11` para usuario no encontrado), acompañado del campo `attr.error`.

2. **¿Qué diferencia hay entre el error de contraseña incorrecta y el de usuario inexistente? ¿Por qué el cliente recibe el mismo mensaje en ambos casos?**
   * **Diferencia interna en la bitácora:** En el servidor (`mongod.log`), el intento con contraseña incorrecta registra `error: 'AuthenticationFailed: SCRAM authentication failed, storedKey mismatch'` (`result: 18`), mientras que el de usuario inexistente registra `error: 'UserNotFound: Could not find user "usuario_inexistente" for db "comercial"'` (`result: 11`).
   * **Por qué el cliente recibe el mismo mensaje (`Authentication failed.`):** Es un control estándar de seguridad para prevenir la **enumeración de usuarios**. Si el motor revelara al cliente si una cuenta existe o no, un atacante podría identificar qué nombres de usuario son válidos antes de dirigir un ataque de fuerza bruta.

3. **¿Por qué el nombre de la aplicación del intento fallido es `mongosh 2.10.0` y no `app-ventas-web`?**
   Porque en la cadena de conexión utilizada para los intentos fallidos se omitió el parámetro `?appName=app-ventas-web`. Cuando una conexión no especifica `appName`, el controlador (*driver*) envía por defecto el nombre y la versión del cliente desde el cual se realizó la conexión (por ejemplo, `mongosh` o MongoDB Compass). *(Nota: En MongoDB 8.0, los metadatos de la aplicación se registran en el evento `NETWORK` de la misma conexión `ctx`, por lo que `e.attr?.doc?.application?.name` aparece como `undefined` al consultar únicamente el evento `ACCESS`).*

   