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

## 7. Otros sistemas internos de las apps (comparación, 29/09)

Relevado leyendo el código completo: RP = `registro-produccion-2.0` (`app.js` v1.9.0, `index.html`, `sw.js`,
`styles.css`, `manifest.json`); GP2 = `Gestion-Productiva-2.0/Produccion/RegistroApp` (`operarios_gp2.js`,
`Operarios_GP2.html`, `Registro_GP2.html`, `sw.js`, `manifest.json`) + las funciones en `db/funciones_GP2.sql`.
Datos confirmados con SELECT (29/09). Lo de envío sin conexión está en 5 y 6 y no se repite.

### 7.1 Resumen

| Sistema | Registro Producción (RP) | GP2 | Riesgo | Para la app nueva |
|---|---|---|---|---|
| Identificación / sesión | legajo tipeado; o gate `vir_legajo_auth` / Google que precarga y **oculta** el legajo todo el día; 3 mails "supervisor" fijos | legajo tipeado; la página exige login de oficina (`auth-guard`) pero el JS manda **anon** | ALTO: RP sin control; GP2 hoy no puede escribir (401) | sesión de operario (`login-operario`) en la base; botón "Cambiar operario / Salir" |
| Estado por legajo y día | `localStorage` por día+legajo, 14 días, en el aparato | igual, más simple | MEDIO: se pierde al cambiar de tablet o a medianoche | estado "abierto" (matriz, TM, rollo) lo devuelve la base; tablet = caché |
| Botones y roles | `capsDe`/`botonVisible` desde flags de `Empleados` | lista fija + `LEGAJO_EDUARDO = "19"` | MEDIO | flags en la tabla de empleados GP2; la base valida que el rol pueda mandar ese código |
| Tiempos muertos | abre/cierra con el mismo botón; demás botones **grises** | abre/cierra; demás botones visibles, rechaza con alert | BAJO | regla en la base (un TM abierto por legajo) + UI de RP |
| PM / RM / PCM | PM = TM + WhatsApp; RM = flujo cajón→rotura→CM; PCM = TM con popup "¿se rompió?" | PM y RM puntuales, sin flujo; sin PCM | MEDIO (PM mide distinto) | decidir PM; flujo RM de RP |
| Cambiar matriz / balancín | CM = TM + `asignar_matriz_balancin` | sin CM | MEDIO | CM como evento; la base asigna balancín |
| Variantes / piezas | lista fija de 8 matrices + variantes por letra | `matriz_salidas` (pieza → stock) | MEDIO | pieza de GP2 (sale de la ruta, no de una lista en el código) |
| Cajón | unidades; stock por cajón `UnixCajon` (faltan X, "cajón completo") | golpes × `uni_x_golpe`; `fabricar_stock` | ALTO: editar/borrar no revierte stock en ninguna | la base deriva stock del evento y lo revierte al anular |
| Rollos / fleje | no | `tomar_rollo`/`cerrar_rollo`, picker de kilaje | MEDIO | si entra, por evento (E con rollo, PR/CT) y sin legajo fijo |
| Continuar cajón entre días | sí (solo matrices sin control de stock, solo mismo aparato, código 151515 para saltearlo) | no | MEDIO | si sigue: la base sabe qué quedó abierto ayer |
| Llegada tarde | LT automático al 1er evento si > **08:30 fijo** | igual | MEDIO: 11 activos entran 08:00 | la base con `hora_entrada` del empleado |
| Historial / deshacer / editar | ver, **editar** y **borrar** todo el día; borra fila en tablas (hard delete) | ver y borrar (`anular_evento_prod`, baja lógica) | ALTO | anulación = evento nuevo; ventana y permisos en la base |
| Terminar Día | cierra TM abierto solo, pregunta último cajón / seguís mañana, FJ con foto del día; **sin red no termina** | bloquea si hay TM o matriz sin C; FJ a la cola | MEDIO | FJ a la cola; la base cierra lo abierto |
| Cálculo tiempos/premio | tablet (resta TM dentro del cajón, recalcula el día) | base, **sin restar TM** | ALTO (GP2) | base, restando TM |
| WhatsApp | Matriz sin tiempo (E), Paro (PM abrir), Rompió (RM) | no | BAJO | trigger en la base, no desde la tablet |
| Reloj / zona | hora del aparato; día AR por `Intl`; estado cambia a medianoche | igual; limpieza con fecha del aparato | MEDIO | la base sella `recibido_en` y controla desvío |
| Versión / actualización | `LOCAL_VERSION` vs `CACHE_VERSION` de `sw.js` cada 60 s, recarga; badge con versión | compara `?v=` del HTML; badge solo cola | BAJO | mecanismo de RP, sin recargar en medio de una carga |
| PWA | manifest + SW registrado (sin caché de archivos) | manifest copiado de RP y **no enlazado**; SW no registrado | BAJO | manifest y SW propios |
| Errores / log | `alert`, `Auditoria_Produccion`, `swLog` a localStorage sin pantalla | `alert` + error en el historial | BAJO | mensajes en pantalla + log de fallas en la base |
| Accesibilidad | campo principal **sin `inputmode`**; textos 9,5–14 px | `inputmode="numeric"`; algunos 11–14 px | MEDIO | regla GP2: ≥18 px en campos, teclado numérico |
| Catálogos al abrir | 4 lecturas a `public`; sin red no entra nadie | bundle; reintenta cada 15 s; sin red no entra nadie | MEDIO | bundle GP2 guardado en el aparato |
| Kiosko / TV | no existe | no existe | — | — |

