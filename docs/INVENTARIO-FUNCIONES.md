# Inventario de funciones — base del registro unificado (2026-09-29)

Objetivo (dueño, 29/09): un repo único **GP2-Registro-Produccion** para registrar la producción de
**Cervantes y Virgilio**, con funciones **GP2 nuevas que exigen sesión** (login de operario por legajo +
red de la empresa, o mail Google). Se arranca por **Cervantes** (menos funciones). Cuando esté todo
migrado, esta lista sirve para **eliminar las originales**. No se toca lo que usan hoy los operarios
hasta que la versión nueva esté probada.

Maestros por planta (no se mueven): Cervantes → `loekemeyer/Registro-Produccion-2.0` · Virgilio →
`loekemeyer/Gestion-Virgilio`.

## 1. Registro Producción 2.0 (Cervantes) — EN USO, schema `public`, clave pública sola

Archivos: `app.js` (3.8k líneas), `index.html`, `sw.js` (cola offline).

| # | Qué | Tipo | Operación | Control hoy | Notas |
|---|---|---|---|---|---|
| 1 | `db_n8n_espejo` | tabla | leer, alta (upsert), editar, borrar | ninguno (anon) | trigger `gp2_matriz_racha_trg_espejo`. **La leen**: reporte diario PDF 18:00, premios, Planify, disruptivas, tiempos |
| 2 | `Registros Produccion Cervantes` | tabla | alta (upsert), editar, borrar | ninguno (anon) | también la escribe la cola offline `sw.js` |
| 3 | `Auditoria_Produccion` | tabla | alta | ninguno | |
| 4 | `registrar_unidades(p_n_matriz, p_cantidad, p_completar, p_legajo, p_evento_id)` | rpc | escribe stock por cajón (`UnixCajon_…`) | ninguno, anon ejecuta | |
| 5 | `asignar_matriz_balancin(p_balancin, p_matriz)` | rpc | escribe `Balancines` (botón CM) | ninguno, anon ejecuta | |
| 6 | `send-whatsapp` | Edge | alertas de matriz (plantillas) | plantillas fijas desde v55 | |
| 7 | `Empleados` | tabla | leer (por legajo; por email en el login de `index.html`) | lectura anon | roles `es_*`/`ve_*` para `capsDe()`/`botonVisible()` |
| 8 | `Matrices` | tabla | leer | lectura anon | |
| 9 | `Balancines` | tabla | leer | lectura anon | |
| 10 | `UnixCajon_Stock_Registro_Prod_Cerv` | tabla | leer | lectura anon | |

## 2. GP2 — app de operarios (`Gestion-Productiva-2.0/Produccion/RegistroApp`) — HOY SIN USO, schema `GP2`

Archivos: `operarios_gp2.js`, `Registro_GP2.html`, `Operarios_GP2.html`, `sw.js` (1.8k líneas).

| # | Qué | Tipo | Operación | Control hoy |
|---|---|---|---|---|
| 1 | `GP2.registrar_evento_prod(p jsonb)` | rpc | escribe eventos de producción | **ya exige sesión** (`_exigir_autorizado`, 28/09) |
| 2 | `GP2.anular_evento_prod(p_id_ejecucion)` | rpc | anula un evento | ya exige sesión |
| 3 | `GP2.tomar_rollo(p_legajo, p_comp_id, p_kg_por_rollo, p_matriz, p_fecha)` | rpc | consumo de fleje | ya exige sesión |
| 4 | `GP2.cerrar_rollo(p_legajo, p_quedo_resto, p_uni_producidas, p_fecha)` | rpc | cierre de rollo | ya exige sesión |
| 5 | `GP2.registrar_produccion(p_legajo, p_matriz, p_uni, p_fecha, p_nombre, p_comp_salida_id, p_golpes)` | rpc | alta de producción (`Registro_GP2.html`) | ya exige sesión |
| 6 | `GP2.registro_operarios_bundle()` | rpc | lectura (operarios, matrices) | lectura anon |
| 7 | `GP2.movimientos_bundle()` | rpc | lectura | lectura anon |
| 8 | `GP2.produccion` | tabla | lectura | |

