---
title: Fibonacci / Fibonacci
description: Three PL/SQL implementations of the Fibonacci sequence.
author: _____
---

## Contexto

Se implementó la serie de Fibonacci de tres formas distintas en PL/SQL para practicar el uso de diferentes estructuras de control: un bloque anónimo con `LOOP`, un procedimiento con `WHILE` y un procedimiento con `LOOP` y parámetros de rango.

---

## Implementación 1 — Bloque Anónimo con LOOP

Imprime los primeros 100 elementos de la serie.

```sql
/*
  Descripción: Imprime la serie de Fibonacci hasta el elemento 100
*/
DECLARE
  a   NUMBER := 0;
  b   NUMBER := 1;
  c   NUMBER;
  num NUMBER := 0;
BEGIN
  DBMS_OUTPUT.PUT_LINE('Serie Fibonacci:');

  LOOP
    num := num + 1;
    DBMS_OUTPUT.PUT_LINE(a);
    c := a + b;
    a := b;
    b := c;
    IF (num = 100) THEN
      EXIT;
    END IF;
  END LOOP;
END;
```

**Lógica:** Las variables `a` y `b` guardan los dos valores anteriores. En cada iteración se imprime `a`, se calcula el siguiente (`c = a + b`) y se desplazan los valores. La condición `EXIT` detiene el loop al llegar a 100 iteraciones.

---

## Implementación 2 — Procedimiento con LOOP (por rango de posición)

Imprime los elementos de Fibonacci entre la posición `p_init` y `p_end`.

```sql
CREATE OR REPLACE PROCEDURE SP_FIBONACCI(p_init NUMBER, p_end NUMBER)
IS
  vn_a        NUMBER := 0;
  vn_b        NUMBER := 1;
  vn_position NUMBER := 1;
  vn_temp     NUMBER;
BEGIN
  IF p_init <= 0 OR p_end <= 0 OR p_init > p_end THEN
    DBMS_OUTPUT.PUT_LINE('Invalid range');
  END IF;

  LOOP
    IF vn_position BETWEEN p_init AND p_end THEN
      DBMS_OUTPUT.PUT_LINE('Fibo(' || vn_position || ') = ' || vn_b);
    END IF;
    vn_temp     := vn_a + vn_b;
    vn_a        := vn_b;
    vn_b        := vn_temp;
    vn_position := vn_position + 1;
    EXIT WHEN vn_position > p_end;
  END LOOP;
END;

-- Ejecución de prueba
BEGIN
  SP_FIBONACCI(3, 100);
END;
```

---

## Implementación 3 — Procedimiento con WHILE (por rango de posición)

Misma lógica de rango pero usando `WHILE` en lugar de `LOOP/EXIT`.

```sql
CREATE OR REPLACE PROCEDURE SP_FIBONACCI_WHILE(p_init NUMBER, p_end NUMBER)
IS
  vn_a        NUMBER := 0;
  vn_b        NUMBER := 1;
  vn_position NUMBER := 1;
  vn_temp     NUMBER;
BEGIN
  IF p_init <= 0 OR p_end <= 0 OR p_init > p_end THEN
    DBMS_OUTPUT.PUT_LINE('Invalid range');
  END IF;

  WHILE (vn_position <= p_end) LOOP
    IF vn_position BETWEEN p_init AND p_end THEN
      DBMS_OUTPUT.PUT_LINE('Fibo(' || vn_position || ') = ' || vn_b);
    END IF;
    vn_temp     := vn_a + vn_b;
    vn_a        := vn_b;
    vn_b        := vn_temp;
    vn_position := vn_position + 1;
  END LOOP;
END;
```

---

## Comparación de Estructuras

| Implementación     |    Estructura      |    Condición de salida  |
|--------------------|--------------------|-------------------------|
| Bloque anónimo     | `LOOP / EXIT WHEN` | Contador llega a 100    |
| SP_FIBONACCI       | `LOOP / EXIT WHEN` | Posición supera `p_end` |
| SP_FIBONACCI_WHILE | `WHILE ... LOOP`   | Posición supera `p_end` |

---

## Aprendizaje

Las tres implementaciones producen el mismo resultado, pero con diferente control de flujo. El `WHILE` evalúa la condición **antes** de cada iteración; el `LOOP/EXIT` la evalúa **dentro**, lo que permite mayor flexibilidad. Para rangos con validación de entrada, los procedimientos son preferibles a los bloques anónimos porque son reutilizables.