# Reporte diario de actividades Jira — INEL

## Objetivo

Generar cada día un reporte ejecutivo con la actividad del día en curso (desde las 00:00 de hoy, America/Lima, hasta el momento de la corrida) de todos los proyectos Jira de INEL, agrupado por integrante, incluyendo cambios de campos (changelog), comentarios y la jerarquía epic → tarea → subtarea.

**Doble entrega, en este orden de prioridad. PROHIBIDO hacer git push — GitHub solo aloja este código, no es canal de entrega:**
1. **Reporte completo en TEXTO como mensaje final de la corrida** (canal principal, nunca falla): el reporte entero con las 4 secciones del formato exacto, en Markdown. NO un resumen — el contenido íntegro.
2. **Archivo en OneDrive vía MCP de Composio** (canal de archivo): sube el reporte a la carpeta **"/09. Marketing/INSTITUTE/REGISTRO/Actividades"** (folder ID `01EJAD6P3ZU2IRMMSWJVELSYZOJJAG3Y26`, OneDrive de natalieaguirre@inelinc.com, conexión Composio `one_drive` ya activa).
   - Intento A: genera el docx con `scripts/render_docx.py` y súbelo con `ONE_DRIVE_ONEDRIVE_UPLOAD_FILE` (file con name `reporte-jira-YYYY-MM-DD.docx`, mimetype `application/vnd.openxmlformats-officedocument.wordprocessingml.document`, folder = el ID de arriba, `conflict_behavior: "replace"`).
   - Intento B (si el staging del binario falla): sube el reporte completo como texto con `ONE_DRIVE_ONEDRIVE_CREATE_TEXT_FILE` (name `reporte-jira-YYYY-MM-DD.md`, mismo folder ID, `conflict_behavior: "replace"`) — este camino está verificado y funciona.
   - Flujo Composio: `COMPOSIO_SEARCH_TOOLS` (use_case de subir archivo a OneDrive) → `COMPOSIO_MULTI_EXECUTE_TOOL` con el tool_slug exacto. La conexión ya existe; no crees conexiones nuevas.
   - Verifica la subida con `ONE_DRIVE_ONEDRIVE_FIND_FILE` (name exacto, mismo folder).

Éxito = reporte completo en texto en el mensaje final + archivo en OneDrive. Si OneDrive falla tras ambos intentos, no es fatal: el texto ya se entregó; menciona el fallo en una línea y exit 0.

## Regla #1 — Autonomía total

Operas sin humano presente. No preguntas, no pides confirmación, no esperas input. Decide con tu juicio. Si algo es genuinamente imposible (credencial faltante, MCP desconectado), detente con nota clara en stderr y exit ≠ 0.

## Asignación MCP vs script

| Operación | Vía | Detalle |
|---|---|---|
| Buscar issues actualizados | MCP Atlassian `searchJiraIssuesUsingJql` | Lectura |
| Changelog de cada issue | MCP Atlassian `getJiraIssue` con `expand: "changelog"` | Lectura |
| Generar .docx | Script `scripts/render_docx.py` | python-docx |
| Publicar reporte | MCP Composio → OneDrive (`ONE_DRIVE_ONEDRIVE_UPLOAD_FILE` / `CREATE_TEXT_FILE`) | NUNCA git push |

## Parámetros fijos

- **cloudId**: `04767bab-fc3e-4d7e-b8c1-d28fc7b2edea` (sitio `inelinc-team-j3j5pw8w.atlassian.net`)
- Proyectos: todos los visibles (POD1, POD2, KAN, OPS, PI2026, MEDD)
- Ventana: día en curso (startOfDay() hasta el momento de la ejecución, America/Lima)
- Zona horaria del equipo: America/Lima

## Procedimiento

### 1. Consulta principal (una sola llamada)

`searchJiraIssuesUsingJql` con:
**El día reportado es el día en curso al momento de la ejecución** (America/Lima). Tú determinas la fecha: consulta la fecha/hora actual y el reporte cubre desde las 00:00 de HOY hasta el momento de la corrida. Si el Routine corre el 15 a las 8am, reporta solo lo del 15 hasta las 8am — nunca una ventana de "últimas 24 horas", que mezclaría dos días.

