# RULES.md — Reglas de negocio de imputación (ALFA6 / INTERF)

Fuente de verdad de la lógica de imputación. El panel HTML y la Agent Skill
**replican** estas reglas; ante cualquier discrepancia, este fichero manda.

## Objetivo de horas por día

Según el día de la semana (zona horaria de Madrid):

| Día                         | Objetivo        |
| --------------------------- | --------------- |
| Lunes, Miércoles            | 510 min (8h30)  |
| Martes, Jueves, Viernes     | 480 min (8h)    |
| Sábado, Domingo             | No laborable    |

Días no laborables adicionales (festivos, vacaciones, ausencias) no se imputan.
Los fines de semana se excluyen automáticamente.

## Cascada de reparto

Dado lo **ya imputado** ese día (`totalLogged`) y el objetivo (`target`), se
calculan las líneas a añadir en este orden. Los topes se aplican sobre el total
del día (ya imputado + nuevo) por concepto:

1. **Daily — 30 min** en el ticket de **Reuniones**, descripción `Daily`.
   Solo si ese día NO existe ya un worklog en Reuniones con "Daily". Se suma al
   efectivo.
2. `gap = target − efectivo`. Si `gap ≤ 0`, no se añade nada más.
3. **Reuniones Varias** (ticket de Reuniones, descripción `Reuniones Varias`),
   tope **120 min/día**: añadir `min(gap, 120 − "Reuniones Varias" ya imputado)`.
4. **Soportes Varios** (ticket de Soporte, descripción `Soportes Varios`),
   tope **180 min/día**: añadir `min(gap, 180 − "Soportes Varios" ya imputado)`.
5. **Desarrollo / relleno** (proyecto grande, `TFEX-5` por defecto; alternativa
   `ALFA6-1306`), descripción `Desarrollo`: añadir el `gap` restante, sin tope.

Descripciones literales de worklog: `Daily`, `Reuniones Varias`,
`Soportes Varios`, `Desarrollo`.

## Tickets del mes

Los tickets de Reuniones y Soporte cambian cada mes. Títulos esperados (mes en
español, minúscula; p. ej. `julio 2026`):

- Reuniones: `Reuniones {mes} {año}`
- Soporte: `Tareas soporte {mes} {año}`

Resolución (ver detalle en `SKILL.md` y contrato de caché en `MEMORY.md`):
memoria → búsqueda por título en JIRA (`project = INTERF AND summary ~ ...`) →
si no se encuentra, preguntar al usuario.

## Invariantes

- **Nunca imputar sin confirmación explícita del usuario** (crear worklogs tiene
  efecto externo e irreversible en la práctica).
- **Respetar siempre lo ya imputado**: la cascada descuenta lo existente; no
  duplicar el `Daily` si ya existe.
- Filtrar worklogs por `author.accountId` del usuario autenticado Y por fecha
  exacta (`started`, `YYYY-MM-DD`).
- **Nunca leer el histórico completo de un ticket ni las entradas de otros
  compañeros.** Reuniones y Soporte son tickets compartidos por decenas de
  personas; solo interesan los worklogs propios del día. Lectura acotada: usar
  el resultado del JQL por `worklogAuthor = currentUser()` para saber en qué
  issues imputó el usuario (los que no aparezcan = 0, no se leen), y en esos
  leer solo la fecha buscada (filtro por fecha si existe, o página pequeña desde
  el final) con tope duro (~100 worklogs/issue). Si no se aíslan las entradas
  propias dentro del tope, preguntar al usuario en vez de seguir leyendo.
- En memoria se guarda **solo el enlace/clave** de los tickets del mes, nunca
  sus entradas.

## Limitación conocida del MCP (Rovo)

`getJiraIssue` devuelve solo la **primera página** del worklog (~20 entradas,
las más antiguas) y **no filtra por fecha ni pagina fiable**. Consecuencias:

- Tickets con pocas entradas (ALFA6 propios): se leen bien; se filtra por
  autor+fecha sin problema.
- Tickets compartidos de mucho volumen (INTERF Reuniones/Soporte, cientos de
  worklogs): la entrada propia del día **no es legible**. Se detecta la
  *presencia* con el JQL `worklogAuthor = currentUser() AND worklogDate = X`,
  pero el *importe* se pide al usuario (no se asume 0).
- Un día en fresco (sin nada imputado) no requiere leer tickets compartidos ni
  preguntar: se aplica la cascada completa directamente.
