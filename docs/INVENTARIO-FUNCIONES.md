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