- `jql`: `updated >= startOfDay() ORDER BY updated DESC` — trae todo lo tocado hoy. La pertenencia al día se confirma por eventos:
  - **Creada hoy** → entra (acción "creado").
  - **Creada antes y actualizada hoy** → entra; pide su changelog y reporta solo los eventos con fecha de hoy (el campo `updated` solo guarda la última modificación, no dice qué pasó hoy).
  - Nombra el archivo del reporte con la fecha de hoy. Si el Routine corre más de una vez el mismo día, sobrescribe el mismo archivo (el reporte del día se va completando).
- `searchResultMode`: `issues`
- `maxResults`: 100. **OBLIGATORIO paginar**: si `pageInfo.hasNextPage` es true, repite la búsqueda con `nextPageToken` hasta agotar los resultados. Un solo día del equipo puede superar los 100 issues; quedarse con la primera página deja integrantes con actividad invisible (falla real detectada: 8 entregas de un integrante quedaron en la página no consultada). Antes de generar el reporte, verifica el total contra la suma por integrante.
- `fields`: `["summary","status","assignee","project","updated","created","duedate","issuetype","parent","comment"]`
- `responseContentFormat`: `markdown`

Reglas aprendidas (respétalas, evitan fallos reales):
- NUNCA filtres por nombre de usuario en JQL (los nombres con tildes fallan por encoding). Si necesitas filtrar por persona usa `accountId`.
- Pide SOLO los campos listados: pedir más desborda el límite de tokens de la respuesta.
- El campo `parent` trae el epic de una tarea o la tarea padre de una subtarea (`parent.key`, `parent.fields.summary`). `issuetype.subtask` distingue subtareas; `issuetype.hierarchyLevel`: 1=epic, 0=tarea, -1=subtarea.
- El campo `comment` trae `fields.comment.comments[]` con `author.displayName`, `created`, `body`. Solo incluye en el reporte los comentarios creados dentro del día en curso.
- Textos pueden venir con mojibake (`Ã³` en vez de `ó`): el script de render los corrige, no te preocupes.

### 2. Changelog — solo donde importa

**Presupuesto de llamadas** (respétalo — es la métrica de eficiencia del sistema): 1-3 páginas de búsqueda principal + 1 changelog por issue que lo requiera (UNA sola llamada por issue aunque caiga en varios grupos — nunca repitas) + 1 búsqueda única para todas las iniciativas. Un día típico ≈ 15-25 llamadas en total.

**Changelogs EN PARALELO**: lanza las llamadas `getJiraIssue` en lotes de 5-7 dentro del mismo turno (múltiples tool calls en un solo mensaje) — son independientes entre sí. Nunca una por una en turnos separados: 14 changelogs secuenciales = 14 round-trips; en lotes paralelos = 2-3.

**Disciplina de contexto** (evita que la corrida se alargue o se comprima el contexto): procesa cada lote de changelogs INMEDIATAMENTE — extrae solo los eventos del día (cambios, transiciones, horas) y anéxalos a `data/report.json` en disco; no cargues ni retengas los JSON crudos de Jira en la conversación. `data/report.json` en disco es tu única memoria de trabajo: constrúyelo incrementalmente (búsqueda → añade issues; cada lote de changelogs → añade cambios/horas; iniciativas → añade avance) y al final el mensaje de texto se genera leyendo ese archivo una sola vez.

NO pidas changelog a: issues creados hoy que siguen en estado "new" (Pendiente/Por iniciar) sin más eventos — su única historia es la creación, ya la tienes del paso 1.

Pide changelog únicamente a estos grupos de issues del paso 1 (los demás no lo necesitan):
- **(a) Issues antiguos tocados**: `created` anterior a hoy — para detectar cambios de campos.
- **(b) Issues completados en la ventana**: estado con `statusCategory.key = "done"` (Entegrado, Finalizada, etc.) — para calcular el tiempo de ciclo.
- **(c) Issues actualmente en curso**: `statusCategory.key = "indeterminate"` (En progreso, En revisión, etc.) — para calcular horas parciales acumuladas.

