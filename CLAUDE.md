# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Qué es este repositorio

Contiene **dos entregables que implementan la misma lógica de negocio**: imputar
horas de trabajo en JIRA (Atlassian) para el proyecto **ALFA6 / INTERF** de
Interfacom/Taxitronic, cerrando cada día laborable a su objetivo (8h o 8h30).

1. **`imputacion-horas.html`** — panel web autocontenido (HTML+CSS+JS en un solo
   fichero) pensado para publicarse como **Artifact de claude.ai**. Llama a
   `https://api.anthropic.com/v1/messages` con `mcp_servers` de Atlassian Rovo y
   persiste su configuración en `window.storage` (sandbox del artefacto).
2. **`imputa-horas-skill/SKILL.md`** — **Agent Skill de claude.ai** que hace lo
   mismo de forma conversacional, llamando directamente a las herramientas MCP de
   Atlassian (sin el `fetch` ni el parseo de JSON del panel) y recordando estado
   en la memoria persistente de claude.ai. Es el **fuente único** de la Skill.
   `.claude/skills/imputa-horas/SKILL.md` es un **symlink** a este fichero, para
   que la Skill funcione también como skill de proyecto de Claude Code sin
   duplicar contenido.

No hay sistema de build, dependencias ni tests: son artefactos que se publican/
suben tal cual.

## Fuente de verdad de la lógica

Las reglas de imputación (objetivos por día, cascada de reparto, topes,
resolución de tickets del mes) están descritas en **`RULES.md`** y **replicadas
en los dos entregables**:

- HTML: funciones `targetForDate`, `computeAllocation`, y los prompts dentro de
  `callClaude`/`createWorklogs` en `imputacion-horas.html`.
- Skill: secciones *Objetivo de horas por día*, *Lógica de reparto* y *Tickets
  del mes* en `imputa-horas-skill/SKILL.md`.

**Al cambiar una regla hay que reflejarla en los tres sitios** (`RULES.md`, el
HTML y el `SKILL.md`) o divergen. `RULES.md` manda; si hay discrepancia, es un
bug en el entregable, no en `RULES.md`.

Diferencia clave entre entregables: el HTML resuelve los tickets del mes pidiendo
al usuario que los configure (`window.storage`); la Skill los **resuelve por
título** (`Reuniones {mes} {año}` / `Tareas soporte {mes} {año}`) y los cachea en
memoria (ver `MEMORY.md`).

## Comandos habituales

```bash
# Reempaquetar la Skill tras editar SKILL.md (para subir a claude.ai)
rm -f imputa-horas-skill.zip && zip -r imputa-horas-skill.zip imputa-horas-skill -x '*.DS_Store'

# Ver el panel en un navegador local (no ejecuta las llamadas MCP fuera de claude.ai)
xdg-open imputacion-horas.html
```

- **Publicar el panel**: se sube como Artifact en claude.ai (no como fichero
  suelto). El `fetch` a la API y `window.storage` solo funcionan dentro del
  runtime del artefacto.
- **Publicar la Skill**: claude.ai → Settings → Capabilities → Skills → Upload
  skill, con `imputa-horas-skill.zip`.

## Convenciones del entorno

- Idioma del proyecto y de las descripciones de worklog: **español**. Meses en
  minúscula (`enero`…`diciembre`) para casar con los títulos de ticket.
- Zona horaria de referencia: **Europa/Madrid**.
- `cloudId` de JIRA: `taxitronic.atlassian.net`.
- Crear worklogs es una acción con efecto externo: **siempre confirmar con el
  usuario antes de imputar** (ver `RULES.md`).