⚠ `_exigir_autorizado` hoy acepta solo mails de la whitelist o service_role: **una sesión de operario
(`login-operario`) sería rechazada**. Hay que sumarle `jwt_rol() = 'operario'` (y validar `p_legajo = jwt_legajo()`).

## 3. Puntos a resolver antes de escribir código

1. **Qué se toma de cada una**: el dueño dijo "funciones de una y la visual de la otra". Recomendación:
   **visual de Registro Producción 2.0** (la conocen los operarios; tiene los roles por botón) y
   **funciones al estilo GP2** (schema `GP2`, con sesión y rol).
2. **`db_n8n_espejo` tiene muchos lectores** (reporte diario, premios, Planify, racha, disruptivas). Si la
   app nueva escribe solo en `GP2`, todo eso se queda sin datos. Opciones: (a) las funciones GP2 escriben
   también el espejo mientras dura la migración; (b) se migran los lectores. Recomendado (a).
3. **Regla 0 de GP2** (nada de `public`): matrices, operarios y balancines de Cervantes hoy viven en
   `public`. Hay que mapearlos a `GP2` (`GP2.matriz` ya existe) o dejar el puente del punto 2 explícito.
4. **Cola offline** (`sw.js`): tiene que mandar la sesión y no perder cargas si vence.

## 4. Diferencias entre los dos sistemas (29/09)

| Tema | Registro Producción 2.0 (Cervantes, en uso) | GP2 app de operarios (sin uso) |
|---|---|---|
| Dónde guarda | `public`: `db_n8n_espejo` + `Registros Produccion Cervantes` | `GP2`: eventos/producción por RPC (`registrar_evento_prod`) |
| Quién calcula | la **tablet** arma tiempos/premio y los escribe en la tabla | la **base** calcula (funciones GP2) |
| Botones | 20: E, C, PB, BC, MOV, LIMP, Perm, AL, PR, PC, **RD**, MOV P, **MM**, **CM**, PM, RM, **REM**, **PCM**, **TRM**, **TL** | 14: faltan RD, MM, CM, REM, PCM, TRM, TL; tiene **CT** (solo legajo 19) |
| Qué ve cada operario | por rol, desde flags de `Empleados` (`capsDe`/`botonVisible`) | todos lo mismo; `if legajo === "19"` hardcodeado |
| Cajón | anota **unidades**; controla stock por cajón (`registrar_unidades` → `UnixCajon`) | anota **golpes** del contador × `GP2.matriz.uni_x_golpe` |
| Fleje / rollos | no | `tomar_rollo` / `cerrar_rollo` (consumo de fleje en el inventario GP2) |
| Cambiar matriz (CM) | asigna la matriz al balancín (`asignar_matriz_balancin`) | no existe |
| Alertas WhatsApp | sí (`send-whatsapp`, plantillas) | no |
| Login | legajo (compartido con GV, `vir_legajo_auth`) o Google para 3 mails fijos en el código | legajo, sin sesión |
| Seguridad hoy | ninguna: todo con la clave pública | las 5 que escriben ya exigen sesión (pero no aceptan rol operario) |
| Cola offline | sí (`sw.js`) | sí (`sw.js`) |
| Quién lee lo que escribe | reporte diario 18:00, premios, Planify, racha, disruptivas, tiempos | stock/costos GP2 |

## 5. Envío sin conexión de Registro Producción 2.0 (cómo reintenta hoy)

Tres colas separadas, todas con la clave pública:

