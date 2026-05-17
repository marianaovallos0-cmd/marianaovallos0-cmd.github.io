---
title: Triggers / Triggers
description: Salary audit trigger for employees in department 80.
author: _____
---

## Contexto

Se creó un trigger que registra automáticamente en una tabla de auditoría cada vez que se actualiza el salario de un empleado perteneciente al **departamento 80** (ventas). Primero se creó la copia de la tabla de empleados sobre la que opera el trigger.

---

## Creación de la Tabla Base

```sql
CREATE TABLE copia_employees AS
SELECT * FROM samesa.EMPLOYEES;
```

Se trabaja sobre una copia para no modificar la tabla original de HR.

---

## Trigger de Auditoría

```sql
CREATE TRIGGER "TR_SALARIO_80"
  AFTER UPDATE
  ON copia_employees
  FOR EACH ROW
  WHEN (OLD.department_id = 80)
DECLARE

BEGIN
  INSERT INTO auditoria_empleados (
    employee_id,
    salario_anterior,
    salario_nuevo,
    fecha_cambio,
    usuario_modifico
  ) VALUES (
    :OLD.employee_id,
    :OLD.salary,
    :NEW.salary,
    SYSDATE,
    USER
  );

EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Error en el trigger TR_SALARIO_80: ' || SQLERRM);
END "TR_SALARIO_80";
/
```

---

## Explicación

|             Elemento            |                               Descripción                                 |
|---------------------------------|---------------------------------------------------------------------------|
|          `AFTER UPDATE`         | Se ejecuta después de que el UPDATE ocurre, no antes.                     |
|          `FOR EACH ROW`         | Se dispara por cada fila afectada, no una sola vez por sentencia.         |
| `WHEN (OLD.department_id = 80)` | Filtra: solo audita empleados del departamento 80.                        |
|          `:OLD.salary`          | Valor del salario antes del cambio.                                       |
|          `:NEW.salary`          | Valor del salario después del cambio.                                     |
|          `USER`                 | Función de Oracle que retorna el usuario de sesión que ejecutó el cambio. |
|          `SYSDATE`              | Fecha y hora exacta del momento del cambio.                               |
|   `EXCEPTION WHEN OTHERS`       | Captura cualquier error inesperado sin detener la ejecución.              |

---

## Aprendizaje

Los triggers son útiles para auditoría porque actúan de forma transparente: el usuario que hace el `UPDATE` no necesita saber que existe el trigger. La cláusula `WHEN` permite filtrar sin meter lógica dentro del cuerpo del trigger, lo que mejora el rendimiento.