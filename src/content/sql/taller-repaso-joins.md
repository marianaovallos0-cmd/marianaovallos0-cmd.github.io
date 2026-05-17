---
title: Workshop 1 — Joins Review / Taller 1 — Repaso Joins
description: Views, joins, sequences and anonymous blocks using the HR schema.
author: Mariana Ovallos Romero
---

## Contexto

Hoja de Trabajo #1 entregada en clase. El objetivo era repasar el uso de JOINs, vistas, secuencias y bloques anónimos sobre el schema HR de Oracle. Docente: Christian Felipe Duarte Arévalo.

---

## Punto 1 — Vista vw_sal_emp

Empleados con salario entre 1800 y 3000 dólares.

```sql
CREATE VIEW vw_sal_emp AS
SELECT E.FIRST_NAME,
       E.HIRE_DATE,
       E.SALARY,
       E.PHONE_NUMBER,
       E.EMAIL
FROM HR.EMPLOYEES E
WHERE E.SALARY BETWEEN 1800 AND 3000;
```

---

## Punto 2 — Vista vw_emp_dept con UNION

Empleados de los departamentos 10, 80 y 90.

```sql
CREATE VIEW vw_emp_dept AS
SELECT E.EMPLOYEE_ID, E.FIRST_NAME, D.DEPARTMENT_NAME
FROM HR.EMPLOYEES E
JOIN HR.DEPARTMENTS D ON E.DEPARTMENT_ID = D.DEPARTMENT_ID
WHERE E.DEPARTMENT_ID = 80
UNION
SELECT E.EMPLOYEE_ID, E.FIRST_NAME, D.DEPARTMENT_NAME
FROM HR.EMPLOYEES E
JOIN HR.DEPARTMENTS D ON E.DEPARTMENT_ID = D.DEPARTMENT_ID
WHERE E.DEPARTMENT_ID = 90
UNION
SELECT E.EMPLOYEE_ID, E.FIRST_NAME, D.DEPARTMENT_NAME
FROM HR.EMPLOYEES E
JOIN HR.DEPARTMENTS D ON E.DEPARTMENT_ID = D.DEPARTMENT_ID
WHERE E.DEPARTMENT_ID = 10;
```

---

## Punto 3 — Vista vw_emp_lower

Empleado con el salario más bajo por departamento.

```sql
CREATE VIEW vw_emp_lower AS
SELECT E.FIRST_NAME, D.DEPARTMENT_NAME, E.SALARY
FROM HR.EMPLOYEES E
JOIN HR.DEPARTMENTS D ON E.DEPARTMENT_ID = D.DEPARTMENT_ID
WHERE E.SALARY IN (
  SELECT MIN(E2.SALARY)
  FROM HR.EMPLOYEES E2
  WHERE E2.DEPARTMENT_ID = E.DEPARTMENT_ID
);
```

---

## Punto 4 — Vista vw_emp_salary

Empleado con el salario más alto de la compañía.

```sql
CREATE VIEW vw_emp_salary AS
SELECT E.FIRST_NAME, J.JOB_TITLE
FROM HR.EMPLOYEES E
JOIN HR.JOBS J ON E.JOB_ID = J.JOB_ID
WHERE E.SALARY = (SELECT MAX(SALARY) FROM HR.EMPLOYEES);
```

---

## Punto 5 — Vista vw_emp_manager

Nombre del empleado, su manager y cargo actual de todos los empleados.

```sql
CREATE VIEW vw_emp_manager AS
SELECT E.FIRST_NAME || ' ' || E.LAST_NAME AS NOMBRE_EMPLEADO,
       M.FIRST_NAME || ' ' || M.LAST_NAME AS NOMBRE_MANAGER,
       J.JOB_TITLE AS CARGO
FROM HR.EMPLOYEES E
LEFT JOIN HR.EMPLOYEES M ON E.MANAGER_ID = M.EMPLOYEE_ID
JOIN HR.JOBS J ON E.JOB_ID = J.JOB_ID;
```

---

## Punto 6 — Vista vw_emp_deptos con FULL OUTER JOIN

```sql
CREATE VIEW vw_emp_deptos AS
SELECT E.FIRST_NAME,
       E.LAST_NAME,
       D.DEPARTMENT_NAME,
       E.SALARY
FROM HR.EMPLOYEES E
FULL OUTER JOIN HR.DEPARTMENTS D ON E.DEPARTMENT_ID = D.DEPARTMENT_ID;
```

---

## Punto 7 — Vista vw_emp_dep_cross (producto cartesiano)

```sql
CREATE VIEW vw_emp_dep_cross AS
SELECT E.*, J.*
FROM HR.EMPLOYEES E
CROSS JOIN HR.JOBS J;
```

---

## Punto 8 — Tabla LOG con secuencia SEQ_LOG

```sql
CREATE SEQUENCE SEQ_LOG
  INCREMENT BY 1
  START WITH 1;

CREATE TABLE LOG (
  ID_LOG    NUMBER PRIMARY KEY,
  DETALLE   VARCHAR2(200),
  FECHA_LOG DATE DEFAULT SYSDATE
);
```

---

## Punto 9 — Bloque anónimo: países por continente

```sql
DECLARE
  CURSOR c_continentes IS
    SELECT R.REGION_NAME, COUNT(C.COUNTRY_ID) AS TOTAL
    FROM HR.REGIONS R
    JOIN HR.COUNTRIES C ON R.REGION_ID = C.REGION_ID
    GROUP BY R.REGION_NAME;
BEGIN
  FOR reg IN c_continentes LOOP
    DBMS_OUTPUT.PUT_LINE('El Continente "' || reg.REGION_NAME || 
                         '" tiene "' || reg.TOTAL || '" países.');
  END LOOP;
END;
```

---

## Aprendizaje

Este taller consolidó el uso de vistas como capa de abstracción sobre consultas complejas, la diferencia entre UNION (elimina duplicados) y UNION ALL, el uso de FULL OUTER JOIN para mostrar registros sin coincidencia en ambos lados, y cómo un cursor en un bloque anónimo permite recorrer resultados agrupados e imprimirlos dinámicamente.