### 7.2 Detalle por sistema

> **Dato del dueño (29/09): en Cervantes cada operario carga desde SU celular, no hay tablet compartida.**
> Donde abajo dice "tablet", en Cervantes léase "celular del operario". Consecuencias: el legajo oculto todo el
> día no es un problema de aparato compartido; sí lo es que el celular esté en el Wi-Fi correcto (regla de red)
> y que el navegador congele la página en el bolsillo (temporizadores, cola en segundo plano).

**1. Identificación y sesión.**
RP: `goToOptions` (app.js:1595) solo exige que el legajo esté en `empleadosMap`, **no mira `Activo`** (hay 70
empleados, 35 activos). El gate de `index.html:29-84` precarga el legajo si hay `vir_legajo_auth` del día o sesión
Google con `Empleados.email`, y **oculta el campo** (index.html:43-45): en una tablet compartida no hay forma de
cambiar de operario ni de cerrar sesión (`backToLegajo`, app.js:1614, solo vuelve a la pantalla). Los 3
`SUPERVISOR_EMAILS` (index.html:32) tipean cualquier legajo. Usada suelta (fuera de `/cervantes/`), cualquiera
tipea cualquier legajo. `cargarCatalogos` lee `Empleados` con `select("*")` y clave pública (app.js:79).
GP2: `goToOptions` (operarios_gp2.js:1058) tampoco mira `activo` (lo trae el bundle). `Operarios_GP2.html:5` carga
`auth-guard.js` (login de oficina prendido desde 28/09), pero `operarios_gp2.js:12-15` arma su propio cliente con
`persistSession:false` → todo sale como **anon** y `registrar_evento_prod` lo rechaza. Además rompe la regla de
`GP2_SB()`.
Nueva: sesión de operario; la base toma el legajo del JWT (`jwt_legajo()`), no del cuerpo; empleado inactivo no
entra; botón visible "Cambiar operario".

**2. Estado por legajo y día.**
RP: clave `prod_state_Cervantes_v2_supa::<día AR>::<legajo>` (app.js:297, 315). Guarda `lastMatrix`, `lastCajon`,
`lastDowntime`, `last2` (historial, hasta 700), `matrixNeedsC`, `lateArrival*`, `cajonContinuado`,
`continuacionConsultada`, `terminoConContinuacion`, `tdCargaPrevia*`, `pendingRM` (app.js:319-349). Se borra al
abrir si cambió el día y la clave tiene más de 14 días (app.js:383-393). Otras claves: `prod_queue…`,
`prod_stockq…`, `prod_balq…`, `prod_day_guard…`, `sw_log_v1`, `__lastUpdateReload` (sessionStorage),
`wa_plantilla_activa` (solo se lee).
GP2: `gp2_op_state::<día>::<legajo>` con `lastMatrix` (+pieza), `lastCajon`, `lastDowntime`, `last2`, `rollo`;
colas `gp2_op_queue`, `gp2_op_rqueue`. `cleanupOldStates` (operarios_gp2.js:1081) calcula los 14 días con la
**fecha del aparato**, no la AR.
Riesgo común: la "matriz abierta" y el "TM abierto" viven solo en ese aparato; otro equipo o un borrado de
datos los pierde. Nueva: la base guarda lo abierto por legajo y el bundle lo devuelve al entrar.

