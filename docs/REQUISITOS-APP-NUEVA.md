# Requisitos de la app unificada de registro de producción

Lista viva de lo que pidió el dueño para la app nueva (GP2-Registro-Produccion). Cada ítem con fecha y cita.

## R1. Auto-actualización cada 10 minutos, sin perder lo que estaba haciendo el operario (29/09)

Dueño: *"para cada 10 m (solo en horario de trabajo), verificar si la app está actualizada, y si no
actualizarla, no sin antes guardar lo que estaba haciendo (si se quedó poniendo valores en un input u
oprimiendo un botón para mandarlo, que al recargar le continúe desde ahí)."*

Nota: reemplaza la decisión del 28/09 para Gestión Virgilio ("no recargar solo porque pueden estar en medio
de una carga"): acá la recarga es automática **porque** se guarda y se restaura el estado.

Cómo:
1. **Cada 10 min, solo en horario de trabajo** (**lunes a sábado, 07:00–18:00 AR**, dueño 29/09: amplio "por si llegan antes y se van después"): pedir
   `version.json` sin caché y comparar con la versión cargada. Fuera de horario, no chequea.
2. **Si hay versión nueva**, antes de recargar:
   - la **cola de envíos** ya está en IndexedDB/localStorage (no se pierde);
   - se guarda un **borrador de pantalla** en `localStorage` (`app_borrador`): legajo logueado, botón/opción
     elegida, valores de TODOS los inputs visibles (número de matriz, golpes/unidades, texto), modal abierto,
     paso del flujo, posición, y la hora en que se guardó;
   - si justo había un **envío en curso** (tocó "Enviar"), se espera a que termine o se deja encolado —
     nunca se recarga en el medio de un envío.
3. **Recarga** forzando el HTML nuevo (`location.replace(path + "?_=" + Date.now())`).
4. **Al abrir**, si hay borrador de menos de X minutos (ej. 30) y del mismo legajo: reabre la misma pantalla,
   el mismo botón/modal y **rellena los inputs**; el operario solo confirma. Después se borra el borrador.
   Si el borrador es viejo o de otro legajo, se descarta.
5. La sesión (login de operario) sobrevive a la recarga (guardada), así que no pide el legajo de nuevo.
6. Guardas: no recargar si la app está en segundo plano con un modal a medio completar sin borrador
   guardado; si falla el guardado del borrador, no recarga y reintenta en el próximo ciclo.
7. Se prueba con Playwright: llenar un input, forzar versión nueva, verificar que tras recargar el input y el
   modal vuelven igual.
8. **Anti-duplicado (dueño 29/09: "no tendría que haber bug de combinar la 2 y la 3: que se envió pero al
   regresar la página aparece para volver a enviar").** Tres candados, porque cualquiera solo puede fallar:
   - **Un id por carga**: al abrir el formulario se genera un `client_id` y viaja DENTRO del borrador. El
     mensaje que se envía lleva ese mismo `client_id`.
   - **El borrador muere al encolar, no al llegar al servidor**: al tocar Enviar, primero se escribe en la
     cola (IndexedDB) y, apenas esa escritura confirma, se borra el borrador y se anota el `client_id` en
     `app_enviados` (últimos 200). Desde ese momento el envío es responsabilidad de la cola, no de la pantalla.
   - **Al restaurar se verifica**: si el `client_id` del borrador ya está en la cola o en `app_enviados`, el
     borrador se descarta y no se muestra. (Cubre el corte justo entre encolar y borrar el borrador.)
   - **Y la base no duplica aunque llegue dos veces**: `GP2.recibir_mensaje_cervantes` es idempotente por
     `client_id` (ya probado): el segundo envío devuelve "ya estaba" sin crear otra fila.
   - Test: tocar Enviar, forzar la recarga en el medio (antes y después de encolar), y verificar que tras
     volver hay exactamente 1 fila en la base y ningún formulario pidiendo reenviar.

9. **En Cervantes el aparato es el CELULAR PROPIO del operario, no una tablet** (dueño 29/09). Cambia dos cosas:
   - **El chequeo no puede depender solo del reloj de 10 min**: con el celular en el bolsillo el navegador
     congela la página y el temporizador no corre. Se chequea además **al volver la app al frente**
     (`visibilitychange`), y si hay versión nueva se recarga **en ese momento**, antes de que toque nada.
   - **Borrador válido hasta el fin del día** (no 30 min): el celular es de una sola persona, así que no hay
     riesgo de que el turno siguiente herede la carga de otro. Se restaura si es del mismo legajo y del mismo día.

## R2. Los operarios salen de `planify.employees` (RRHH), no de `public."Empleados"` (29/09)

- El legajo es **texto con la letra de la empresa** (`c94` ≠ `94`; hay colisiones reales: 29/c29, 122/C122).
  Comparar en minúscula. Nunca identificar por el número solo.
- Permisos de botones y horario de producción: tabla GP2 atada a `planify.employees.id`.
- `login-operario` (hoy valida contra `public."Empleados"`) pasa a validar contra `planify.employees` activo.
- Pantalla de legajo: la letra es parte del legajo → teclado propio en pantalla (0-9 + C), botones grandes.
- Filtro: `planify.employees` activo + `planify.empleados_liquidacion.tipo_empleado='planta'` activo (por `employee_id`)
  + casilla `GP2.operario.registra_produccion` (planta sola no alcanza). Todo resuelto en `GP2.operario_por_legajo`.
  Esa tabla es de sueldos: se lee solo vía función SECURITY DEFINER que devuelve legajo, nombre y si es planta.
- Legajo 600 = pruebas (caso especial).
- Detalle: `CONOCIMIENTO_GP2.md` §4gq (repo Gestion-Productiva-2.0).

## Pendiente de confirmar con el dueño
- (nada pendiente de R1)

Dato del dueño 29/09: **no hay Wi-Fi de invitados; cada sede tiene un solo Wi-Fi.** Entra quien tenga
la clave del Wi-Fi + un legajo activo.
