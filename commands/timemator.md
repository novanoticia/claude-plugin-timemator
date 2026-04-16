---
description: Controla Timemator — timers, consultas de tiempo y reportes
allowed-tools: Bash, mcp__Control_your_Mac__osascript
argument-hint: "[start|stop|status|today|week|report|add]"
---

# Comando /timemator

Cuando el usuario invoca `/timemator`, lee los argumentos que pasó y sigue el flujo correspondiente.
Si no hay argumentos, muestra el menú de opciones.

## Argumentos reconocidos

- `start [proyecto]` — inicia timer, opcionalmente para un proyecto específico
- `stop` — para el timer activo
- `pause` — pausa el timer activo
- `status` — muestra qué timer está corriendo ahora mismo
- `today` — resumen del tiempo trabajado hoy
- `week` — resumen de la semana actual
- `report [días]` — reporte detallado (por defecto 7 días)
- `add <duración> <proyecto> [fecha]` — crea una entrada manual

## Si no hay argumentos

Presenta estas opciones al usuario:

```
¿Qué quieres hacer con Timemator?

1. ▶  Iniciar timer
2. ⏹  Parar timer
3. 📊 Tiempo de hoy
4. 📅 Resumen semanal
5. ✏️  Añadir tiempo manual
6. 📋 Reporte detallado
```

## Flujo: start

1. Antes de iniciar, comprueba si el timer ya está corriendo (Start Timer disabled = ya corre).
2. Si ya corre, informa al usuario en lugar de intentar iniciar.
3. Si no corre, ejecuta:

```applescript
tell application "Timemator" to activate
delay 0.5
tell application "System Events"
    tell process "Timemator"
        click menu item "Start Timer" of menu "Tracking" of menu bar 1
    end tell
end tell
```

4. Confirma leyendo el estado del menú: si "Start Timer" quedó disabled → "✅ Timer iniciado"
5. Si devuelve error de accesibilidad, indica al usuario que añada Claude a Accesibilidad en
   Ajustes del Sistema → Privacidad y Seguridad → Accesibilidad.

## Flujo: stop (pause)

Timemator no tiene "Stop" como tal — usa Pause para detener el timer activo:

```applescript
tell application "Timemator" to activate
delay 0.5
tell application "System Events"
    tell process "Timemator"
        click menu item "Pause Timer" of menu "Tracking" of menu bar 1
    end tell
end tell
```

Confirma leyendo el menú: si "Start Timer" quedó enabled → "⏸ Timer pausado"

## Flujo: status

Lee el archivo JSON del Group Container (sin permisos):

```python
import json, os
from datetime import datetime

BASE = os.path.expanduser(
    "~/Library/Group Containers/UZ3GNZNN2A.com.catforce.timemator.group/widgetData/"
)

with open(os.path.join(BASE, "currentSession")) as f:
    sesion = json.load(f)

corriendo = sesion.get("timerIsRunning", False)
tarea = sesion.get("currentTaskName", "—")
proyecto = sesion.get("parentName", "—")
inicio_ts = sesion.get("currentSessionBeginTimestamp")

if corriendo and inicio_ts:
    inicio = datetime.fromtimestamp(inicio_ts)
    elapsed = datetime.now() - inicio
    mins = int(elapsed.total_seconds() // 60)
    h, m = divmod(mins, 60)
    duracion = f"{h}h {m}m" if h > 0 else f"{m}m"
    print(f"▶ Corriendo: {tarea} ({proyecto}) — {duracion}")
else:
    with open(os.path.join(BASE, "timer")) as f:
        t = json.load(f)
    ultima = t.get("taskName", "—")
    ultimo_proy = t.get("folderName", "—")
    print(f"⏸ Sin timer activo. Último: {ultima} ({ultimo_proy})")
```

Si hay timer activo, muestra: "▶ Timer corriendo: [proyecto] — iniciado hace Xh Ym"
Si no hay, muestra: "⏸ No hay timer activo. Último: [tarea] ([proyecto])"

## Flujo: today

Lee `weeklyTimeTrackedChart` del Group Container y filtra por hoy.
Script completo en `references/data-reading.md` (Script 2: Resumen de hoy).
Presenta resultado como tabla:

```
📊 Tiempo trabajado hoy — jueves 26 marzo

Proyecto A     2h 15m  ████████████░░
Proyecto B     1h 00m  ██████░░░░░░░░
Sin proyecto   0h 30m  ███░░░░░░░░░░░
─────────────────────
Total          3h 45m
```

## Flujo: week

Lee `weeklyTimeTrackedChart` (Script 3 de `references/data-reading.md`).
Muestra desglose por día con el total de la semana. Para el mes, usa `monthlyTimeTrackedChart` (Script 4).

## Flujo: report [días]

1. Si no se especificaron días, pregunta el rango o usa 7 por defecto.
2. Genera reporte completo incluyendo:
   - Totales por proyecto
   - Promedio diario
   - Día más productivo
   - Comparación con semana anterior si hay datos
3. Ofrece exportar como Markdown o CSV.

## Flujo: nueva carpeta <nombre>

**Importante:** Timemator usa edición inline para el nombre. Necesita la ventana abierta.

```applescript
tell application "Timemator" to activate
delay 0.5
tell application "System Events"
    tell process "Timemator"
        -- Abrir ventana primero (sin ella no hay campo de texto para el nombre)
        click menu item "Show Tracking" of menu "Tracking" of menu bar 1
        delay 0.8
        click menu item "New Folder..." of menu "File" of menu bar 1
        delay 1.0
        -- Limpiar nombre por defecto y escribir el nuevo
        keystroke "a" using {command down}
        delay 0.2
        keystroke "NOMBRE_CARPETA"
        delay 0.3
        key code 36 -- Return para confirmar
    end tell
end tell
```

## Flujo: add <duración> <proyecto> [fecha]

Interpreta el tiempo en formato natural: "2h", "1h30m", "90 minutos", "media hora".
Interpreta la fecha: "ayer", "el lunes", "hoy a las 15:00", etc.

Abre Timemator y usa el menú File → New Task... o Cmd+N:

```applescript
tell application "Timemator" to activate
delay 0.5
tell application "System Events"
    tell process "Timemator"
        click menu item "Show Tracking" of menu "Tracking" of menu bar 1
        delay 0.6
        keystroke "n" using {command down}
        delay 0.5
        -- Rellenar nombre de tarea, luego navegar con Tab a los campos de tiempo
    end tell
end tell
```

Confirma con los detalles de la entrada creada.

## Manejo de errores

- **Timemator no está instalado**: "No encuentro Timemator instalado. ¿Está en /Applications?"
- **Sin permiso de accesibilidad**: Mostrar instrucciones del setup-guide.md
- **Error leyendo datos**: Informar y ofrecer alternativa por exportación CSV
- **Proyecto no encontrado**: Listar proyectos disponibles y preguntar cuál usar