**3. Botones y roles.**
RP: `OPTIONS` (app.js:1023-1048, 20 códigos), `NORMAL_BASE`, `capsDe`, `botonVisible` (app.js:1060-1093); los
ocultos no se dibujan y la grilla se reacomoda (app.js:1577-1586). Datos: 1 alimentador, 4 piedra, 2 matricería,
5 `ve_cm`.
GP2: `OPTIONS` fija (operarios_gp2.js:69-89, 14 códigos) + `CT_OPTION` solo si `legajo === "19"`
(operarios_gp2.js:10, 419, 631). Eduardo además cierra rollo en PR (líneas 726-733, 893-901).
Nueva: `capsDe`/`botonVisible` leyendo flags del empleado GP2; CT/rollo = flag, no legajo; la base rechaza un
código que el rol no tiene.

**4. Flujo de cada botón.**
RP (`NON_DOWNTIME` app.js:1053): **puntuales** E, C, RM, RD, LT; **tiempo muerto** (1er toque abre, 2do cierra
midiendo) todo el resto: PB, BC, MOV, MOV P, MM, LIMP, Perm, AL, PR, PC, PM, PCM, CM, TRM, TL, REM.
Validaciones en `sendFast` (app.js:2028): E solo dígitos, matriz existente (o con variante), bloqueado si
`matrixNeedsC` (hay que mandar C antes), alerta si `Tiempo_Historico` = 0 (140 matrices así); C/RM/PM/RD/PCM
exigen matriz activa; C entero (501 acepta decimal con coma); CM/TRM matriz existente. **No valida que la matriz
esté activa** (RP no tiene ese dato).
GP2 (`NON_DOWNTIME` operarios_gp2.js:93): puntuales E, C, CT, **RM, PM**, RD, LT. Valida matriz existente **y
activa** (líneas 810-828), pieza si la matriz tiene varias salidas, confirmación "golpes × factor" (línea 848).
Dos toques seguidos: el mismo botón de TM cierra; otro botón con TM abierto → RP lo tiene gris (app.js:1539),
GP2 lo deja tocar y rechaza al enviar (operarios_gp2.js:838); en GP2, con TM abierto la flecha "volver" no
funciona (operarios_gp2.js:775) y el operario queda trabado en la selección equivocada hasta ir a la pantalla
de legajo.

**5. Tiempos muertos abiertos y su cierre.**
Ambas: `lastDowntime` local; el cierre manda `hs_inicio` = hora de apertura. RP los cierra solos en Terminar
Día (app.js:3182-3196); GP2 no deja terminar el día con uno abierto (operarios_gp2.js:985). Un TM abierto
**desaparece a medianoche** en las dos (el estado es por día). RP resta del cajón solo los TM que caen
**enteros** dentro de él (app.js:683, 731): un TM que empezó antes del cajón no se descuenta.
Nueva: la base lleva el TM abierto por legajo, lo cierra al recibir el cierre o el FJ, y resta la parte que se
solapa con cada cajón.

**6. Llegada tarde.** `maybeSendLateArrival` RP app.js:2005 / GP2 operarios_gp2.js:337: al primer evento del día
en ese aparato, si son más de las **08:30** (fijo) manda LT desde 08:30. `Empleados.hora_entrada` existe (08:00
para 11 activos, 07:00 y 09:00 para uno cada uno) pero solo se usa en la continuación de cajón
(app.js:1931). Se dispara antes de validar el botón (app.js:2033). En GP2 el LT viaja sin segundos
(`toRpcPayload` solo manda segundos a C o TM, operarios_gp2.js:294-297). Últimos 30 días: 61 LT.
Nueva: LT lo calcula la base con la `hora_entrada` del empleado y el primer evento del día (de cualquier aparato).

