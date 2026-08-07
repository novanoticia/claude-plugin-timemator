# Plugin: Timemator para Claude

Controla Timemator desde Claude usando lenguaje natural o el comando `/timemator`.

> **Compatible con [Agent Plugins 1.0.0](https://agent-plugins.org/specification)** — el formato portátil de empaquetado de la Agentic AI Foundation (OpenAI, Amazon,
> Microsoft, Cursor y Vercel, con Google como *core maintainer*).
> El paquete lleva el manifiesto portable `plugin.json` en la raíz del plugin y
> el skill en `skills/timemator/SKILL.md`, así que cualquier cliente conformante lo descubre.
>
> **Qué necesita el cliente para ejecutarlo.** El skill viaja a cualquier cliente
> conformante, pero para *controlar Timemator de verdad* el cliente tiene que poder
> ejecutar `osascript` en el mismo Mac donde está la app, con permisos de accesibilidad
> para el GUI scripting. Con esa capacidad funciona; sin ella el skill se carga y puede
> explicar la operación, pero no actúa sobre la app. Está declarado en el campo
> `compatibility`, que es donde la especificación de Agent Skills manda poner los
> requisitos de entorno.
>
> Deliberadamente no lleva una lista de qué productos pueden y no pueden: las
> capacidades de cada cliente cambian y una lista así envejece mal. Si no sabes si el
> tuyo puede, pídele que ejecute
> `osascript -e 'tell application "Timemator" to activate'` y observa si la app se abre.

## Qué puede hacer

- **Iniciar y parar timers** para cualquier proyecto
- **Consultar tiempo trabajado** hoy o esta semana
- **Crear entradas manuales** ("registra 2h de diseño de ayer")
- **Generar reportes** por proyecto y período

## Configuración inicial (una sola vez)

### 1. Permiso de Accesibilidad

Para controlar los timers, macOS necesita que autorices el acceso de asistencia:

1. **Preferencias del Sistema → Privacidad y Seguridad → Accesibilidad**
2. Añade y activa la app que ejecuta Claude (Terminal o la app de Claude)

> Sin este permiso, los reportes y consultas de tiempo seguirán funcionando.
> Solo el control de timer (start/stop) requiere este permiso.

### 2. Requiere

- macOS 10.15 o superior
- Timemator instalado en `/Applications/Timemator.app`
- MCP "Control your Mac" conectado en Claude

## Uso

```
/timemator start Proyecto X      → inicia timer para "Proyecto X"
/timemator stop                  → para el timer activo
/timemator status                → ¿qué está corriendo ahora?
/timemator today                 → resumen del día
/timemator week                  → resumen semanal
/timemator report 14             → reporte de los últimos 14 días
/timemator add 2h Diseño ayer    → entrada manual
```

También puedes hablar en lenguaje natural:
- "¿Cuánto tiempo llevo trabajando hoy?"
- "Para el timer"
- "Registra una hora de reuniones esta mañana"

## Cómo funciona

Timemator no tiene API ni AppleScript nativo, así que el plugin usa tres estrategias:

1. **Control de UI**: GUI scripting vía System Events para start/stop
2. **Lectura directa**: Lee los archivos de datos de Timemator (no requiere permisos extra)
3. **Exportación CSV**: Para reportes detallados

## Archivos del plugin

```
timemator/
├── commands/timemator.md           ← comando /timemator
└── skills/timemator/
    ├── SKILL.md                    ← lógica principal
    └── references/
        ├── setup-guide.md          ← configuración y troubleshooting
        └── data-reading.md         ← scripts Python para leer datos
```

## Limitaciones conocidas

- El control de timer requiere que Timemator esté abierto (se abre automáticamente si no lo está)
- La creación de entradas manuales depende del idioma del sistema operativo para los menús
- La lectura de datos puede requerir ajustes si Timemator cambia su formato interno

---

*Desarrollado por Pablo • Plugin experimental — puede requerir ajustes según tu configuración*
