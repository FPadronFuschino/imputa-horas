# MEMORY.md — Contrato de memoria de la Skill

La Agent Skill `imputa-horas` usa la **memoria persistente de claude.ai** para no
resolver los tickets del mes en cada sesión. Este fichero documenta qué se
guarda y cómo. (No es la memoria de un runtime propio: depende de que la función
de memoria esté activada en la cuenta de claude.ai; si no lo está, la Skill
resuelve por búsqueda cada sesión y sigue funcionando.)

## Qué se guarda

Por cada mes, **solo el enlace/clave** de los tickets de Reuniones y Soporte
resueltos por título. **Nunca** worklogs ni entradas de los tickets (son
compartidos por decenas de personas y ocupan muchísimo contexto). Formato
sugerido de la entrada:

```
imputa-horas: reuniones julio 2026 = https://taxitronic.atlassian.net/browse/INTERF-34 ; soporte julio 2026 = https://taxitronic.atlassian.net/browse/INTERF-35
```

La clave del ticket es el último segmento del enlace (`INTERF-34`). Indexado por
**`{mes} {año}`** (español, minúscula).

## Ciclo de vida

- Al empezar una imputación, la Skill busca en memoria la entrada del mes actual.
- Si existe → la usa sin volver a consultar JIRA.
- Si no existe (primer uso del mes / mes nuevo) → busca por título en JIRA y
  guarda el resultado.
- **Verificación "cada 1 de mes"**: sale gratis del indexado por mes+año — al
  cambiar el mes no hay entrada y se dispara una búsqueda fresca.
- Si la búsqueda no encuentra el ticket → la Skill pregunta al usuario y guarda
  lo que reciba.

## Qué NO se guarda

- **Worklogs ni entradas de los tickets** — jamás. Solo el enlace/clave.
- Objetivos por día, topes y cascada de reparto: son fijos y viven en `RULES.md`
  / `SKILL.md`, no en memoria.
- Credenciales ni tokens de JIRA: la autenticación la gestiona el MCP de
  Atlassian.