**7. Cambiar matriz y balancín (CM).**
RP: CM abre modal matriz + balancín activo (app.js:2365-2436); `asignarMatrizBalancin` actualiza local y llama
la RPC **antes** de que `sendFast` valide el evento (app.js:2452 vs 2178-2194); CM es TM (2º toque lo cierra).
CM **no cambia la matriz activa**: después hay que tocar E. 29 CM en 30 días.
GP2: CM sacado del menú (operarios_gp2.js:85-86), aunque el código lo sigue contemplando.
Nueva: CM como evento que la base traduce en asignación de balancín; decidir si CM deja la matriz nueva como
activa.

**8. PM / RM / PCM (paradas y rotura).**
RP: PM = TM, WhatsApp "Paro Matriz" al abrir (app.js:2245-2261). RM = `ejecutarFlujoRM` (app.js:2505): pide
unidades (modal sin cancelar), cierra un C con `completar=true` (deja el stock de cajón en 0), manda RM puntual +
WhatsApp "Rompió Matriz", y termina en CM (o popup "No cambiar / Cambiar" si la matriz es Tipo A); se retoma
tras F5 con `pendingRM` (app.js:2606). PCM = TM; al cerrar pregunta "¿se rompió?" y si sí dispara RM
(app.js:2617). GP2: PM y RM puntuales, sin cantidad ni CM; no hay PCM.

**9. Cajón: unidades, golpes y stock.**
RP: C en **unidades**; si la matriz está en `UnixCajon_Stock_Registro_Prod_Cerv` (321 filas; no están 501 ni las
Tipo E) muestra "faltan X para completar" al tipear E y al abrir C (app.js:1484-1521), checkbox "cajón completo",
y llama `registrar_unidades` idempotente por id (app.js:202). Matriz Tipo A + rol CM → popup "Seguir / Cambiar
matriz" (app.js:2286).
GP2: C en **golpes** × `matriz.uni_x_golpe` (`registro_en_golpes`), con aviso en vivo y confirmación si el factor
es > 1; la base mueve stock con `fabricar_stock` según la pieza.
Riesgo: ni borrar ni editar un C revierte el stock (RP: `deleteHistItem` app.js:1157 y edición app.js:1248 no
tocan `UnixCajon`; no hay trigger en esas tablas. GP2: `anular_evento_prod` solo pone `eliminar='S'`,
funciones_GP2.sql:650-665, y `GP2.produccion` no tiene trigger).

**10. Rollos / fleje.** Solo GP2: en E cualquier operario elige el rollo por kilaje (`actualizarRolloPicker`,
operarios_gp2.js:739) → `tomar_rollo`; el estado estima kg usados = uni / `ppk` (líneas 191-196); Eduardo cierra
con CT o PR (+"quedó resto"). `cerrarRollo` se llama en **los dos toques** de PR (abrir y cerrar el TM,
operarios_gp2.js:897-900). Borrar el E no devuelve el rollo.

**11. Continuar cajón entre días (solo RP).** Si ayer, en Terminar Día, eligió "sigo mañana" y la matriz no tiene
control de stock, hoy aparece "⚡ Continuar Matriz" (app.js:1731-1818); el C suma los segundos de ayer hasta
`hora_salida` (32 de 70 empleados la tienen). Solo funciona en el **mismo aparato**. Tocar otra cosa pide el
código de Logística **`151515` escrito en el código** (app.js:1831), visible en el repo público.

**12. Historial, deshacer y edición.**
RP: historial del día y de 14 días (solo del aparato). **Editar** (app.js:1248-1450): cambia código y valor de
cualquier registro del día, recalcula tiempos/premio en la tablet y hace update/insert/delete en `db_n8n_espejo`;
no ajusta `lastDowntime` ni stock. **Borrar** (app.js:1157-1215): borra la fila de `Registros Produccion
Cervantes` y de `db_n8n_espejo` con la clave pública y deja rastro en `Auditoria_Produccion`. Sin ventana de
tiempo ni permiso. Último mes aprox.: 53 ELIMINAR, 7 EDITAR.
GP2: solo borrar; si ya se envió llama `anular_evento_prod` (baja lógica), si está en cola lo saca; sin red no
deja. `editModal` está en el HTML (Operarios_GP2.html:254-277) pero no tiene código.
Nueva: el operario solo "deshace" (evento de anulación que referencia el id), dentro de una ventana a definir; la
base revierte stock y recalcula; editar lo hace un supervisor.