| Cola | Dónde vive | Qué manda | Quién reintenta |
|---|---|---|---|
| **Principal** (`enqueue` → `flushQueue`) | `localStorage` + copia en IndexedDB (`idbPut`) | mensaje a `Registros Produccion Cervantes` (`postToSupabase`) y después calcula el espejo en la tablet (`procesarParaEspejo` → `db_n8n_espejo`) | la página cada 3 s (`setInterval`), al volver la red (`online`), y el **service worker** en segundo plano (`sync` "flush-queue" → `processQueueInBackground`, lee IndexedDB) |
| Stock de cajón (`flushStockQueue`) | solo `localStorage` | `rpc registrar_unidades` | la página cada 3 s |
| Balancín (`flushBalancinQueue`) | solo `localStorage` | `rpc asignar_matriz_balancin` | la página |

Detalles: lotes de 20; cada falla suma `__tries` y se anota en `Auditoria_Produccion` (1er intento y cada 5);
timeout 15 s; `reconcileQueueWithIDB` saca de la cola de la página lo que el service worker ya mandó; el
badge muestra ✓ / ⏳ N / ⚠ N.

⚠ **[Probable] pérdida silenciosa hoy**: cuando el service worker manda un mensaje en segundo plano (app
cerrada), sube solo el mensaje; `procesarParaEspejo` corre únicamente en la página. Al reabrir,
`reconcileQueueWithIDB` lo da por enviado y **nunca calcula su fila en `db_n8n_espejo`** (tiempos/premio).
En el diseño nuevo no pasa: la base calcula al recibir el mensaje, venga de la página o del service worker.

Para la versión nueva: una sola cola (mensaje con `client_id`), el service worker manda la **sesión**
guardada en IndexedDB en vez de la clave pública; si da 401 el ítem queda en cola hasta renovar la sesión.
Stock y balancín dejan de ser colas aparte: los resuelve la base a partir del mensaje (C, CM).

## 6. Envío sin conexión: Registro Producción 2.0 vs GP2 (29/09)

| Tema | Registro Producción 2.0 | GP2 (`operarios_gp2.js`) |
|---|---|---|
| Dónde guarda la cola | `localStorage` **+ IndexedDB** | solo `localStorage` |
| Service worker (manda con la app cerrada) | **sí**, `sync` "flush-queue" | **no**: hay un `sw.js` en la carpeta pero la app **no lo registra** (copia muerta; además apunta a una tabla de `public`) |
| Cada cuánto reintenta | cada **3 s** + al volver la red (`online`) | cada **60 s** + al tocar el badge / Terminar Día; no escucha `online` |
| Qué manda | el mensaje crudo a la tabla, y **la tablet** calcula el espejo | el evento ya armado a `registrar_evento_prod` y **la base** calcula |
| Lote / timeout | 20 por vuelta, 15 s de timeout | toda la cola, sin timeout |
| Registro de fallas | a `Auditoria_Produccion` (1er intento y cada 5) | solo en la pantalla (`markFailed`) |
| Colas aparte | stock de cajón y balancín (solo `localStorage`, sin SW) | rollos (`tomar_rollo`/`cerrar_rollo`), FIFO, corta al primer fallo de red; si el servidor rechaza con código, **lo descarta** |
| Duplicados | `on_conflict=id` / `ID_Ejecucion` | `id_ejecucion` único en `GP2.produccion` |
| Hueco conocido | lo que manda el SW no calcula el espejo (sección 5) | con la app cerrada no se manda nada; y desde el 28/09 `registrar_evento_prod` exige sesión → hoy toda carga anónima fallaría y quedaría en cola |

**Para la app nueva se toma lo mejor de cada una**: cola en `localStorage` + IndexedDB y service worker de
Registro Producción (reintento cada pocos segundos, `online`, timeout, auditoría de fallas), con el modelo
de GP2 (mandar el mensaje y que **la base** calcule, idempotente por `client_id`), una sola cola (sin colas
aparte de stock/balancín/rollos: la base las deriva del mensaje) y **la sesión** en cada envío, también
desde el service worker.
