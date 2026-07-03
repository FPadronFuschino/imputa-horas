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

**`fpi`** ("lo que falta por imputar") = `objetivo − ya imputado ese día` (todo
concepto ya registrado, no solo lo de este cálculo). Se calcula una vez y decide
el tramo de Reuniones Varias.

Líneas a añadir, en orden:

1. **Daily — 30 min (OBLIGATORIO)** en el ticket de **Reuniones**, descripción
   `Daily`. El ticket de Reuniones debe tener siempre al menos estos 30m.
   Añádelo salvo que ese día ya exista un worklog `Daily` (no duplicar).
2. **Reuniones Varias** (ticket de Reuniones, descripción `Reuniones Varias`),
   importe fijo según `fpi`:
   - `fpi ≥ 5h` (≥300 min) → **+1h30 (90 min)** → Reuniones total 2h.
   - `3h ≤ fpi < 5h` (180–299 min) → **+1h (60 min)** → Reuniones total 1h30.
   - `fpi < 3h` (<180 min) → **nada** → Reuniones total 30m (solo Daily).
   Añade `min(gap restante, objetivo_del_tramo − "Reuniones Varias" ya imputado)`.
3. **Soportes Varios** (ticket de Soporte, descripción `Soportes Varios`),
   tope **180 min/día**: `min(gap restante, 180 − "Soportes Varios" ya imputado)`.
4. **Desarrollo / relleno** (proyecto grande, `TFEX-5` por defecto; alternativa
   `ALFA6-1306`), descripción `Desarrollo`: el `gap` restante, sin tope.

`gap restante` = objetivo − (ya imputado + lo añadido en los pasos anteriores).
Descripciones literales de worklog: `Daily`, `Reuniones Varias`,
`Soportes Varios`, `Desarrollo`.

Ejemplos (objetivo 8h30 = 510 min):

| Ya imputado | fpi   | Daily | Reuniones Varias | Soportes Varios | Desarrollo |
| ----------- | ----- | ----- | ---------------- | --------------- | ---------- |
| 0           | 8h30  | 30m   | 1h30             | 3h              | 3h30       |
| 4h (ALFA6)  | 4h30  | 30m   | 1h               | 3h              | —          |
| 6h30        | 2h    | 30m   | —                | 1h30            | —          |

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
