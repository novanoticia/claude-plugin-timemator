# Lectura de datos de Timemator — Guía real y verificada

## Ruta de datos (Group Container)

Los datos del widget de Timemator están en archivos JSON sin extensión en:
```
~/Library/Group Containers/UZ3GNZNN2A.com.catforce.timemator.group/widgetData/
```

Archivos disponibles y su contenido:

| Archivo | Contenido |
|---------|-----------|
| `timer` | Último timer (tarea, proyecto, timestamps inicio/fin) |
| `currentSession` | Estado actual: si está corriendo, tarea y proyecto activos |
| `weeklyTimeTrackedChart` | Tiempo por día esta semana (en segundos por tarea) |
| `monthlyTimeTrackedChart` | Tiempo por día este mes |
| `weeklyChart` | Gráfico semanal (vista alternativa) |
| `monthlyChart` | Gráfico mensual con columnas |
| `weeklyRevenueChart` | Ingresos semanales |
| `monthlyRevenueChart` | Ingresos mensuales |

> No se requiere ningún permiso especial para leer estos archivos.

---

## Estructura JSON de cada archivo

### `currentSession` — estado del timer en tiempo real

```json
{
  "parentName": "Read",
  "currentTaskName": "Digital Reading",
  "timerIsRunning": false,
  "currentSessionBeginTimestamp": 1749712462,
  "currentSessionEndTimestamp": 1749712492,
  "currentTaskP3ColorRedComponent": 0.627,
  "currentTaskP3ColorGreenComponent": 0.909,
  "currentTaskP3ColorBlueComponent": 0.062,
  "currentTaskP3ColorAlphaComponent": 1
}
```

**Campos clave:**
- `timerIsRunning` → boolean, si hay timer activo ahora mismo
- `currentTaskName` → nombre de la tarea/actividad
- `parentName` → carpeta/proyecto al que pertenece
- `currentSessionBeginTimestamp` → Unix timestamp de inicio

### `timer` — detalles del último timer registrado

```json
{
  "taskName": "Digital Reading",
  "folderName": "Read",
  "taskId": "B59BDD1B-A62E-4E26-A126-0E9A1D213790",
  "timeEntryBeginTimestamp": 1774373563,
  "timeEntryEndTimestamp": 1774373589,
  "taskBackgroundColorOptimizedForLightContent": { ... },
  "taskColor": { ... }
}
```

**Campos clave:**
- `taskName` + `folderName` → tarea y proyecto
- `timeEntryBeginTimestamp` / `timeEntryEndTimestamp` → inicio y fin en Unix timestamp
- Duración = `timeEntryEndTimestamp - timeEntryBeginTimestamp` (segundos)

### `weeklyTimeTrackedChart` — tiempo semanal

```json
{
  "firstDayOfWeek": 2,
  "bars": [
    {
      "date": 1774306800,
      "totalValue": 89,
      "timeEntryCount": 2,
      "segments": [
        {
          "taskId": "x-coredata://041E7D34.../Task/p70",
          "value": 89,
          "taskColor": { ... }
        }
      ]
    }
  ]
}
```

**Campos clave:**
- `bars` → array de 7 días, cada uno con su fecha (Unix timestamp al inicio del día)
- `totalValue` → segundos totales trabajados ese día
- `segments` → desglose por tarea (cada segmento tiene `value` en segundos)
- `firstDayOfWeek: 2` → la semana empieza el lunes (ISO 8601)

**Limitación importante:** Los `segments` solo contienen `taskId` (formato CoreData), no el nombre de la tarea directamente. Para cruzar con nombres, usa `currentSession` o el archivo `timer`.

---

## Scripts Python listos para usar

### Script 1: Estado actual del timer

```python
import json, os
from datetime import datetime

BASE = os.path.expanduser(
    "~/Library/Group Containers/UZ3GNZNN2A.com.catforce.timemator.group/widgetData/"
)

def estado_timer():
    with open(os.path.join(BASE, "currentSession")) as f:
        data = json.load(f)

    corriendo = data.get("timerIsRunning", False)
    tarea = data.get("currentTaskName", "—")
    proyecto = data.get("parentName", "—")
    inicio = data.get("currentSessionBeginTimestamp")

    if corriendo and inicio:
        inicio_dt = datetime.fromtimestamp(inicio)
        elapsed = datetime.now() - inicio_dt
        mins = int(elapsed.total_seconds() // 60)
        horas = mins // 60
        mins_r = mins % 60
        duracion = f"{horas}h {mins_r}m" if horas > 0 else f"{mins_r}m"
        return f"▶ Corriendo: {tarea} ({proyecto}) — {duracion}"
    else:
        # Leer el último timer registrado
        with open(os.path.join(BASE, "timer")) as f:
            t = json.load(f)
        ultima_tarea = t.get("taskName", "—")
        ultimo_proyecto = t.get("folderName", "—")
        return f"⏸ Sin timer activo. Último: {ultima_tarea} ({ultimo_proyecto})"

print(estado_timer())
```

