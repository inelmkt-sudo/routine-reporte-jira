# Reporte diario Jira — INEL

## Objetivo

Generar un reporte ejecutivo de actividad del día en curso (00:00 America/Lima → momento de ejecución) de todos los proyectos Jira de INEL. **Entrega única: el reporte completo en texto Markdown como mensaje final de la corrida.** No subir a OneDrive, no generar docx, no hacer git push.

## Regla #1 — Autonomía total

Operas sin humano. No preguntas, no pides confirmación. Si algo es imposible (MCP desconectado), detente con nota clara y exit ≠ 0.

## Parámetros fijos

- **cloudId**: `04767bab-fc3e-4d7e-b8c1-d28fc7b2edea`
- Proyectos: todos (POD1, POD2, KAN, OPS, PI2026, MEDD)
- Zona horaria: America/Lima

## Procedimiento (4 pasos)

### 1. Búsqueda principal

`searchJiraIssuesUsingJql`:
- `jql`: `updated >= startOfDay() ORDER BY updated DESC`
- `maxResults`: 100 — **paginar** hasta agotar `pageInfo.hasNextPage`
- `fields`: `["summary","status","assignee","project","updated","created","duedate","issuetype","parent","comment"]`
- Verificar conteo: procesados == total devuelto por Jira
- NUNCA filtrar por nombre en JQL (tildes fallan). Usar accountId si hace falta.

### 2. Changelogs — solo donde importa

**Presupuesto**: 1 changelog por issue que lo requiera, lanzados en lotes paralelos de 5-7.

Pedir changelog SOLO a:
- (a) Issues con `created` anterior a hoy
- (b) Issues con `statusCategory.key = "done"`
- (c) Issues con `statusCategory.key = "indeterminate"`

`getJiraIssue` con `expand: "changelog"`, `fields: ["summary","duedate","status"]`.

Del changelog extraer SOLO entradas con `created` de hoy. Procesar cada lote INMEDIATAMENTE: extraer eventos del día, calcular horas, y **descartar el JSON crudo de tu contexto mental** — retén solo los datos procesados. Las reglas de horas laborales y alertas están en `scripts/BUSINESS_RULES.md` (léelo UNA vez al inicio de este paso).

### 3. Iniciativas transversales — SIN búsquedas extra

Detectar iniciativas usando SOLO los issues del paso 1 (0 llamadas MCP adicionales):
- Agrupar por palabras clave en summary (campañas, eventos, códigos de curso, países)
- Una iniciativa es transversal si cruza 2+ integrantes o 2+ proyectos
- Avance = solo issues del día (marcar como "avance parcial del día")

### 4. Mensaje final = el reporte completo

Lee `scripts/FORMAT.md` y genera el reporte con las 4 secciones exactas como mensaje final. Es la ÚNICA entrega. Nada de OneDrive, nada de docx.

## Eficiencia — reglas duras

- Un día típico ≈ 10-18 llamadas MCP en total (2-3 páginas búsqueda + changelogs en lotes)
- NO retener JSON crudo de Jira en contexto — procesar y descartar
- NO buscar avance total de iniciativas (solo datos del día)
- NO intentar render docx ni upload a OneDrive
- Las specs de formato y reglas de negocio están en archivos auxiliares — leerlos UNA vez, no cargarlos en cada turno
