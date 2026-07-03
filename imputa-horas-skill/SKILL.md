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
   ese `{mes} {año}`. Guarda **solo el enlace/clave** de cada ticket, nunca sus
   entradas. Ejemplo de entrada de memoria:
   `imputa-horas: reuniones julio 2026 = https://taxitronic.atlassian.net/browse/INTERF-34 ; soporte julio 2026 = https://taxitronic.atlassian.net/browse/INTERF-35`.
   La clave del ticket es el último segmento del enlace (`INTERF-34`).
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
4. **Guarda solo el enlace/clave** en tu memoria persistente para ese
   `{mes} {año}`, para no repetir la búsqueda en futuras sesiones. No guardes
   worklogs ni entradas del ticket.
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

Calcula primero **`fpi`** ("lo que falta por imputar") = `target − totalLogged`,
donde `totalLogged` es todo lo ya imputado ese día. `fpi` decide cuánto va en
Reuniones Varias. Luego añade las líneas en este orden:

1. **Daily — 30 min (OBLIGATORIO)** en el ticket de Reuniones, descripción
   `Daily`. El ticket de Reuniones lleva siempre al menos estos 30m. Añádelo
   salvo que ese día ya exista un worklog cuya descripción contenga "Daily" (no
   dupliques). Suma esos 30m al efectivo.
2. **Reuniones Varias** (ticket de Reuniones, descripción `Reuniones Varias`),
   importe fijo según `fpi`:
   - `fpi ≥ 5h` (≥300 min) → **+1h30 (90 min)** (Reuniones total 2h).
   - `3h ≤ fpi < 5h` (180–299 min) → **+1h (60 min)** (Reuniones total 1h30).
   - `fpi < 3h` (<180 min) → **nada** (Reuniones se queda en el Daily de 30m).
   Añade `min(gap, objetivo_del_tramo − "Reuniones Varias" ya imputado)`. Resta
   del gap.
3. **Soportes Varios** (ticket de Soporte, descripción `Soportes Varios`), con
   **tope de 180 min/día**: añade `min(gap, 180 − "Soportes Varios" ya imputado)`.
   Resta del gap.
4. **Relleno / Desarrollo** (proyecto grande, `TFEX-5` por defecto), descripción
   `Desarrollo`: añade todo el `gap` restante.

Donde `gap = target − efectivo` en cada paso (efectivo = ya imputado + lo añadido
hasta ese punto). Si `gap ≤ 0`, no añadas más (pero el Daily del paso 1 es
obligatorio de todas formas).

Las descripciones deben quedar literalmente: `Daily`, `Reuniones Varias`,
`Soportes Varios`, `Desarrollo`.

## Flujo: imputar un día

1. Determina la fecha (por defecto hoy, Madrid). Si es fin de semana o no
   laborable, avisa y termina.
2. Resuelve los tickets de Reuniones y Soporte del mes (ver
   *Tickets del mes* en Configuración: memoria → búsqueda por título →
   preguntar). No sigas sin tener ambas claves.
