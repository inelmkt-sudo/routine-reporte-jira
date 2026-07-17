# Routine: Reporte diario Jira — INEL

Genera cada día un Word (.docx) con la actividad de las últimas 24h en todos los proyectos Jira: actividad por integrante, cambios de campos (fechas de vencimiento movidas, info borrada), comentarios, jerarquía epic → tarea → subtarea, y alertas de trazabilidad para el líder.

Los reportes quedan en `reports/reporte-jira-YYYY-MM-DD.docx` dentro de este mismo repo.

## Cómo conectarlo

1. Sube esta carpeta a un repo **privado** de GitHub (ej. `routine-reporte-jira`).
2. Ve a `claude.ai/code/routines` → **New routine**.
3. Pega este prompt:

```
Lee CLAUDE.md y ejecuta la automatización descrita ahí. Operas con autonomía
plena: no preguntas, no pides confirmación, no esperas input — nadie puede
contestarte. Tienes libertad total para decidir según las instrucciones y tu
juicio. Si algo es genuinamente imposible, detente con nota clara y exit error.
Respeta la asignación MCP vs script del CLAUDE.md. Reporta resumen al final.
```

4. Selecciona el repo.
5. Environment custom: setup command `bash setup.sh`. Sin variables de entorno. Network access: **Full**.
6. Connectors: deja **Atlassian** (Jira) y **Composio** (para subir el reporte a OneDrive). Quita los demás.
7. Trigger: schedule diario a la hora que prefieras (ej. 18:00 America/Lima).
8. **Run now** y revisa el primer run: el reporte completo debe aparecer como texto en el chat de la corrida, y el archivo en la carpeta "Reportes Jira" del OneDrive conectado en Composio.

## Dónde ver el reporte

- En GitHub: carpeta `reports/` del repo (descarga el .docx).
- Opcional: clona el repo dentro de tu OneDrive y haz `git pull` para tenerlo sincronizado localmente.
