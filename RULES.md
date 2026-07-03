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
- Eficiencia: en issues con mucho histórico (proyecto de relleno), leer las
  entradas más recientes primero, no todo el histórico.