Para cada uno llama `getJiraIssue` con:
- `expand`: `"changelog"`
- `fields`: `["summary","duedate","status"]`

Del changelog extrae solo las entradas (`histories[]`) con `created` dentro del día en curso. Cada entrada tiene `author.displayName` e `items[]` con `field`, `fromString`, `toString`. Cambios de interés prioritario:
- `duedate` (fecha de vencimiento movida — indica retraso o reprogramación)
- `status` (transiciones)
- `assignee` (reasignaciones)
- `description` u otros campos donde `toString` sea null/vacío y `fromString` no (posible borrado de información — márcalo como ALERTA)

Los issues creados DENTRO de la ventana y no completados no necesitan changelog (son nuevos; repórtalos como "creados").

**Tiempo de ciclo (horas)**: para cada issue del grupo (b), busca en el changelog:
- inicio = timestamp de la PRIMERA transición de status hacia "En progreso" (si nunca pasó por En progreso, usa `created`)
- fin = timestamp de la transición hacia el estado done
- `horas = fin - inicio`, redondeado a 1 decimal, contando SOLO horas laborales. Si el issue pasó por "Bloqueado", calcula también las horas bloqueado (mismas reglas) y repórtalas aparte (ej. "18.5h total, de las cuales 11h bloqueado").

**Horas laborales (America/Lima)** — toda resta de timestamps usa este calendario, nunca horas de reloj corrido:
- Jornada: 8:00–13:00 y 14:00–20:00 (el almuerzo 13:00–14:00 no cuenta) = máximo 11h por día.
- Fuera de 8:00–20:00 no corre el reloj.
- Sábados y domingos no cuentan.
- Ejemplo: inicio martes 18:35, fin miércoles 09:15 → 1.4h del martes (18:35–20:00) + 1.25h del miércoles (8:00–9:15) = **2.7h**, no 14.7h.
- Aplica igual a horas de ciclo, horas en curso, horas bloqueado y a los umbrales de alerta (48h en curso = ~4.4 días laborales; usa 22h laborales ≈ 2 días como umbral de "en curso demasiado tiempo" y 11h laborales ≈ 1 día para "bloqueado demasiado tiempo").

Para el grupo (c) — issues en curso sin terminar: `horas = ahora - primera transición a En progreso`, formato "Xh en curso". Marca como ALERTA todo issue con más de 48h en curso o más de 24h bloqueado.

**Regresiones de estado**: si en el changelog un issue retrocede (En progreso → Pendiente, Entegrado → reabierto, etc.), repórtalo como ALERTA con las horas que estuvo en el estado del que retrocedió (ej. "POD1-1143 retrocedió a Pendiente tras 2.7h laborales en progreso"). El estado ACTUAL de un issue no cuenta toda la historia: un "Pendiente" pudo haber estado en progreso y retroceder — por eso el grupo (a) incluye estos casos.

**Fecha corrida reincidente**: si el `duedate` de un issue se ha movido 2+ veces en su historia (cuenta todas las entradas `duedate` del changelog, no solo las de la ventana), márcalo como ALERTA: "vencimiento movido N veces" — es señal de tarea que se está pateando.

Las tareas que siguen en "Pendiente"/"Por iniciar" no tienen horas: el reloj arranca en la primera transición a En progreso. Si un equipo entrega tareas directo de Pendiente → Entegrado (sin pasar por En progreso), usa `created` como inicio y márcalo con asterisco: "Xh* (sin fase en progreso)".

### 3. Análisis macro — iniciativas transversales

Con los issues del paso 1 (sin llamadas adicionales a Jira), detecta iniciativas que cruzan integrantes y pods:

1. Extrae de cada `summary` los códigos y palabras clave significativas: nombres de campañas/eventos/programas (ej. "BESS Colombia", "Cyber", "fiestas patrias", "masterclass", "Energy Data", "IEEE", "preventa", "summit"), códigos de curso (ej. "CP. 21.42"), y países (Brasil, Colombia). Normaliza mayúsculas/tildes al comparar.
2. Agrupa los issues por palabra clave. Una iniciativa es transversal si sus issues abarcan **2+ integrantes o 2+ proyectos**.
3. Descarta palabras genéricas sin valor (correo, post, video, reunión, historia) salvo que formen parte de un nombre compuesto.
4. También usa `parent`: issues de distintos pods que comparten epic o cuyo epic tiene la misma palabra clave pertenecen a la misma iniciativa.

