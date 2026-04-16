# Guía de configuración: Timemator + Claude

## Paso 1: Permiso de Accesibilidad (necesario para control de timer)

Para que Claude pueda iniciar y parar timers en Timemator:

1. Abre **Preferencias del Sistema** (o Ajustes del Sistema en macOS Ventura+)
2. Ve a **Privacidad y Seguridad → Accesibilidad**
3. Añade la app que ejecuta los comandos de Claude (Terminal, o la app de Claude si aparece)
4. Activa el interruptor junto a ella

Sin este permiso, las funciones de **consulta y reportes** (que leen los archivos directamente)
seguirán funcionando. Solo el control del timer (start/stop) queda limitado.

## Paso 2: Configurar atajo global en Timemator (recomendado)

Para que el control del timer sea más fiable:

1. Abre **Timemator → Preferencias → Atajos**
2. Configura un atajo para "Iniciar/Pausar temporizador" (recomendado: `⌘⇧T`)
3. Anota el atajo que uses, el skill lo necesita para enviarlo

## Paso 3: Verificar ubicación de datos

Los datos de Timemator están en una de estas rutas:

```
~/Library/Application Support/Timemator/
~/Library/Containers/com.catforce.Timemator/Data/Library/Application Support/Timemator/
```

Puedes verificar cuál usa ejecutando:
```bash
ls ~/Library/Application\ Support/Timemator/ 2>/dev/null || \
ls ~/Library/Containers/com.catforce.Timemator/Data/Library/Application\ Support/Timemator/ 2>/dev/null
```

## Troubleshooting

| Problema | Causa probable | Solución |
|----------|---------------|----------|
| "not allowed assistive access" | Sin permiso de Accesibilidad | Ver Paso 1 |
| Timer no responde | Timemator cerrado | Se abrirá automáticamente |
| Datos vacíos en reporte | Sin entradas en el periodo | Verificar rango de fechas |
| Error al exportar CSV | Diálogo de Guardar en idioma distinto | Revisar idioma del sistema |
