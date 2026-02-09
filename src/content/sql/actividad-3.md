---
title: Actividad 3
description: Employee and manager relationship.
---

```sql
/*
Autor: Mariana Ovallos Romero
Fecha: 6/02/2026
Descripción: Traer todos los nombres de los empleados y los nombres de sus managers
*/

SELECT
E.FIRST_NAME || ' ' || E.LAST_NAME AS NOMBRE_EMPLEADO,
M.FIRST_NAME || ' ' || M.LAST_NAME AS NOMBRE_MANAGER
FROM HR.EMPLOYEES E LEFT JOIN HR.EMPLOYEES M
ON E.MANAGER_ID = M.EMPLOYEE_ID;