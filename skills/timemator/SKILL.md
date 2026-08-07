---
name: timemator
description: >
  This skill should be used when the user wants to control Timemator or check tracked time.
  Triggers on "inicia el timer", "para el timer", "cuánto tiempo llevo hoy", "registra 2 horas de X",
  "resumen semanal de tiempo", "reporte Timemator", "cuánto he trabajado", "/timemator", "timer de",
  "start timer", "stop timer", "tiempo trabajado", "horas trabajadas", "pausa el timer".
compatibility: >
  macOS con Timemator instalado. Requiere que el cliente pueda ejecutar `osascript`
  en esa misma máquina (Claude Code, o Cowork con el conector "Control your Mac";
  cualquier cliente con acceso equivalente al Mac sirve) y permisos de
  accesibilidad para el GUI scripting. Sin esa capacidad el skill se carga y puede
  explicar la operación, pero no actúa sobre la app.
metadata:
  version: "0.1.0"
---

# Skill: Timemator

Este skill permite controlar Timemator desde Claude usando AppleScript y GUI scripting. Timemator no
tiene diccionario AppleScript propio, por lo que la integración usa tres estrategias:

1. **Control de timer**: vía teclas de acceso rápido globales (configuradas en Timemator)
2. **Lectura de datos**: leyendo los archivos de base de datos (binary plist) directamente
3. **Exportación**: vía GUI scripting para obtener CSV detallado

## Requisito previo: permisos de accesibilidad

GUI scripting en macOS requiere que el proceso que ejecuta osascript tenga permiso de Accesibilidad.
Si una operación falla con "not allowed assistive access", indica al usuario:

> "Para que pueda controlar Timemator, necesitas dar permiso de Accesibilidad a la app que ejecuta
> Claude. Ve a **Preferencias del Sistema → Privacidad y Seguridad → Accesibilidad** y añade la
> aplicación correspondiente (Terminal, Claude, etc.)."

## Fuente de datos principal (sin permisos)

Timemator escribe datos en tiempo real en archivos JSON que usa el widget del sistema.
**Usa siempre esta fuente para consultas y reportes** — no necesita ningún permiso extra:

```
~/Library/Group Containers/UZ3GNZNN2A.com.catforce.timemator.group/widgetData/
```

Archivos clave:
- `currentSession` → estado actual (`timerIsRunning`, tarea activa, timestamps)
- `timer` → detalles del último timer registrado
- `weeklyTimeTrackedChart` → tiempo por día de la semana (en segundos)
- `monthlyTimeTrackedChart` → tiempo por día del mes

Los scripts Python completos están en `references/data-reading.md`.

## Acciones disponibles

### 1. Iniciar / Pausar timer

Los controles del timer están en el menú **Tracking** (verificado). Requiere permiso de Accesibilidad
para Claude.app en Ajustes del Sistema → Privacidad y Seguridad → Accesibilidad.

**Iniciar timer:**
```applescript
tell application "Timemator" to activate
delay 0.5
tell application "System Events"
    tell process "Timemator"
        click menu item "Start Timer" of menu "Tracking" of menu bar 1
    end tell
end tell
```

**Pausar timer:**
```applescript
tell application "Timemator" to activate
delay 0.5
tell application "System Events"
    tell process "Timemator"
        click menu item "Pause Timer" of menu "Tracking" of menu bar 1
    end tell
end tell
```

**Comprobar si el timer está corriendo (método fiable — menú):**
```applescript
tell application "Timemator" to activate
delay 0.3
tell application "System Events"
    tell process "Timemator"
        set startEnabled to enabled of menu item "Start Timer" of menu "Tracking" of menu bar 1
        -- Si Start está deshabilitado → timer corriendo
        -- Si Start está habilitado → timer parado
        if startEnabled then
            return "parado"
        else
            return "corriendo"
        end if
    end tell
end tell
```

> **Nota:** El archivo JSON del widget (`currentSession.timerIsRunning`) se actualiza con retraso.
> Para saber el estado en tiempo real, usa siempre el método del menú de arriba.

### 2. Leer estado actual y datos de tiempo (sin permisos)

Timemator escribe archivos JSON en su Group Container para alimentar los widgets del sistema.
**Esta es la fuente de datos principal** — no requiere permisos de accesibilidad ni GUI.

Ruta: `~/Library/Group Containers/UZ3GNZNN2A.com.catforce.timemator.group/widgetData/`