**13. Terminar Día.**
RP (`confirmarTerminarDia` app.js:3166): reenvía en bloque todo el historial del día (idempotente), cierra el TM
abierto, y según la matriz pregunta "¿hiciste un último cajón?" (con stock) o "¿seguís mañana?" (sin stock); el
FJ lleva la foto del día en JSON y se manda **directo, no por la cola**: sin red avisa error y no termina
(app.js:3239-3250). GP2 (`openTerminarDia` operarios_gp2.js:980): bloquea con TM o matriz sin C; el FJ va a la
cola como un evento más (`registrar_evento_prod` con matriz "FJ").
Nueva: FJ a la cola; la base cierra lo abierto con la hora del FJ; la foto del día sirve para auditar pérdidas.

**14. Cálculo de tiempos y premio.**
RP: la tablet (`procesarParaEspejo` app.js:762) calcula segundos, TM a restar, `Tiempo_Toma`, `Premio`
(= (1 − neto/uni/histórico) × 10) y recalcula todos los cajones del día cuando llega un TM (app.js:692).
GP2: la tablet manda `segundos_trabajados` = fin − inicio **bruto** (operarios_gp2.js:291-296) y la base calcula
el premio con eso (funciones_GP2.sql:6296-6303) **sin restar los tiempos muertos** → premio más bajo del real.
Además `hora_inicio`/`hora_fin` viajan sin fecha.
Nueva: la base calcula todo con la fórmula de RP, restando el solapamiento con TM.

**15. Alertas WhatsApp (solo RP).** `enviarAlertaWA` → Edge `send-whatsapp` (app.js:29-66): "Matriz sin Tiempo" en
cada E de matriz sin histórico (5 en 30 días), "Paro Matriz" al abrir PM (30), "Rompió Matriz" en RM (2). Plantilla
por `localStorage.wa_plantilla_activa`, que **ninguna app escribe** (siempre sale la "reducida"). Se llama desde
la tablet con la clave pública; sin red la alerta se pierde (no hay cola).
Nueva: la base dispara la alerta al recibir el evento (una vez, aunque se reenvíe).

**16. Reloj, fecha y zona horaria.** Las dos sellan el evento con el **reloj del aparato** (`isoNow`) y calculan el
día AR con `Intl` (bien). La clave de estado se recalcula en cada lectura: a las 00:00 el estado del día anterior
deja de verse (matriz y TM abiertos se "pierden"). RP recién limpia claves viejas al recargar. `Registro_GP2.html`
arma la fecha como `YYYY-MM-DDT12:00:00` sin zona.
Nueva: la base guarda la hora del aparato y la de recepción, marca desvíos grandes, y define qué pasa con lo
abierto a medianoche (según turnos).

**17. Versión y auto-actualización.** RP: `LOCAL_VERSION` (app.js:291) contra `CACHE_VERSION` de `sw.js` pedido
sin caché cada 60 s y al volver a la pestaña; si difiere recarga (antiloop 60 s en sessionStorage,
app.js:3406-3433); además recarga en `controllerchange` y `SW_UPDATED`. La recarga puede caer en medio de una
carga (los modales RM/TD se retoman, lo tipeado se pierde). El badge muestra la versión. GP2: compara el `?v=` del
script dentro del HTML (Operarios_GP2.html:19-37); el badge muestra solo la cola.
Nueva: mecanismo de RP pero esperando a que no haya selección ni modal abierto.

**18. PWA.** RP: `manifest.json` enlazado, SW registrado que no cachea archivos (solo background sync). GP2: el
`manifest.json` es copia del de RP ("Cervantes") y **no está enlazado**; `sw.js` no se registra (sección 6).