Esto responde la pregunta del líder: "¿quiénes están trabajando en lo mismo desde pods distintos?"

**Avance de cada iniciativa — UNA sola búsqueda para todas**: el avance se mide sobre TODOS los issues de cada iniciativa, no solo los de hoy. Combina todas las claves detectadas (máximo 8, prioriza las que cruzan más pods) en un único JQL con OR:
`summary ~ "clave1" OR summary ~ "clave2" OR ...`
con `fields: ["summary","status"]` (SOLO esos dos — más campos desborda tokens; el modo "count" no es confiable, devuelve issues completos), `maxResults: 100`, paginando si hay más. Luego clasifica localmente: asigna cada issue devuelto a su iniciativa comparando el summary contra las claves, y calcula por iniciativa total / completados (statusCategory done) / en curso / pendientes / porcentaje. Una llamada (o 2-3 páginas) en vez de una por iniciativa.

### 4. Construir el JSON intermedio

Escribe `data/report.json` con esta estructura exacta:

```json
{
  "fecha": "YYYY-MM-DD",
  "resumen": {
    "total_issues": 0,
    "creados": 0,
    "completados": 0,
    "en_progreso": 0,
    "bloqueados": 0,
    "alertas": ["texto de cada alerta: fechas movidas, info borrada, tareas sin asignar, vencidas"]
  },
  "iniciativas": [
    {
      "clave": "BESS Colombia",
      "pods": ["POD1", "POD2"],
      "integrantes": ["César", "Alexis"],
      "issues": ["POD1-1102", "POD1-1134", "KAN-582"],
      "avance": {"total": 3, "completados": 1, "en_progreso": 1, "pendientes": 1, "porcentaje": 33}
    }
  ],
  "integrantes": [
    {
      "nombre": "...",
      "total_horas_ciclo": "suma de horas_ciclo de sus issues completados, ej. '12.4h en 3 tareas completadas'",
      "issues": [
        {
          "key": "POD1-1102",
          "resumen": "...",
          "proyecto": "POD1",
          "tipo": "Tarea|Subtask|Epic",
          "padre": "POD1-1098 — Título del epic/tarea padre (o null)",
          "estado": "...",
          "vencimiento": "YYYY-MM-DD o null",
          "accion": "creado|completado|actualizado|comentado",
          "horas_ciclo": "125.5h (111h bloqueado) — solo para completados; null en los demás",
          "cambios": ["duedate: 2026-07-09 → 2026-07-14 (por César, 14/07 10:21)"],
          "comentarios": ["Autor (14/07 10:21): texto"]
        }
      ]
    }
  ]
}
```

En `resumen.alertas` incluye SIEMPRE, si aplica: fechas de vencimiento movidas en tareas antiguas (con cuántos días se corrió), campos borrados, issues sin asignar, e issues cuyo `duedate` ya pasó y siguen abiertos.

Regla de exhaustividad: en `integrantes[].issues` va **TODO** issue tocado por esa persona en la ventana — sin omitir, sin resumir, sin agrupar "y 5 más". El resumen ejecutivo condensa; el detalle es completo. La optimización está en el formato (el script lo renderiza como tabla compacta), no en recortar datos.

### 5. Render y publicación a OneDrive

```
pip show python-docx >/dev/null 2>&1 || pip install python-docx
python scripts/render_docx.py data/report.json
```

El script produce `reports/reporte-jira-YYYY-MM-DD.docx`. Luego súbelo a OneDrive según la sección "Doble entrega" del inicio (Composio, intento A docx / intento B md). NADA de git: ni add, ni commit, ni push.

**Composio SIN búsqueda previa**: los tool slugs ya están fijados en este documento (`ONE_DRIVE_ONEDRIVE_UPLOAD_FILE`, `ONE_DRIVE_ONEDRIVE_CREATE_TEXT_FILE`, folder ID `01EJAD6P3ZU2IRMMSWJVELSYZOJJAG3Y26`). Llama `COMPOSIO_MULTI_EXECUTE_TOOL` DIRECTAMENTE con ellos — no gastes llamadas en `COMPOSIO_SEARCH_TOOLS` salvo que la ejecución directa devuelva error de slug desconocido.

