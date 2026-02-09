---
title: Actividad 1
description: Employee job count analysis.
---

```sql
/*
Autor: Mariana Ovallos Romero
Fecha: 6/02/2026
Descripción: mostrar los datos del empleado siendo estos nombre y la cantidad de cargos que ha tenido en la compañía.
*/

SELECT E.FIRST_NAME,
       E.EMPLOYEE_ID,
       COUNT (J.JOB_ID)+1
FROM HR.EMPLOYEES E
JOIN HR.JOB_HISTORY J
ON E.EMPLOYEE_ID = J.EMPLOYEE_ID
GROUP BY E.FIRST_NAME,
         E.EMPLOYEE_ID
HAVING COUNT (J.JOB_ID)+1 >= 2;
