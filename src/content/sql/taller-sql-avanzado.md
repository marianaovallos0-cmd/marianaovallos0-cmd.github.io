---
title: Workshop 2 — Advanced SQL + ACID / Taller 2 — SQL Avanzado + ACID
description: Salary adjustment transaction with CTEs, MERGE, SAVEPOINT and complete audit trail.
author: Mariana Ovallos Romero, Jhonatan Lara
---

## Contexto

Taller aplicado en clase sobre SQL avanzado y propiedades ACID. El caso simula un ajuste salarial extraordinario en una empresa, donde no todos los empleados son elegibles. La solución debía seguir 5 etapas: diagnóstico, decisión, prevalidación, ejecución transaccional y validación posterior. Docente: Christian Felipe Duarte Arévalo.

La variante asignada determinaba los parámetros dinámicos (departamento excluido, antigüedad mínima, umbrales de brecha salarial, porcentajes de aumento) leídos desde la tabla `T1_VARIANTS`.

---

## Etapa 1 — Consulta Diagnóstica

Consolida el estado actual de cada empleado: salario, promedio departamental, ranking salarial e historial reciente de cargos. Usa CTEs y funciones analíticas para no modificar datos.

```sql
WITH variant_data AS (
    SELECT * FROM t1_variants WHERE variant_id = &p_variant_id
),
dept_stats AS (
    SELECT department_id,
           ROUND(AVG(salary),2) AS dept_avg_salary,
           MAX(salary) AS dept_max_salary,
           COUNT(employee_id) AS dept_employee_count
    FROM t1_employees
    GROUP BY department_id
),
recent_hist AS (
    SELECT DISTINCT e.employee_id
    FROM t1_employees e
    JOIN t1_job_history jh ON e.employee_id = jh.employee_id
    CROSS JOIN variant_data v
    WHERE jh.end_date >= ADD_MONTHS(SYSDATE, -(v.recent_job_history_months))
),
base_data AS (
    SELECT e.employee_id, e.first_name, e.last_name, e.job_id,
           e.department_id, d.department_name, e.salary, e.hire_date,
           TRUNC(MONTHS_BETWEEN(SYSDATE, e.hire_date)/12) AS years_service,
           NVL(ds.dept_avg_salary, 0) AS dept_avg_salary,
           NVL(ds.dept_max_salary, 0) AS dept_max_salary,
           NVL(ds.dept_employee_count, 0) AS dept_employee_count,
           CASE WHEN ds.dept_avg_salary > 0
                THEN ROUND(((ds.dept_avg_salary - e.salary) / ds.dept_avg_salary) * 100, 2)
                ELSE 0 END AS pct_gap_to_avg,
           CASE WHEN rh.employee_id IS NOT NULL THEN 'SI' ELSE 'NO' END AS recent_job_history_flag,
           RANK() OVER(PARTITION BY e.department_id ORDER BY e.salary DESC) AS salary_rank_in_department,
           v.excluded_department_id, v.min_years_service,
           v.gap_high_threshold_pct, v.gap_mid_threshold_pct,
           v.raise_high_pct, v.raise_mid_pct, v.raise_low_pct, v.max_salary_vs_avg_pct
    FROM t1_employees e
    LEFT JOIN t1_departments d ON e.department_id = d.department_id
    LEFT JOIN dept_stats ds ON e.department_id = ds.department_id
    LEFT JOIN recent_hist rh ON e.employee_id = rh.employee_id
    CROSS JOIN variant_data v
)
SELECT employee_id, first_name, last_name, department_id, department_name,
       salary, years_service, dept_avg_salary, dept_max_salary,
       dept_employee_count, pct_gap_to_avg, recent_job_history_flag,
       salary_rank_in_department
FROM base_data
ORDER BY department_id, salary_rank_in_department;
```

**Por qué CTEs:** permiten separar claramente cada capa de cálculo (estadísticas departamentales, historial reciente, datos base) sin repetir lógica ni crear tablas temporales. La función `RANK()` con `PARTITION BY` da el ranking salarial dentro de cada departamento en una sola pasada.

---

## Etapa 2 — Decisión de Elegibilidad

Vista temporal que determina quién recibe ajuste y qué porcentaje aplica según la variante.

```sql
CREATE OR REPLACE VIEW v_temp_decision AS
-- [CTEs idénticos a la etapa 1]
SELECT b.employee_id, b.department_id, b.salary AS salary_before,
       b.dept_avg_salary, b.pct_gap_to_avg,
       CASE
         WHEN b.department_id IS NULL
           OR b.manager_or_exec_flag = 'SI'
           OR b.department_id = b.excluded_department_id
           OR b.years_service < b.min_years_service
           OR b.recent_job_history_flag = 'SI'
           OR b.dept_employee_count < 3
           OR b.salary >= (b.dept_avg_salary * (1 + (b.max_salary_vs_avg_pct / 100)))
         THEN 'NO_ELEGIBLE'
         ELSE 'ELEGIBLE'
       END AS eligibility_flag,
       CASE
         WHEN b.pct_gap_to_avg >= b.gap_high_threshold_pct THEN b.raise_high_pct
         WHEN b.pct_gap_to_avg >= b.gap_mid_threshold_pct  THEN b.raise_mid_pct
         ELSE b.raise_low_pct
       END AS adj_pct,
       CASE
         WHEN b.pct_gap_to_avg >= b.gap_high_threshold_pct THEN 'AJUSTE_ALTO'
         WHEN b.pct_gap_to_avg >= b.gap_mid_threshold_pct  THEN 'AJUSTE_MEDIO'
         ELSE 'AJUSTE_BAJO'
       END AS rule_name
FROM base_data b;
```

