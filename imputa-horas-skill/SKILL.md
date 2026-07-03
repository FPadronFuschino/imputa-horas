---
name: imputa-horas
description: >-
  Imputa horas de trabajo en JIRA (Atlassian) para el proyecto ALFA6 / INTERF de
  Fernando (Taxitronic/Interfacom), cerrando cada día laborable a su objetivo (8h
  o 8h30) repartiendo el tiempo entre Daily, Reuniones, Soporte y el proyecto de
  relleno, respetando lo ya imputado. Úsala cuando el usuario pida "imputa hoy",
  "imputa el día X", "comprueba/cierra la semana", "cuánto me falta hoy", o
  cualquier tarea de imputación de worklogs en JIRA. Requiere el MCP de Atlassian
  (Rovo) conectado.
---

# Imputación de horas en JIRA (ALFA6 / INTERF)

Cierras cada día laborable a su objetivo de horas repartiendo entre Daily,
Reuniones, Soporte y un proyecto de relleno de desarrollo, **respetando siempre
lo que ya haya imputado** ese día. Trabajas contra JIRA usando exclusivamente las
herramientas MCP de Atlassian (Rovo).

## Requisitos previos

- El **MCP de Atlassian (Rovo)** debe estar conectado. Herramientas que usarás:
  `atlassianUserInfo`, `searchJiraIssuesUsingJql`, `getJiraIssue`,
  `addWorklogToJiraIssue`.
- Si el MCP no está disponible, dilo y detente — no inventes worklogs.

## Configuración

`cloudId`: **taxitronic.atlassian.net**

Proyecto grande de relleno (Desarrollo), por defecto: **TFEX-5** (alternativa:
`ALFA6-1306`).

### Tickets del mes (se resuelven solos por título)

Los tickets de Reuniones y Soporte son distintos cada mes. **No están fijados
aquí**: se buscan en JIRA por su título y se recuerdan. Los títulos siguen este
patrón (mes en español, en minúsculas; p. ej. `julio 2026`):

- **Reuniones:** `Reuniones {mes} {año}`
- **Soporte:** `Tareas soporte {mes} {año}`

**Procedimiento de resolución** (ejecútalo al empezar cualquier imputación,
usando el mes y año actuales en zona horaria de Madrid):

1. Calcula `{mes} {año}` actuales (p. ej. `julio 2026`).
2. **Mira en tu memoria persistente** si ya tienes resueltos los tickets para
   ese `{mes} {año}` (p. ej. una entrada tipo
   `imputa-horas: reuniones julio 2026 = INTERF-34, soporte julio 2026 = INTERF-35`).
   Si la tienes, úsala directamente — no vuelvas a buscar.
   > Como la memoria va indexada por mes+año, al cambiar de mes (el día 1) no
   > habrá entrada para el mes nuevo y se dispara automáticamente una búsqueda
   > fresca. Eso es la "verificación de cada 1 de mes".
3. Si no está en memoria, **búscalos en JIRA por título** con
   `searchJiraIssuesUsingJql`, `fields=["summary"]`:
   - Reuniones: `jql = 'project = INTERF AND summary ~ "Reuniones {mes} {año}"'`
   - Soporte:   `jql = 'project = INTERF AND summary ~ "Tareas soporte {mes} {año}"'`
   Verifica que el `summary` del issue devuelto coincide con el título esperado
   (comparación sin distinguir mayúsculas). Si hay exactamente un match claro,
   tómalo.
4. **Guárdalo en tu memoria persistente** para ese `{mes} {año}`, para no
   repetir la búsqueda en futuras sesiones.
