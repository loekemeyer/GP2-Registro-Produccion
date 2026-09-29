# Requisitos de la app unificada de registro de producción

Lista viva de lo que pidió el dueño para la app nueva (GP2-Registro-Produccion). Cada ítem con fecha y cita.

## R1. Auto-actualización cada 10 minutos, sin perder lo que estaba haciendo el operario (29/09)

Dueño: *"para cada 10 m (solo en horario de trabajo), verificar si la app está actualizada, y si no
actualizarla, no sin antes guardar lo que estaba haciendo (si se quedó poniendo valores en un input u
oprimiendo un botón para mandarlo, que al recargar le continúe desde ahí)."*

Nota: reemplaza la decisión del 28/09 para Gestión Virgilio ("no recargar solo porque pueden estar en medio
de una carga"): acá la recarga es automática **porque** se guarda y se restaura el estado.

Cómo:
1. **Cada 10 min, solo en horario de trabajo** (lun–vie; horario a confirmar, ej. 06:00–18:00 AR): pedir
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

## Pendiente de confirmar con el dueño
- Horario de trabajo exacto para el chequeo (¿06:00–18:00? ¿sábados?).
- Minutos máximos para restaurar un borrador (propuesto: 30).