3. **Lee SOLO tus propias entradas de ese día.** Los tickets de Reuniones y
   Soporte son compartidos: tienen cientos de worklogs de decenas de personas.
   **NUNCA cargues el histórico completo de un ticket ni las entradas de otros
   compañeros** — es un desperdicio enorme de contexto. Solo necesitas los
   worklogs del usuario autenticado en la fecha exacta.
   - `atlassianUserInfo` → `accountId` del usuario actual.
   - `searchJiraIssuesUsingJql` con
     `jql = 'worklogAuthor = currentUser() AND worklogDate = "<fecha>"'`,
     `fields=["summary"]`. Devuelve SOLO los issues donde **tú** imputaste ese
     día (claves + summary, barato). **Si un ticket no aparece en este
     resultado, no has imputado nada en él ese día → cuéntalo como 0 y NO leas
     sus worklogs.**
   - Solo para los issues que aparezcan, lee tus entradas de ESE día con **una
     sola** llamada `getJiraIssue` (`fields=["worklog"]`). Comportamiento REAL
     de este MCP (comprobado): la respuesta trae `fields.worklog.total` y como
     mucho ~20 worklogs, que son los **más antiguos**; **no pagina de forma
     fiable hasta el final ni filtra por fecha** (aunque pases
     `startAt`/`startedAfter`, puede ignorarlos).
     - Si `fields.worklog.total` ≤ ~20, o la página devuelta contiene la fecha
       buscada → filtra por tu `accountId` + fecha (`started`, `YYYY-MM-DD`) y
       úsalo. (Caso típico de tus tickets ALFA6, con pocas entradas.)
     - Si `total` es alto y la página (entradas viejas) NO contiene la fecha →
       **NO insistas con más páginas** (no funciona y malgasta contexto). Es un
       ticket compartido de mucho volumen (INTERF de Reuniones/Soporte, cientos
       de worklogs): tu entrada del día **no es legible por MCP**. Ve al
       fallback.
   - **Fallback para ticket compartido ilegible** (apareció en el JQL del día
     pero no puedes leer tu entrada):
     - Sabes que imputaste *algo* ese día en ese ticket. Muestra lo que sí es
       legible y **pregunta al usuario cuánto tiene ese día en ese ticket** y en
       qué concepto (p. ej. Daily 30m / Reuniones Varias Xm). **No asumas 0**:
       infravalorar lo imputado haría que sobre-imputes el resto en TFEX-5 y el
       día pase del objetivo.
     - Con su respuesta, aplica la cascada normal (no dupliques el Daily si ya
       existe).
   - Suma `timeSpentSeconds/60` de tus entradas legibles + lo confirmado por el
     usuario. Anota su `comment`.
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
   `jql = 'worklogAuthor = currentUser() AND worklogDate >= "<lunes>" AND worklogDate <= "<viernes>"'`.
   Devuelve SOLO los issues donde tú imputaste en la semana; los que no
   aparezcan cuentan como 0 y no se leen. Para los que aparezcan, aplica la
   **misma lectura acotada del flujo diario** (filtro por fecha si existe, o
   página pequeña desde el final, tope duro ~100 worklogs, y preguntar si no se
   aíslan tus entradas). **Nunca leas el histórico completo ni las entradas de
   otros.** Agrupa tus entradas por fecha.
4. Para cada día laborable calcula objetivo, ya imputado y pendiente con la
   lógica de reparto.
5. Muestra una tabla semanal (día · fecha · ya imputado · objetivo · pendiente ·
   estado) y el total pendiente.
6. Pide confirmación y, al confirmar, imputa todos los días pendientes con
   `addWorklogToJiraIssue`, cada worklog con su fecha (`started`) correcta.
   Reporta resultados.

## Notas

- **Regla de oro de contexto:** los tickets de Reuniones y Soporte son
  compartidos por decenas de personas. No leas su histórico completo ni las
  entradas de otros: solo tus propios worklogs de la fecha concreta, de forma
  acotada. En memoria guarda únicamente el enlace/clave del ticket.
- **Limitación conocida del MCP:** `getJiraIssue` devuelve solo la primera
  página del worklog (~20 entradas, las más antiguas) y no filtra por fecha ni
  pagina fiable. En tickets con cientos de worklogs (Reuniones/Soporte) no se
  puede leer tu entrada del día → se resuelve por confirmación del usuario
  (fallback). Los tickets propios ALFA6, con pocas entradas, sí se leen bien.
- **Día "en fresco" (nada imputado aún):** el JQL autor+fecha no devuelve nada,
  así que no se lee ningún ticket compartido ni se pregunta nada — se aplica la
  cascada completa directamente. El fallback (preguntar) solo aparece al
  verificar/completar un día ya imputado parcialmente en el ticket compartido.
- Formato JIRA de tiempo: horas y minutos, p. ej. `8h`, `8h 30m`, `30m`.
- Nunca imputes sin confirmación del usuario, ni dupliques worklogs ya
  existentes: la lógica de reparto ya descuenta lo imputado, pero verifica que
  no re-añades un `Daily` si ya existe.
- Si un `getJiraIssue` falla o un issue no devuelve worklogs, trátalo como 0 min
  para ese issue y continúa.
