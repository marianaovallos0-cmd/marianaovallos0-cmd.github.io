---
title: Activity 1 — Sequences / Actividad 1 — Secuencias
description: Sequence creation with error handling and ALTER correction.
author: _____
---

## Contexto

En este ejercicio se creó una secuencia llamada `SEQ_numeroPlanta` para generar valores automáticos para la tabla `plantas`. El objetivo era insertar 500 registros usando un bloque `FOR LOOP`. Sin embargo, al ejecutarlo se generó un error porque el `MAXVALUE` de la secuencia era insuficiente.

---

## Creación de la Secuencia y la Tabla

```sql
CREATE SEQUENCE SEQ_numeroPlanta
  INCREMENT BY 100
  START WITH 100
  MAXVALUE 2000;

CREATE TABLE plantas (
  num NUMBER,
  uso VARCHAR(50)
);
```

La secuencia inicia en 100 e incrementa de 100 en 100. Con `MAXVALUE 2000`, solo puede generar **20 valores** antes de agotarse.

---

## Bloque de Inserción

```sql
BEGIN
  FOR indice IN 1..500 LOOP
    INSERT INTO plantas(num, uso)
    VALUES(SEQ_numeroPlanta.NEXTVAL, 'Suites');
  END LOOP;
END;
```

El loop intenta insertar 500 filas. En la iteración número 21, la secuencia supera su `MAXVALUE` y Oracle lanza un error.

---

## Error Encontrado
ORA-08004: secuencia SEQ_NUMEROPLANTA.NEXTVAL exceeds MAXVALUE y no se puede instanciar


**Causa:** Con `INCREMENT BY 100` y `MAXVALUE 2000`, la secuencia se agota después de 20 llamadas. El loop necesita 500, lo que supera ese límite ampliamente.

---

## Solución: ALTER SEQUENCE

```sql
ALTER SEQUENCE SEQ_numeroPlanta
  MAXVALUE 5000
  CYCLE;
```

Con `MAXVALUE 5000` la secuencia puede generar hasta 50 valores incrementales. La opción `CYCLE` permite que al llegar al tope, la secuencia reinicie automáticamente desde el valor inicial, evitando el error en ejecuciones largas.

---

## Aprendizaje

Antes de crear una secuencia para un loop, es fundamental calcular cuántas veces se llamará `NEXTVAL` y definir un `MAXVALUE` que lo soporte, o usar `CYCLE` y `NOCACHE` según el caso de uso.