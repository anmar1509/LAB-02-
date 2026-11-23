# Actividad 2 — El incidente sin perfilado

## Preguntas

1. **¿Puedes demostrar qué cambió? ¿Y quién se conectó a esa hora?**
   * **Qué cambió:** No hay evidencia con esta edición de qué documento o campo cambió ni de quién ejecutó la modificación. Como el perfilador de base de datos estaba apagado (`was: 0`), `db.system.profile.countDocuments()` devolvió `0`, y la bitácora del servidor (`mongod.log` / `getLog`) no registra operaciones CRUD por defecto, devolviendo `0` coincidencias para `"GAMA780912XXX"`.
   * **Quién se conectó a esa hora:** Sí es posible demostrar quién se conectó y desde qué aplicación. La bitácora de `mongod` registró en el componente `NETWORK` (`client metadata`) conexiones identificadas con `appName: "app-ventas-web"` (`conn31`, `conn32`, `conn33`, `conn34`, `conn35`, etc., a las `2026-11-22T20:15:00`) y en el componente `ACCESS` la autenticación exitosa (`Successfully authenticated`) del usuario `app_ventas`.

2. **¿Por qué `mongosh` abre varias conexiones (`conn8`, `conn9`, ...) para una sola instrucción? ¿Cuál es la que se autentica?**
   * El controlador (*driver*) moderno de MongoDB utilizado por clientes como `mongosh` o MongoDB Compass administra un grupo (*pool*) de conexiones y separa el descubrimiento y monitoreo de la topología del servidor (SDAM — *Server Discovery and Monitoring*) de las conexiones de ejecución de operaciones. Por ello abre conexiones iniciales sin autenticar (`Connection not authenticating`) solo para verificar el estado del servidor e intercambiar metadatos (`client metadata`), y abre otras conexiones independientes para procesar consultas y comandos.
   * **¿Cuál es la que se autentica?** La conexión dedicada del *pool* de trabajo encargada de ejecutar las operaciones (por ejemplo, `conn9` en la guía), que realiza el proceso de autenticación SCRAM (`Successfully authenticated`) antes de enviar la instrucción `updateOne`.