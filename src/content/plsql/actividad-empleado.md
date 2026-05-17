---
title: Employee Query / Consulta Empleado
description: ROWTYPE variable with salary and commission calculation using NVL.
author: Christian Duarte
---

## Contexto

Se usó una variable de tipo `%ROWTYPE` para recuperar toda la fila de un empleado y calcular su salario real descontando la comisión. Se trabajó con dos versiones: una que genera un error de tipo de datos y la versión corregida con `TO_CHAR`.

---

## Versión Corregida

```sql
/*
  Autor: Christian Duarte
  Fecha: 01/12/2026
  Descripción: Mi primer script
*/
DECLARE
  v_empleado HR.EMPLOYEES%ROWTYPE;
BEGIN
  SELECT * INTO v_empleado
  FROM HR.EMPLOYEES
  WHERE EMPLOYEE_ID = 109;

  DBMS_OUTPUT.PUT_LINE(
    'El salario es: ' || v_empleado.SALARY ||
    ' y el nombre es: ' || v_empleado.FIRST_NAME ||
    ', pero por culpa del Gobierno su salario es ' ||
    TO_CHAR(v_empleado.salary - (v_empleado.salary * NVL(v_empleado.commission_pct, 0)))
  );
END;
```

---

## Versión con Error (para comparación)

```sql
DECLARE
  v_empleado HR.EMPLOYEES%ROWTYPE;
BEGIN
  SELECT * INTO v_empleado
  FROM HR.EMPLOYEES
  WHERE EMPLOYEE_ID = 109;

  DBMS_OUTPUT.PUT_LINE(
    'El salario es: ' || v_empleado.SALARY ||
    ' y el nombre es: ' || v_empleado.FIRST_NAME ||
    ', pero por culpa del Gobierno su salario es ' ||
    v_empleado.salary - (v_empleado.salary * NVL(v_empleado.commission_pct, 0))
  );
END;
```

**Error:** Oracle no puede concatenar (`||`) un `NUMBER` directamente con un `VARCHAR2`. La expresión aritmética al final no está envuelta en `TO_CHAR`, lo que produce un error de tipos en tiempo de ejecución.

---

## Explicación de elementos clave

|         Elemento        |                                         Descripción                                        |
|-------------------------|--------------------------------------------------------------------------------------------|
|        `%ROWTYPE`       | La variable hereda la estructura completa de la tabla, incluyendo todos sus campos.        |
| `NVL(commission_pct, 0)`| Si no tiene comisión, lo trata como `0` para que la multiplicación no devuelva `NULL`.     |
|      `TO_CHAR(...)`     | Convierte el resultado numérico a texto para poder concatenarlo en el `PUT_LINE`.          |
|     `SELECT * INTO`     | Recupera exactamente una fila. Si devuelve cero o más de una, Oracle lanza `NO_DATA_FOUND` 
|                         |                                                                          o `TOO_MANY_ROWS`.|

---

## Aprendizaje

`%ROWTYPE` es una de las herramientas más útiles en PL/SQL para trabajar con filas completas sin declarar cada campo individualmente. El uso de `NVL` es fundamental cuando hay campos opcionales que podrían ser `NULL` y participan en cálculos: cualquier operación aritmética con `NULL` produce `NULL`.