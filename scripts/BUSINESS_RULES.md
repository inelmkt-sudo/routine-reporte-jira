# Reglas de negocio — Horas laborales y alertas

Lee este archivo en el paso 0 (pre-carga), ANTES de la búsqueda JQL. Así las reglas están en contexto cuando procesas changelogs en paso 2 sin necesidad de releer. No lo recargues — si el sistema comprime, las reglas sobreviven en el resumen comprimido porque son pocas y están al inicio del contexto.

## Horas laborales (America/Lima)

- Jornada: 8:00–13:00 y 14:00–20:00 (almuerzo 13:00–14:00 no cuenta) = 11h/día max
- Fuera de 8:00–20:00 no corre el reloj
- Sábados y domingos no cuentan
- Ejemplo: inicio martes 18:35, fin miércoles 09:15 → 1.4h (18:35–20:00) + 1.25h (8:00–9:15) = 2.7h

## Tiempo de ciclo (grupo b — completados)

- Inicio = primera transición a "En progreso" (si nunca pasó, usar `created`)
- Fin = transición al estado done
- `horas = fin - inicio` en horas laborales, 1 decimal
- Si pasó por "Bloqueado", calcular horas bloqueado aparte: "18.5h total, 11h bloqueado"
- Sin fase en progreso (Pendiente → Done directo): marcar "Xh* (sin fase en progreso)"

## Horas en curso (grupo c — indeterminate)

- `horas = ahora - primera transición a En progreso`, formato "Xh en curso"
- ALERTA si >22h laborales (~2 días) en curso
- ALERTA si >11h laborales (~1 día) bloqueado

## Alertas a detectar

1. **Regresiones de estado**: issue retrocede (En progreso → Pendiente, Done → reabierto). Reportar con horas en estado previo.
2. **Vencimiento movido 2+ veces**: contar TODAS las entradas `duedate` del changelog completo. Señal de tarea pateada.
3. **Vencimiento movido hoy**: reportar cuántos días se corrió.
4. **Campos borrados**: `toString` null/vacío y `fromString` con valor → ALERTA.
5. **Issues sin asignar**: assignee null.
6. **Issues vencidos abiertos**: duedate pasado y no en statusCategory done.

## Campos del changelog

Cada `histories[]` tiene `author.displayName`, `created`, e `items[]` con `field`, `fromString`, `toString`.

Cambios de interés: `duedate`, `status`, `assignee`, `description`.

## Jerarquía de issues

- `issuetype.hierarchyLevel`: 1=epic, 0=tarea, -1=subtarea
- `parent` da epic/tarea padre (parent.key, parent.fields.summary)
- `issuetype.subtask` = true → subtarea

## Comentarios

`fields.comment.comments[]` con `author.displayName`, `created`, `body`. Solo incluir los creados hoy.