### 6. Mensaje final = el reporte completo en texto

El mensaje final de la corrida ES el reporte: las 4 secciones del formato exacto, completas, en Markdown (tablas incluidas). Prohibido resumir o remitir al archivo ("ver detalles en el docx" está prohibido). Al final agrega una línea de estado: "OneDrive: reporte-jira-YYYY-MM-DD.docx subido a Actividades ✓" (o el .md del intento B, o "subida a OneDrive falló: {motivo}").

## Formato EXACTO del reporte — no improvises

El docx tiene SIEMPRE estas 4 secciones, en este orden, con estos títulos literales. El script `render_docx.py` es la única fuente del formato: tu trabajo es llenar el JSON, nunca inventar secciones, columnas ni renombrarlas. Plantilla:

```
Reporte diario Jira — {día_semana} {DD/MM/YYYY} (parcial, corte {HH:MM})

1. Resumen ejecutivo
   Tabla (1 fila): Issues tocados | Creados | Completados | En progreso | Bloqueados

2. Alertas de trazabilidad
   Viñetas en este orden fijo: (1) regresiones de estado, (2) vencimientos movidos 2+ veces,
   (3) vencimientos movidos hoy / patrón de arrastre, (4) registros retroactivos,
   (5) issues sin asignar, (6) issues vencidos abiertos.
   Formato de cada viñeta: "{KEY} «{título de la tarea}» ({asignado}): {qué pasó} — {dato de horas si aplica}"
   El título de la tarea es OBLIGATORIO en cada alerta — el líder no memoriza códigos; "POD1-1134" solo no dice nada, "POD1-1134 «Estrategia mailing BESS Colombia»" sí. Si el título supera ~60 caracteres, recórtalo con "...".
   Si no hay alertas: la línea fija "Sin alertas: no se detectaron fechas movidas, borrados ni tareas vencidas."

3. Iniciativas transversales
   Tabla: Iniciativa | Avance | Pods | Integrantes | Issues (hoy)
   Avance con formato fijo: "N% (X/Y done, Z en curso, W pend.)"

4. Actividad por integrante (detalle completo)
   Integrantes ordenados por nº de issues descendente; "Sin asignar" siempre al final.
   Encabezado por persona: "{nombre} — {N} issue(s)"
   Línea siguiente: "Horas de ciclo del día: {X}h en {N} tareas completadas ({detalle bloqueado/asteriscos})"
     — si no completó nada: "Horas de ciclo del día: 0h (sin completadas)"
   Tabla (TODAS sus issues del día, orden: completadas → en curso → pendientes):
     Issue | Resumen | Estado | Acción | Horas ciclo | Vence | Pertenece a
     · Acción ∈ {creado, completado, actualizado, comentado, regresión}
     · Horas ciclo: "N.Nh" | "N.Nh* (sin fase en progreso)" | "N.Nh en curso" | "—"
     · Vence: "DD/MM" o "—" · Pertenece a: "KEY — título" o "—"
   Debajo de la tabla, viñetas solo para issues con cambios/comentarios:
     "{KEY} — Cambio: {campo}: {antes} → {después} ({autor}, {HH:MM})"
     "{KEY} — Comentario: {autor} ({HH:MM}): {texto}"
```

Reglas duras del formato: mismos títulos siempre; ninguna sección se omite (si está vacía, va con su texto de "sin datos"); ningún integrante se resume ("y 5 más" está prohibido); fechas DD/MM, horas HH:MM de America/Lima, horas laborales con 1 decimal.

## Variables de entorno

Ninguna. La autenticación Jira va por el connector Atlassian y la subida a OneDrive por el connector Composio (ambos MCP remotos).

## Scripts disponibles

- `scripts/render_docx.py <ruta-json>` — lee el JSON intermedio y genera el Word en `reports/`. Stdout: JSON `{"ok": true, "file": "..."}`. Exit ≠ 0 si el JSON está malformado.