```python
import json, os
from datetime import datetime, date

BASE = os.path.expanduser(
    "~/Library/Group Containers/UZ3GNZNN2A.com.catforce.timemator.group/widgetData/"
)

# Estado actual del timer
with open(os.path.join(BASE, "currentSession")) as f:
    sesion = json.load(f)

corriendo = sesion.get("timerIsRunning", False)
tarea = sesion.get("currentTaskName", "—")
proyecto = sesion.get("parentName", "—")
inicio_ts = sesion.get("currentSessionBeginTimestamp")

# Tiempo de hoy (de los datos semanales)
with open(os.path.join(BASE, "weeklyTimeTrackedChart")) as f:
    semana = json.load(f)

for bar in semana["bars"]:
    if datetime.fromtimestamp(bar["date"]).date() == date.today():
        segundos_hoy = bar.get("totalValue", 0)
        break
```

Los scripts completos para estado, resumen diario, semanal y mensual están en
`references/data-reading.md`.

### 3. Exportar CSV para reportes

Para generar reportes precisos, exporta un CSV via GUI scripting:

```applescript
tell application "Timemator" to activate
delay 0.5
tell application "System Events"
    tell process "Timemator"
        -- Abre el diálogo de exportación
        try
            click menu item "Export..." of menu "File" of menu bar 1
        on error
            click menu item "Exportar..." of menu "Archivo" of menu bar 1
        end try
    end tell
end tell
delay 1
-- Guardar en una ruta temporal conocida
tell application "System Events"
    tell process "Timemator"
        -- Navegar al diálogo de guardar y especificar ruta
        keystroke "g" using {command down, shift down}
        delay 0.5
        keystroke "/tmp/timemator_export.csv"
        keystroke return
        delay 0.5
        keystroke return
    end tell
end tell
```

Luego parsea el CSV con Python:
```python
import csv
from datetime import datetime, date, timedelta

with open('/tmp/timemator_export.csv', 'r', encoding='utf-8') as f:
    reader = csv.DictReader(f)
    entries = list(reader)

# Filtrar por fecha
today = date.today()
today_entries = [e for e in entries if e.get('Date', '').startswith(str(today))]
```

### 4. Crear entrada manual

Para añadir tiempo pasado, usa el menú de Timemator o el diálogo de nueva entrada:

```applescript
tell application "Timemator" to activate
delay 0.5
tell application "System Events"
    tell process "Timemator"
        -- Abrir diálogo de nueva entrada manual
        try
            click menu item "New Time Entry..." of menu "File" of menu bar 1
        on error
            keystroke "n" using {command down}
        end try
    end tell
end tell
```

## Flujo recomendado para cada acción

### /timemator start [proyecto]
1. Ejecuta Estrategia C o D para iniciar
2. Si hay proyecto especificado, busca en la lista de proyectos y selecciona
3. Confirma el inicio leyendo el estado actual

### /timemator stop
1. Usa Estrategia D (Stop)
2. Confirma que el timer se ha detenido

### /timemator status
1. Intenta leer el plist de la base de datos para el estado actual
2. Si no es posible, activa Timemator e inspecciona la ventana

### /timemator today / week
1. Exporta CSV a /tmp/timemator_export.csv
2. Parsea y agrupa por proyecto
3. Presenta el resumen en formato tabla

### /timemator report
1. Pregunta el rango de fechas si no se especificó
2. Exporta CSV
3. Genera resumen con totales por proyecto, promedio diario, día más productivo

### /timemator add [duración] [proyecto] [fecha]
1. Abre diálogo de nueva entrada manual
2. Rellena los campos según los parámetros
3. Confirma la creación

## Manejo de errores comunes

- **"not allowed assistive access"** → Pedir al usuario que añada permisos en Preferencias del Sistema
- **"Timemator no está en ejecución"** → `tell application "Timemator" to activate` antes de cualquier acción
- **Archivo plist no encontrado** → Buscar en `~/Library/Containers/` si Timemator usa sandboxing
- **CSV vacío o no exportado** → Verificar que hay entradas en el rango seleccionado

## Nota sobre accesibilidad

Si el usuario no tiene permisos de accesibilidad configurados, la lectura de datos vía plist
(estrategias que no requieren GUI) seguirá funcionando. Solo las acciones de control
(start/stop/add) necesitan el permiso de accesibilidad.
