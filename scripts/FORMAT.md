# Formato exacto del reporte

Lee este archivo UNA sola vez cuando vayas a generar el mensaje final (paso 4). No lo cargues antes.

## Título

```
Reporte diario Jira — {día_semana} {DD/MM/YYYY} (parcial, corte {HH:MM})
```

## Sección 1 — Resumen ejecutivo

Tabla (1 fila):

| Issues tocados | Creados | Completados | En progreso | Bloqueados |
|---|---|---|---|---|
| N | N | N | N | N |

## Sección 2 — Alertas de trazabilidad

Viñetas en este orden fijo:
1. Regresiones de estado
2. Vencimientos movidos 2+ veces
3. Vencimientos movidos hoy / patrón de arrastre
4. Registros retroactivos
5. Issues sin asignar
6. Issues vencidos abiertos

Formato: `{KEY} «{título ≤60 chars}» ({asignado}): {qué pasó} — {horas si aplica}`

Sin alertas → "Sin alertas: no se detectaron fechas movidas, borrados ni tareas vencidas."

## Sección 3 — Iniciativas transversales

| Iniciativa | Avance (del día) | Pods | Integrantes | Issues (hoy) |
|---|---|---|---|---|
| Nombre | N% (X/Y done, Z en curso, W pend.) | POD1, POD2 | César, Alexis | POD1-1102, KAN-582 |

Sin iniciativas → "No se detectaron iniciativas transversales en esta ventana."

## Sección 4 — Actividad por integrante (detalle completo)

Integrantes ordenados por nº de issues desc. "Sin asignar" al final.

Encabezado por persona:
```
### {nombre} — {N} issue(s)
**Horas de ciclo del día: {X}h en {N} tareas completadas** (o "0h (sin completadas)")
```

Tabla con TODAS sus issues (orden: completadas → en curso → pendientes):

| Issue | Resumen | Estado | Acción | Horas ciclo | Vence | Pertenece a |
|---|---|---|---|---|---|---|
| KEY (PROJ) | texto | estado | creado/completado/actualizado/comentado/regresión | N.Nh / N.Nh en curso / — | DD/MM / — | KEY — título / — |

Debajo de la tabla, viñetas solo para issues con cambios/comentarios:
- `{KEY} — Cambio: {campo}: {antes} → {después} ({autor}, {HH:MM})`
- `{KEY} — Comentario: {autor} ({HH:MM}): {texto}`

## Reglas duras

- Mismos títulos siempre, ninguna sección se omite
- Ningún integrante se resume ("y 5 más" prohibido)
- Fechas DD/MM, horas HH:MM America/Lima, horas laborales con 1 decimal
