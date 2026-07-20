# Reporte diario Jira — INEL

## Objetivo

Generar un reporte ejecutivo de actividad del día en curso (00:00 America/Lima → momento de ejecución) de todos los proyectos Jira de INEL. **Entrega única: el reporte completo en texto Markdown como mensaje final de la corrida.** No subir a OneDrive, no generar docx, no hacer git push.

## Regla #1 — Autonomía total

Operas sin humano. No preguntas, no pides confirmación. Si algo es imposible (MCP desconectado), detente con nota clara y exit ≠ 0.

## Parámetros fijos

- **cloudId**: `04767bab-fc3e-4d7e-b8c1-d28fc7b2edea`
- Proyectos: todos (POD1, POD2, KAN, OPS, PI2026, MEDD)
- Zona horaria: America/Lima

## ARQUITECTURA ANTI-OVERFLOW — léela antes de hacer CUALQUIER cosa

El enemigo es el contexto. Cada tool result se queda físicamente en la ventana hasta que el sistema comprime. Si acumulas >80% se dispara auto-compaction y pierdes datos intermedios. **Todo el procesamiento pesado va a disco, nunca en contexto.**

### Scratchpad como memoria de trabajo

Usa el directorio scratchpad para TODO dato intermedio. El flujo es: MCP → disco → leer de disco solo lo mínimo.

### Pre-cargar schemas MCP (paso 0)

ANTES de cualquier llamada Jira, carga los schemas con UN solo ToolSearch:
```
ToolSearch query: "select:mcp__818f3f1a-ecd0-4f31-9ace-74259ac7ded5__searchJiraIssuesUsingJql,mcp__818f3f1a-ecd0-4f31-9ace-74259ac7ded5__getJiraIssue"
```
Esto elimina 2-3 ToolSearch posteriores que desperdician round-trips.

## Procedimiento

### Paso 0 — Pre-carga

1. ToolSearch con los 2 schemas de Jira (ver arriba)
2. Leer `scripts/BUSINESS_RULES.md` — ÚNICA lectura de reglas de negocio para toda la corrida
3. NO leer FORMAT.md todavía (se lee en paso 4)

### Paso 1 — Búsqueda principal (volcado directo a disco)

`searchJiraIssuesUsingJql`:
- `jql`: `updated >= startOfDay() ORDER BY updated DESC`
- `maxResults`: 100 — **paginar** hasta agotar `pageInfo.hasNextPage`
- `fields`: `["summary","status","assignee","project","updated","created","duedate","issuetype","parent","comment"]`
- NUNCA filtrar por nombre en JQL (tildes fallan). Usar accountId si hace falta.

**CRÍTICO — el resultado del JQL puede pesar 160k+ chars.** Inmediatamente después de CADA página:

1. Extraer con Bash/jq SOLO los campos necesarios de cada issue y APPENDEAR a `{scratchpad}/issues.jsonl` (una línea JSON por issue):
   ```bash
   # Pseudo — adaptar al formato real del MCP response
   echo '$RESPONSE' | jq -c '.[] | {key,summary:.fields.summary,status:.fields.status.name,statusCat:.fields.status.statusCategory.key,assignee:.fields.assignee.displayName,project:.fields.project.key,updated:.fields.updated,created:.fields.created,duedate:.fields.duedate,type:.fields.issuetype.name,hierarchyLevel:.fields.issuetype.hierarchyLevel,subtask:.fields.issuetype.subtask,parentKey:.fields.parent.key,parentSummary:.fields.parent.fields.summary,comments:[.fields.comment.comments[]|{author:.author.displayName,created:.created,body:.body}]}' >> issues.jsonl
   ```
2. NO retener el JSON crudo del MCP en tu razonamiento — ya está en disco.
3. Al terminar todas las páginas, verificar conteo: `wc -l issues.jsonl` == total reportado por Jira.

### Paso 2 — Changelogs (máximo paralelismo, volcado a disco)

Leer `{scratchpad}/issues.jsonl` con Bash para determinar qué issues necesitan changelog:
```bash
# Issues que necesitan changelog: creados antes de hoy, o done, o indeterminate
cat issues.jsonl | jq -r 'select(.statusCat == "done" or .statusCat == "indeterminate" or (.created | startswith("YYYY-MM-DD") | not)) | .key'
```

**Lanzar TODOS los changelogs en UN solo turno** (no en lotes de 5-7 — el límite real de parallel tool calls es ~20). Si son 23 issues, lanza 23 `getJiraIssue` en un solo mensaje. Si son >20, usa 2 turnos máximo.

`getJiraIssue` con `expand: "changelog"`, `fields: ["summary","duedate","status"]`.

Después de recibir CADA lote de respuestas, INMEDIATAMENTE volcar a disco con Bash:
1. Para cada issue, extraer solo los history entries de HOY y escribir a `{scratchpad}/changes.jsonl`:
   ```
   {key, changes: [{field, from, to, author, timestamp}], transitions: [{from, to, timestamp}]}
   ```
2. Calcular horas de ciclo/en curso/bloqueado AQUÍ MISMO en el pipe de Bash (las reglas de BUSINESS_RULES.md ya están en contexto del paso 0).
3. Appendear resultado a `{scratchpad}/calculated.jsonl`:
   ```
   {key, horas_ciclo, horas_bloqueado, horas_en_curso, alertas: [...]}
   ```

**NO** retengas los changelogs crudos en contexto. El dato útil está en `calculated.jsonl`.

### Paso 3 — Ensamblaje (todo en disco, lectura mínima)

Con Bash, leer `issues.jsonl` + `changes.jsonl` + `calculated.jsonl` y generar `{scratchpad}/report_data.json` con la estructura del reporte. El script Bash/jq o PowerShell hace:
- Agrupar issues por assignee
- Detectar iniciativas transversales (keywords en summary que cruzan 2+ personas o 2+ proyectos) — SIN llamadas MCP extra
- Calcular resumen (totales, creados, completados, en_progreso, bloqueados)
- Compilar alertas

### Paso 4 — Generar texto final

1. Leer `scripts/FORMAT.md` (ÚNICA lectura)
2. Leer `{scratchpad}/report_data.json`
3. Emitir el reporte completo como mensaje final con las 4 secciones exactas del formato

## Eficiencia — reglas duras

| Regla | Por qué |
|---|---|
| MCP → disco → leer mínimo | El MCP de Jira devuelve 5k-30k chars por issue. 23 changelogs = 115k-690k chars que NO deben quedarse en contexto |
| Changelogs en 1-2 turnos, no 5 | 23 calls en 1 turno = 1 round-trip. En lotes de 5 = 5 round-trips = 4 turnos desperdiciados |
| Pre-cargar schemas MCP | Evita 2-3 ToolSearch mid-run que cuestan un turno cada uno |
| BUSINESS_RULES.md se lee en paso 0 | Si se lee después del paso 1, el contexto ya está cargado con datos JQL y compite por espacio |
| FORMAT.md se lee en paso 4 | No necesita estar en contexto durante el procesamiento de datos |
| Nunca render docx ni upload OneDrive | Ahorra 3-5 turnos + las dependencias de Composio/Python |
| Presupuesto total: ≤8-12 llamadas MCP | 2-3 páginas búsqueda + 1-2 turnos de changelogs = máximo 5 turnos con datos Jira |
