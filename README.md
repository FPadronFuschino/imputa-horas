# imputa-horas

Herramientas para **imputar horas de trabajo en JIRA** (Atlassian) del proyecto
**ALFA6 / INTERF** de Interfacom/Taxitronic, cerrando cada día laborable a su
objetivo (8h o 8h30) y repartiendo el tiempo entre Daily, Reuniones, Soporte y un
proyecto de relleno de desarrollo, **respetando siempre lo que ya haya imputado**.

Se apoya en el **MCP de Atlassian (Rovo)** para leer y crear worklogs.

## Dos formas de usarlo

| Entregable | Qué es | Cómo se usa |
| --- | --- | --- |
| `imputacion-horas.html` | Panel web autocontenido (un solo fichero) | Se publica como **Artifact** en claude.ai; se opera con botones y tablas |
| `imputa-horas-skill/` | **Agent Skill** de claude.ai | Se sube en Settings → Capabilities → Skills; se opera conversando |

Ambos implementan las mismas reglas de negocio (ver [`RULES.md`](RULES.md)).

### El panel (`imputacion-horas.html`)

Interfaz gráfica: comprobar un día, comprobar/cerrar la semana, ver la propuesta
de reparto e imputar con un clic. Llama a la API de Anthropic con `mcp_servers`
de Atlassian y guarda su configuración (tickets del mes, festivos) en el
almacenamiento del artefacto.

**Publicar**: subirlo como Artifact en claude.ai (no como fichero suelto — el
`fetch` a la API y el almacenamiento solo funcionan dentro del runtime del
artefacto).

### La Skill (`imputa-horas-skill/`)

Flujo conversacional (*"imputa hoy"*, *"cierra la semana"*, *"¿cuánto me falta
hoy?"*). Llama directamente a las herramientas MCP de Atlassian y **resuelve los
tickets del mes por título** (`Reuniones {mes} {año}`, `Tareas soporte
{mes} {año}`), guardándolos en memoria; si no los encuentra, pregunta. Ver
[`MEMORY.md`](MEMORY.md).

**Empaquetar y publicar**:

```bash
rm -f imputa-horas-skill.zip && zip -r imputa-horas-skill.zip imputa-horas-skill -x '*.DS_Store'
```

Luego: claude.ai → Settings → Capabilities → Skills → Upload skill → `imputa-horas-skill.zip`.

## Reglas de imputación (resumen)

- **Objetivo por día**: Lunes/Miércoles 8h30; Martes/Jueves/Viernes 8h;
  findes y festivos no laborables.
- **Reparto**: Daily 30m → Reuniones Varias (tope 120m) → Soportes Varios
  (tope 180m) → Desarrollo (`TFEX-5`, resto).
- Nunca imputa sin confirmación; nunca duplica lo ya imputado.

Detalle completo en [`RULES.md`](RULES.md).

## Requisitos

- Cuenta de claude.ai con el **MCP de Atlassian (Rovo)** conectado.
- Para la Skill, memoria de claude.ai activada (recomendado, no obligatorio).
- `cloudId` de JIRA: `taxitronic.atlassian.net`.

## Documentación del repositorio

- [`RULES.md`](RULES.md) — reglas de negocio (fuente de verdad).
- [`MEMORY.md`](MEMORY.md) — contrato de memoria de la Skill.
- [`CLAUDE.md`](CLAUDE.md) — guía para Claude Code al trabajar en este repo.