5. Si **no encuentras** el ticket (0 resultados) o hay varios y ninguno coincide
   con claridad, **pregunta al usuario** la clave del ticket que falte
   (p. ej. "No encuentro el ticket de Reuniones de julio 2026, ¿cuál es su
   clave?"). Al recibirla, guárdala en memoria igual que en el paso 4.

El usuario también puede dar las claves en línea (*"imputa hoy,
reuniones=INTERF-40 soporte=INTERF-41"*); en ese caso úsalas y guárdalas en
memoria para ese mes.

## Objetivo de horas por día

Calcula el día de la semana de la fecha (zona horaria de Madrid):

- **Sábado / Domingo** → no laborable, nada que imputar.
- **Lunes y Miércoles** → 510 minutos (8h30).
- **Martes, Jueves y Viernes** → 480 minutos (8h).

Días no laborables adicionales (festivos, vacaciones, ausencias): si el usuario
menciona que un día no es laborable, o mantiene una lista aquí, respétalo y no
imputes ese día. Los fines de semana ya se excluyen solos.

<!-- Días no laborables conocidos (añadir según calendario Interfacom):
  - 2026-01-01  Festivo
-->

## Lógica de reparto

Dado lo **ya imputado** ese día (`totalLogged` minutos) y el objetivo del día
(`target`), calcula las líneas a añadir en este orden. Los topes son sobre el
total del día (ya imputado + lo que vas a añadir), no solo sobre lo nuevo.

1. **Daily (30 min) en el ticket de Reuniones.** Si ese día NO existe ya un
   worklog en el ticket de Reuniones con descripción que contenga "Daily",
   añade 30 min con descripción `Daily`. Suma esos 30 min al efectivo.
2. `gap = target − efectivo`. Si `gap ≤ 0`, no añadas nada más.
3. **Reuniones Varias** (ticket de Reuniones, descripción `Reuniones Varias`),
   con **tope de 120 min/día** en ese concepto: añade
   `min(gap, 120 − "Reuniones Varias" ya imputado)`. Resta del gap.
4. **Soportes Varios** (ticket de Soporte, descripción `Soportes Varios`), con
   **tope de 180 min/día** en ese concepto: añade
   `min(gap, 180 − "Soportes Varios" ya imputado)`. Resta del gap.
5. **Relleno / Desarrollo** (proyecto grande, `TFEX-5` por defecto), descripción
   `Desarrollo`: añade todo el `gap` restante.

Las descripciones deben quedar literalmente: `Daily`, `Reuniones Varias`,
`Soportes Varios`, `Desarrollo`.

## Flujo: imputar un día

1. Determina la fecha (por defecto hoy, Madrid). Si es fin de semana o no
   laborable, avisa y termina.
2. Resuelve los tickets de Reuniones y Soporte del mes (ver
   *Tickets del mes* en Configuración: memoria → búsqueda por título →
   preguntar). No sigas sin tener ambas claves.
3. **Lee lo ya imputado ese día** en TODOS los issues del usuario:
   - `atlassianUserInfo` → `accountId` del usuario actual.
   - `searchJiraIssuesUsingJql` con
     `jql = 'worklogAuthor = currentUser() AND worklogDate = "<fecha>"'`,
     `fields=["summary"]` → claves de issues con worklog ese día.
   - Para cada issue, `getJiraIssue` (cloudId, issueIdOrKey, `fields=["worklog"]`)
     y filtra `fields.worklog.worklogs` cuyo `author.accountId` coincida Y cuyo
     `started` (primeros 10 caracteres, `YYYY-MM-DD`) sea exactamente la fecha.
   - **Eficiencia:** si un issue tiene mucho histórico (`fields.worklog.total`
     alto, p. ej. el proyecto de relleno), no traigas todo desde el principio;
     pide las entradas más recientes (mayor `maxResults`, o `startAt` cercano al
     final), ya que la fecha buscada suele estar entre las últimas.
   - Suma `timeSpentSeconds/60` por entrada. Anota la descripción (`comment`) de
     cada una, en texto plano.
4. Calcula el reparto con la lógica de arriba.
5. **Muestra al usuario** una tabla con: lo ya imputado (issue · descripción ·
   minutos), el objetivo del día, y la propuesta a imputar (ticket · descripción
   · tiempo). Convierte minutos a formato JIRA (p. ej. 510 → `8h 30m`).
6. **Pide confirmación** antes de crear worklogs (crear worklogs es una acción
   con efecto externo). Si ya está cubierto el objetivo, dilo y no imputes.
7. Al confirmar, crea un worklog por línea con `addWorklogToJiraIssue`
   (cloudId, issueIdOrKey, timeSpent, comment literal). El `started` debe ser
   la fecha indicada; usa las 09:00 hora de Madrid si la herramienta requiere
   hora. Reporta el resultado de cada uno (imputado / error).

## Flujo: comprobar / cerrar la semana

1. A partir de una fecha de referencia (por defecto hoy), calcula el **lunes**
   de esa semana y los 5 días laborables (lunes–viernes).
2. Excluye fines de semana y días no laborables.
3. Lee lo ya imputado en el rango con una sola búsqueda:
   `jql = 'worklogAuthor = currentUser() AND worklogDate >= "<lunes>" AND worklogDate <= "<viernes>"'`,
   luego `getJiraIssue` por issue con `fields=["worklog"]` y agrupa por fecha
   (mismo filtro por `accountId` y por fecha exacta que en el flujo diario;
   misma nota de eficiencia para issues con mucho histórico).
4. Para cada día laborable calcula objetivo, ya imputado y pendiente con la
   lógica de reparto.
5. Muestra una tabla semanal (día · fecha · ya imputado · objetivo · pendiente ·
   estado) y el total pendiente.
6. Pide confirmación y, al confirmar, imputa todos los días pendientes con
   `addWorklogToJiraIssue`, cada worklog con su fecha (`started`) correcta.
   Reporta resultados.

## Notas

- Formato JIRA de tiempo: horas y minutos, p. ej. `8h`, `8h 30m`, `30m`.
- Nunca imputes sin confirmación del usuario, ni dupliques worklogs ya
  existentes: la lógica de reparto ya descuenta lo imputado, pero verifica que
  no re-añades un `Daily` si ya existe.
- Si un `getJiraIssue` falla o un issue no devuelve worklogs, trátalo como 0 min
  para ese issue y continúa.