**19. Errores y mensajes.** RP: `alert()` y textos en `#error`; fallas de envío a `Auditoria_Produccion`; `swLog`
guarda 100 líneas en `sw_log_v1` pero **no existe `#swLog`** en `index.html` (no se ve). GP2: `alert()` y el error en
el historial del día.

**20. Accesibilidad.** RP: el campo principal (`textInput`, index.html:131) **no tiene `inputmode`** → sale el
teclado completo para E/C/CM/TRM; `box-desc` 11 px (9,5 px en celular), cuerpo de Terminar Día 14 px, edición
16 px. Los modales nuevos sí (28 px, numérico). GP2: `inputmode="numeric"` en los campos; descripciones 11 px,
`#matrizInfo` 14 px.
Nueva: regla de GP2 (≥18 px en campos, `inputmode` siempre, etiquetas visibles, 44 px de toque).

**21. Reglas de negocio escritas en el código.**
RP: matriz **501** (piedra) acepta decimal con coma y no tiene control de cajón (app.js:162, 2053); Tipo **E** sin
control (por no estar en `UnixCajon`); Tipo **A** = alimentador (app.js:176); **8 matrices con variante** a mano
(12, 10, 28, 39, 79, 80, 81, 127; app.js:2079-2139) + variantes por letra desde la base; **08:30** de entrada
(app.js:2011); código **151515** (app.js:1831); 3 mails supervisores (index.html:32); "08:30:00" por defecto de
`hora_entrada` (app.js:1775).
GP2: legajo **19** (operarios_gp2.js:10); **08:30** (línea 342); golpes/unidades por parámetro de la base.
Nueva: todo esto como datos (tipo de matriz, pieza, horario del empleado, permisos), no en el JS.

**22. Catálogos al abrir.** RP: 4 lecturas en paralelo a `public` sin guardar copia (app.js:77-107); si no hay red
al abrir, "Legajo no encontrado" para todos. GP2: `registro_operarios_bundle` sin copia; reintenta cada 15 s
(operarios_gp2.js:29-41).
Nueva: guardar el último bundle en el aparato y entrar con él sin red.

**23. Kiosko / TV.** No existe en ninguna de las dos.

### 7.3 Decisiones pendientes del dueño (29/09)

Resuelta: ~~aparato compartido~~ → en Cervantes cada operario usa su celular (29/09).

| # | Tema | Decisión del dueño (29/09) |
|---|---|---|
| 1 | Llegada tarde | horario de Planify de cada operario (`empleados_liquidacion.horario_laboral`, vía `GP2.operario_por_legajo`) |
| 2 | PM | como RP: tiempo muerto con duración + aviso WhatsApp al abrir |
| 3 | Cajón | `GP2.matriz.carga_en`: golpes si sale más de 1 por golpe (18 matrices), unidades si sale 1 (388), kg la piedra 501 (coma o punto = decimal; se guarda numérico) |
| 4 | CM | solo roles específicos + matricería + alimentador; asigna matriz↔balancín; NO deja la matriz activa a quien la cambia |
| 5 | RM | igual que hoy + aviso WhatsApp |
| 6 | Deshacer / editar | en el admin (maestro) |
| 7 | Terminar Día con TM abierto | se cierra solo (RP ya lo hace, `confirmarTerminarDia` paso 2) |
| 8 | Seguir cajón al otro día | se mantiene; el código de Logística = secreto en la base |
| 9 | Rollos | los maneja el alimentador (rol), no el legajo 19 |
| 10 | Turnos después de medianoche | no hay; al terminar el día se cierran los TM, no el cajón "sigo mañana" |
| 11 | WhatsApp | los que ya existen (matriz sin tiempo, paro, rotura) |
| 12 | Botones | los de RP con sus flags (RD, REM, MM, TRM, TL, PCM incluidos) |
| + | Quién entra | planta NO alcanza (Pregelj 203 y Cornejo c91 son planta y no operarios): **PENDIENTE** permiso "registra producción" |
| + | Piedra "pendiente de pesar" | válida, pero **solo si el admin la habilita en el panel admin** (apagada por defecto; la base la rechaza si está apagada). El peso se carga después en el admin sobre el cajón original |