**Criterios de exclusión aplicados:** managers/ejecutivos, departamento excluido por variante, antigüedad insuficiente, historial reciente de cambio de cargo, departamentos con menos de 3 empleados, y salario ya por encima del tope máximo permitido.

---

## Etapa 3 — Prevalidación

Calcula el impacto económico antes de ejecutar cualquier cambio.

```sql
-- Resumen económico
SELECT COUNT(employee_id)                              AS total_elegibles,
       SUM(salary_before)                              AS total_salario_antes,
       SUM(salary_before * (1 + (adj_pct / 100)))      AS total_salario_despues,
       SUM((salary_before * (1 + (adj_pct / 100))) - salary_before) AS total_incremento
FROM v_temp_decision
WHERE eligibility_flag = 'ELEGIBLE';

-- Control de topes por departamento
SELECT DISTINCT department_id, dept_avg_salary,
       ROUND((dept_avg_salary * (1 + (max_salary_vs_avg_pct / 100))), 2) AS salario_maximo_permitido
FROM v_temp_decision
WHERE department_id IS NOT NULL
ORDER BY department_id;
```

---

## Etapa 4 — Ejecución Transaccional

```sql
SAVEPOINT sv_before_adjustment;

-- Actualización con MERGE
MERGE INTO t1_employees e
USING (SELECT employee_id, adj_pct FROM v_temp_decision WHERE eligibility_flag = 'ELEGIBLE') d
ON (e.employee_id = d.employee_id)
WHEN MATCHED THEN
    UPDATE SET e.salary = e.salary * (1 + (d.adj_pct / 100));

-- Inserción en auditoría
INSERT INTO audit_salary_adjustments_t1 (
    audit_id, execution_tag, variant_id, employee_id, department_id,
    salary_before, salary_after, pct_gap_to_avg_before, rule_applied,
    executed_by, executed_at, notes
)
SELECT AUDIT_SALARY_ADJ_T1_SEQ.NEXTVAL, '&p_execution_tag', &p_variant_id,
       v.employee_id, v.department_id, v.salary_before, e.salary,
       v.pct_gap_to_avg, v.rule_name, USER, SYSDATE,
       'Actualización transaccional ejecutada'
FROM v_temp_decision v
JOIN t1_employees e ON v.employee_id = e.employee_id
WHERE v.eligibility_flag = 'ELEGIBLE';

COMMIT;
```

---

## Etapa 5 — Validación Posterior

```sql
-- Empleados impactados
SELECT e.employee_id, e.first_name, e.last_name,
       a.salary_before, a.salary_after, a.execution_tag
FROM t1_employees e
JOIN audit_salary_adjustments_t1 a ON e.employee_id = a.employee_id
WHERE a.execution_tag = '&p_execution_tag';

-- Verificación de topes
SELECT a.employee_id, a.salary_after,
       CASE WHEN a.salary_after <= (v.dept_avg_salary * (1 + (v.max_salary_vs_avg_pct / 100)))
            THEN 'CUMPLE' ELSE 'NO_CUMPLE' END AS estado_tope
FROM audit_salary_adjustments_t1 a
JOIN v_temp_decision v ON a.employee_id = v.employee_id
WHERE a.execution_tag = '&p_execution_tag';
```

---

## Propiedades ACID aplicadas

| Propiedad | Implementación |
|-----------|---------------|
| **Atomicidad** | MERGE + INSERT empaquetados como una unidad. Si algo falla, `ROLLBACK TO sv_before_adjustment` revierte ambas operaciones. |
| **Consistencia** | La prevalidación y validación intermedia garantizan que ningún salario supere el tope máximo permitido. |
| **Aislamiento** | El SAVEPOINT y los bloqueos exclusivos de la transacción activa evitan que otras sesiones lean datos sucios durante el proceso. |
| **Durabilidad** | El COMMIT final escribe permanentemente en los archivos de datos y redo logs de Oracle. |

---

## Aprendizaje

Este taller demostró que una transacción correcta no se mide solo por si "corre sin errores", sino por si respeta las reglas de negocio, deja evidencia auditable y puede revertirse de forma segura. El uso de `SAVEPOINT` antes del `MERGE` fue clave para tener un punto de retorno sin afectar transacciones anteriores de la sesión. Los CTEs evitaron repetir lógica compleja y mejoraron la legibilidad del script completo.