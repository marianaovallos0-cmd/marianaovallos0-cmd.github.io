---
title: Quiz — Indexes / Quiz — Índices
description: Composite index analysis over PAY_TIME_ENTRIES.
author: _____
---

## Contexto

Se creó un índice compuesto sobre la tabla `PAY_TIME_ENTRIES` con dos columnas: primero `EMPLOYEE_ID` y luego `WORK_DATE`. La pregunta del quiz era determinar qué consultas se benefician realmente de ese índice y por qué.

---

## Creación del Índice

```sql
CREATE INDEX IX_TIME_EMP_DATE
ON PAY_TIME_ENTRIES (EMPLOYEE_ID, WORK_DATE);
```

Un índice compuesto funciona de izquierda a derecha: la **columna líder** (`EMPLOYEE_ID`) debe estar presente en la condición para que el índice sea aprovechado eficientemente.

---

## Consultas Evaluadas

**Consulta A** — Usa ambas columnas: filtro de igualdad sobre la líder + rango sobre la segunda.
```sql
SELECT * FROM PAY_TIME_ENTRIES
WHERE EMPLOYEE_ID = 100
  AND WORK_DATE BETWEEN DATE '2026-03-01' AND DATE '2026-03-31';
```

**Consulta B** — Solo usa `WORK_DATE` (no es la columna líder).
```sql
SELECT * FROM PAY_TIME_ENTRIES
WHERE WORK_DATE = DATE '2026-03-15';
```

**Consulta C** — Solo usa `EMPLOYEE_ID` (la columna líder).
```sql
SELECT * FROM PAY_TIME_ENTRIES
WHERE EMPLOYEE_ID = 100;
```

**Consulta D** — Usa ambas columnas, pero con una desigualdad (`>`) sobre la líder.
```sql
SELECT * FROM PAY_TIME_ENTRIES
WHERE WORK_DATE BETWEEN DATE '2026-03-01' AND DATE '2026-03-31'
  AND EMPLOYEE_ID > 100;
```

---

## Respuesta Correcta: **B**

> *El índice beneficia claramente a A y C, puede ayudar en D en ciertos escenarios, pero no es la mejor opción para B porque `WORK_DATE` no es la columna líder.*

---

## Análisis

| Consulta | ¿Usa índice? | Razón |
|----------|-------------|-------|
| A | Sí, eficientemente | Igualdad en líder + rango en segunda: uso óptimo del índice. |
| B | No eficientemente | `WORK_DATE` no es la columna líder; Oracle haría full scan. |
| C | Sí | Solo la columna líder es suficiente para activar el índice. |
| D | Parcialmente | La desigualdad (`>`) en la líder limita el uso; depende del optimizador. |

---

## Aprendizaje

Al diseñar índices compuestos, la columna que se filtra con **igualdad más frecuente** debe ir primero. Los filtros de rango o desigualdad en la columna líder reducen la efectividad del índice. Si la consulta más frecuente es sobre `WORK_DATE`, sería mejor un índice separado para esa columna.