### Script 2: Resumen de hoy

```python
import json, os
from datetime import datetime, date

BASE = os.path.expanduser(
    "~/Library/Group Containers/UZ3GNZNN2A.com.catforce.timemator.group/widgetData/"
)

def resumen_hoy():
    with open(os.path.join(BASE, "weeklyTimeTrackedChart")) as f:
        data = json.load(f)

    hoy = date.today()
    for bar in data["bars"]:
        fecha_bar = datetime.fromtimestamp(bar["date"]).date()
        if fecha_bar == hoy:
            segundos = bar.get("totalValue", 0)
            count = bar.get("timeEntryCount", 0)
            h = segundos // 3600
            m = (segundos % 3600) // 60
            return f"📊 Hoy: {h}h {m}m ({count} entradas)"

    return "📊 Hoy: sin registros"

print(resumen_hoy())
```

### Script 3: Resumen semanal por día

```python
import json, os
from datetime import datetime, date

BASE = os.path.expanduser(
    "~/Library/Group Containers/UZ3GNZNN2A.com.catforce.timemator.group/widgetData/"
)
DIAS = ["Lun", "Mar", "Mié", "Jue", "Vie", "Sáb", "Dom"]

def resumen_semana():
    with open(os.path.join(BASE, "weeklyTimeTrackedChart")) as f:
        data = json.load(f)

    total = 0
    lineas = []
    for bar in data["bars"]:
        dt = datetime.fromtimestamp(bar["date"])
        nombre_dia = DIAS[dt.weekday()]
        seg = bar.get("totalValue", 0)
        total += seg
        h = seg // 3600
        m = (seg % 3600) // 60
        marcador = "◀ hoy" if dt.date() == date.today() else ""
        if seg > 0 or dt.date() == date.today():
            lineas.append(f"  {nombre_dia} {dt.strftime('%d/%m')}: {h}h {m:02d}m {marcador}")

    total_h = total // 3600
    total_m = (total % 3600) // 60
    lineas.append(f"\n  Total semana: {total_h}h {total_m:02d}m")
    return "📅 Esta semana:\n" + "\n".join(lineas)

print(resumen_semana())
```

### Script 4: Resumen mensual

```python
import json, os
from datetime import datetime, date

BASE = os.path.expanduser(
    "~/Library/Group Containers/UZ3GNZNN2A.com.catforce.timemator.group/widgetData/"
)

def resumen_mes():
    with open(os.path.join(BASE, "monthlyTimeTrackedChart")) as f:
        data = json.load(f)

    total = 0
    dias_trabajados = 0
    for bar in data["bars"]:
        seg = bar.get("totalValue", 0)
        if seg > 0:
            total += seg
            dias_trabajados += 1

    total_h = total // 3600
    total_m = (total % 3600) // 60
    promedio = (total / dias_trabajados / 3600) if dias_trabajados > 0 else 0

    return (f"📆 Este mes: {total_h}h {total_m:02d}m en {dias_trabajados} días "
            f"(promedio {promedio:.1f}h/día)")

print(resumen_mes())
```

---

## Limitaciones de este enfoque

1. **Los datos del widget se actualizan cuando Timemator está en primer plano** o cuando el widget se refresca. Si Timemator lleva tiempo cerrado, los datos pueden estar desactualizados.

2. **Sin lista de proyectos**: Los archivos del widget no contienen la lista completa de proyectos/tareas. Para saber qué proyectos tienes, usa la exportación CSV desde Timemator.

3. **`segments` sin nombre de tarea**: El desglose por tarea en los gráficos usa `taskId` (formato CoreData), no nombres. Para reportes con nombres de tarea por día, combina con el archivo `timer` o usa exportación CSV.

4. **Timer en curso**: Cuando el timer está corriendo, `currentSession.currentSessionBeginTimestamp` da la hora de inicio, pero la duración exacta hay que calcularla con `datetime.now()`.
