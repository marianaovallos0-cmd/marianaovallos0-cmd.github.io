---
title: Actividad 2
description: Employees in Europe with salary filter.
---

```sql
/*
Autor: Mariana Ovallos Romero
Fecha: 6/02/2026
Descripción: De los empleados se muestran son nombre empleado, país y salario que vivan o trabajen en Europa y que ganen entre 7000 y 9000 dólares.
*/

SELECT
E.FIRST_NAME || ' ' || E.LAST_NAME AS NOMBRE,
CO.COUNTRY_NAME AS PAIS,
E.SALARY
FROM HR.EMPLOYEES E JOIN HR.DEPARTMENTS D
ON E.DEPARTMENT_ID= D.DEPARTMENT_ID
JOIN HR.LOCATIONS L ON D.LOCATION_ID = L.LOCATION_ID
JOIN HR.COUNTRIES CO ON L.COUNTRY_ID = CO.COUNTRY_ID
WHERE REGION_ID = 10 AND SALARY BETWEEN 7000 AND 9000;
