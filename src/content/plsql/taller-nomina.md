---
title: Workshop — Payroll System / Taller — Sistema de Nómina
description: Full PL/SQL payroll liquidation for HotelGroup S.A. with packages, triggers and BULK COLLECT.
author: Mariana Ovallos Romero, Jhonatan Lara
---

## Contexto

Taller avanzado de PL/SQL desarrollado en clase en 90 minutos. El caso modelaba la liquidación quincenal de nómina para HotelGroup S.A., una cadena hotelera con 4 sedes en Colombia (Bogotá, Medellín, Santa Marta y Cartagena). Docente: Christian Felipe Duarte Arévalo.

El taller evaluó 8 puntos que cubrían toda la capa PL/SQL: bloques anónimos, funciones encadenadas, procedimientos con excepciones, packages con sobrecarga, compound triggers, BULK COLLECT + FORALL, pipelined functions y transacciones autónomas.

---

## Tipos de contrato y reglas base

| Tipo | Salario base quincenal | Recargos | Aux. transporte |
|------|----------------------|----------|-----------------|
| PLANTA | `salario_base / 2` | Sí | Si base mensual ≤ 2×SMLMV |
| TEMPORAL | `valor_hora × horas_normales` | Sí | Si base mensual equiv. ≤ 2×SMLMV |
| SERVICIOS | `(salario_base − retención 11%) / 2` | No | No |

---

## Punto 1 — Bloque Anónimo: Liquidación Individual

Calcula y muestra la liquidación quincenal de un empleado usando cursor explícito y variables `%TYPE`.

```sql
DECLARE
  v_empleado      EMPLEADOS%ROWTYPE;
  v_smlmv         PARAMETROS.valor%TYPE;
  v_aux_transp    PARAMETROS.valor%TYPE;
  v_ret_serv      PARAMETROS.valor%TYPE;
  v_rec_noct      PARAMETROS.valor%TYPE;
  v_rec_dom       PARAMETROS.valor%TYPE;
  v_rec_noct_dom  PARAMETROS.valor%TYPE;

  vn_salario_base_q  NUMBER := 0;
  vn_recargos        NUMBER := 0;
  vn_bonificacion    NUMBER := 0;
  vn_valor_hora      NUMBER := 0;
  vn_antiguedad      NUMBER := 0;
  vn_sanciones       NUMBER := 0;

  CURSOR c_horas (p_id NUMBER, p_quincena VARCHAR2) IS
    SELECT tipo_hora, cantidad_horas
    FROM HORAS_TRABAJADAS
    WHERE id_empleado = p_id AND id_quincena = p_quincena;

BEGIN
  -- Leer parámetros desde tabla (sin hardcodear)
  SELECT valor INTO v_smlmv        FROM PARAMETROS WHERE cod_parametro = 'SMLMV';
  SELECT valor INTO v_aux_transp   FROM PARAMETROS WHERE cod_parametro = 'AUX_TRANSPORTE';
  SELECT valor INTO v_ret_serv     FROM PARAMETROS WHERE cod_parametro = 'RET_SERVICIOS';
  SELECT valor INTO v_rec_noct     FROM PARAMETROS WHERE cod_parametro = 'RECARGO_NOCTURNO';
  SELECT valor INTO v_rec_dom      FROM PARAMETROS WHERE cod_parametro = 'RECARGO_DOMINICAL';
  SELECT valor INTO v_rec_noct_dom FROM PARAMETROS WHERE cod_parametro = 'RECARGO_NOCT_DOM';

  SELECT * INTO v_empleado FROM EMPLEADOS WHERE id_empleado = 1001;

  -- Calcular salario base quincenal según tipo de contrato
  IF v_empleado.tipo_contrato = 'PLANTA' THEN
    vn_salario_base_q := v_empleado.salario_base / 2;
    vn_valor_hora     := v_empleado.salario_base / 240;
  ELSIF v_empleado.tipo_contrato = 'TEMPORAL' THEN
    SELECT NVL(SUM(cantidad_horas), 0) INTO vn_salario_base_q
    FROM HORAS_TRABAJADAS
    WHERE id_empleado = 1001 AND id_quincena = '2026-Q1-ENE' AND tipo_hora = 'NORMAL';
    vn_salario_base_q := v_empleado.salario_base * vn_salario_base_q;
    vn_valor_hora     := v_empleado.salario_base;
  ELSIF v_empleado.tipo_contrato = 'SERVICIOS' THEN
    vn_salario_base_q := (v_empleado.salario_base - (v_empleado.salario_base * v_ret_serv / 100)) / 2;
  END IF;

  -- Recargos (solo PLANTA y TEMPORAL)
  IF v_empleado.tipo_contrato != 'SERVICIOS' THEN
    FOR h IN c_horas(1001, '2026-Q1-ENE') LOOP
      IF h.tipo_hora = 'NOCTURNA' THEN
        vn_recargos := vn_recargos + (h.cantidad_horas * vn_valor_hora * v_rec_noct / 100);
      ELSIF h.tipo_hora = 'DOMINICAL' THEN
        vn_recargos := vn_recargos + (h.cantidad_horas * vn_valor_hora * v_rec_dom / 100);
      ELSIF h.tipo_hora = 'NOCTURNA_DOM' THEN
        vn_recargos := vn_recargos + (h.cantidad_horas * vn_valor_hora * v_rec_noct_dom / 100);
      END IF;
    END LOOP;
  END IF;

  -- Bonificación por antigüedad (Regla 3)
  vn_antiguedad := TRUNC(MONTHS_BETWEEN(SYSDATE, v_empleado.fecha_ingreso) / 12);
  SELECT COUNT(*) INTO vn_sanciones
  FROM SANCIONES
  WHERE id_empleado = 1001 AND fecha_sancion >= ADD_MONTHS(SYSDATE, -6);

  IF vn_sanciones <= 2 AND v_empleado.tipo_contrato != 'SERVICIOS' THEN
    IF    vn_antiguedad BETWEEN 3 AND 5  THEN vn_bonificacion := vn_salario_base_q * 0.03;
    ELSIF vn_antiguedad BETWEEN 6 AND 10 THEN vn_bonificacion := vn_salario_base_q * 0.06;
    ELSIF vn_antiguedad > 10             THEN vn_bonificacion := vn_salario_base_q * 0.10;
    END IF;
  END IF;

  -- Salida formateada
  DBMS_OUTPUT.PUT_LINE('=== LIQUIDACIÓN QUINCENAL ===');
  DBMS_OUTPUT.PUT_LINE('Empleado: ' || v_empleado.nombre || ' (' || v_empleado.id_empleado || ')');
  DBMS_OUTPUT.PUT_LINE('Tipo contrato: ' || v_empleado.tipo_contrato);
  DBMS_OUTPUT.PUT_LINE('Antigüedad: ' || vn_antiguedad || ' años');
  DBMS_OUTPUT.PUT_LINE('-----------------------------');
  DBMS_OUTPUT.PUT_LINE('Salario base Q:  ' || TO_CHAR(vn_salario_base_q, 'FM999,999,999.00'));
  DBMS_OUTPUT.PUT_LINE('Recargos:        ' || TO_CHAR(vn_recargos, 'FM999,999,999.00'));
  DBMS_OUTPUT.PUT_LINE('Bonificación:    ' || TO_CHAR(vn_bonificacion, 'FM999,999,999.00'));
  DBMS_OUTPUT.PUT_LINE('-----------------------------');
  DBMS_OUTPUT.PUT_LINE('SUBTOTAL: ' || TO_CHAR(vn_salario_base_q + vn_recargos + vn_bonificacion, 'FM999,999,999.00'));
  DBMS_OUTPUT.PUT_LINE('=============================');
END;
```

---

## Punto 2 — Funciones Encadenadas

Cuatro funciones que leen parámetros desde `PARAMETROS` y se encadenan para calcular el bruto.

```sql
-- Función 1: Salario base quincenal
CREATE OR REPLACE FUNCTION fn_salario_base_q(
  p_id_empleado NUMBER, p_id_quincena VARCHAR2
) RETURN NUMBER IS
  v_tipo    EMPLEADOS.tipo_contrato%TYPE;
  v_base    EMPLEADOS.salario_base%TYPE;
  v_ret     PARAMETROS.valor%TYPE;
  v_horas   NUMBER := 0;
  v_result  NUMBER := 0;
BEGIN
  SELECT tipo_contrato, salario_base INTO v_tipo, v_base
  FROM EMPLEADOS WHERE id_empleado = p_id_empleado;

  IF v_tipo = 'PLANTA' THEN
    v_result := v_base / 2;
  ELSIF v_tipo = 'TEMPORAL' THEN
    SELECT NVL(SUM(cantidad_horas), 0) INTO v_horas
    FROM HORAS_TRABAJADAS
    WHERE id_empleado = p_id_empleado AND id_quincena = p_id_quincena AND tipo_hora = 'NORMAL';
    v_result := v_base * v_horas;
  ELSIF v_tipo = 'SERVICIOS' THEN
    SELECT valor INTO v_ret FROM PARAMETROS WHERE cod_parametro = 'RET_SERVICIOS';
    v_result := (v_base - (v_base * v_ret / 100)) / 2;
  END IF;
  RETURN v_result;
END;
/

-- Función 4: Bruto total (llama a las 3 anteriores)
CREATE OR REPLACE FUNCTION fn_bruto(
  p_id_empleado NUMBER, p_id_quincena VARCHAR2
) RETURN NUMBER IS
  v_base_q    NUMBER;
  v_recargos  NUMBER;
  v_bonif     NUMBER;
  v_aux       NUMBER := 0;
  v_bono_sede NUMBER := 0;
  v_tipo      EMPLEADOS.tipo_contrato%TYPE;
  v_sede      EMPLEADOS.cod_sede%TYPE;
  v_smlmv     PARAMETROS.valor%TYPE;
  v_aux_val   PARAMETROS.valor%TYPE;
  v_bono_sma  PARAMETROS.valor%TYPE;
BEGIN
  v_base_q   := fn_salario_base_q(p_id_empleado, p_id_quincena);
  v_recargos := fn_recargos(p_id_empleado, p_id_quincena);
  v_bonif    := fn_bonificacion(p_id_empleado);

  SELECT tipo_contrato, cod_sede INTO v_tipo, v_sede
  FROM EMPLEADOS WHERE id_empleado = p_id_empleado;

  -- Auxilio de transporte (Regla 4)
  IF v_tipo != 'SERVICIOS' THEN
    SELECT valor INTO v_smlmv   FROM PARAMETROS WHERE cod_parametro = 'SMLMV';
    SELECT valor INTO v_aux_val FROM PARAMETROS WHERE cod_parametro = 'AUX_TRANSPORTE';
    IF (v_base_q * 2) <= (2 * v_smlmv) THEN
      v_aux := v_aux_val / 2;
    END IF;
  END IF;

  -- Bono sede Santa Marta (Regla 5)
  IF v_sede = 'SMA' AND v_tipo != 'SERVICIOS' THEN
    SELECT valor INTO v_bono_sma FROM PARAMETROS WHERE cod_parametro = 'BONO_CLIMA_SMA';
    v_bono_sede := v_bono_sma;
  END IF;

  RETURN v_base_q + v_recargos + v_bonif + v_aux + v_bono_sede;
END;
/
```

---

## Punto 3 — Procedimiento con Excepciones

```sql
CREATE OR REPLACE PROCEDURE sp_liquidar_empleado(
  p_id_empleado NUMBER, p_id_quincena VARCHAR2
) IS
  v_empleado  EMPLEADOS%ROWTYPE;
  v_estado    EMPLEADOS.estado%TYPE;
  v_count     NUMBER;
  vn_bruto    NUMBER;
  vn_salud    NUMBER;
  vn_pension  NUMBER;
  vn_embargo  NUMBER := 0;
  vn_libranza NUMBER := 0;
  vn_neto     NUMBER;
  v_pct_sal   PARAMETROS.valor%TYPE;
  v_pct_pen   PARAMETROS.valor%TYPE;
BEGIN
  -- Validación 1: empleado existe
  SELECT * INTO v_empleado FROM EMPLEADOS WHERE id_empleado = p_id_empleado;

  -- Validación 2: empleado activo
  IF v_empleado.estado != 'ACTIVO' THEN
    RAISE_APPLICATION_ERROR(-20002, 'Empleado no activo: estado = ' || v_empleado.estado);
  END IF;

  -- Validación 3: no liquidado ya
  SELECT COUNT(*) INTO v_count FROM LIQUIDACION
  WHERE id_empleado = p_id_empleado AND id_quincena = p_id_quincena;
  IF v_count > 0 THEN
    RAISE_APPLICATION_ERROR(-20003, 'Liquidación ya existe para empleado '
      || p_id_empleado || ' quincena ' || p_id_quincena);
  END IF;

  -- Cálculo
  vn_bruto  := fn_bruto(p_id_empleado, p_id_quincena);
  SELECT valor INTO v_pct_sal FROM PARAMETROS WHERE cod_parametro = 'PCT_SALUD';
  SELECT valor INTO v_pct_pen FROM PARAMETROS WHERE cod_parametro = 'PCT_PENSION';
  vn_salud   := vn_bruto * v_pct_sal / 100;
  vn_pension := vn_bruto * v_pct_pen / 100;

  SELECT NVL(SUM(porcentaje), 0) INTO vn_embargo
  FROM EMBARGOS WHERE id_empleado = p_id_empleado AND estado = 'ACTIVO';
  vn_embargo := (vn_bruto - vn_salud - vn_pension) * vn_embargo / 100;

  SELECT NVL(SUM(cuota_mensual), 0) / 2 INTO vn_libranza
  FROM LIBRANZAS WHERE id_empleado = p_id_empleado AND estado = 'ACTIVA';

  vn_neto := vn_bruto - vn_salud - vn_pension - vn_embargo - vn_libranza;

  -- Caso especial: neto negativo
  IF vn_neto < 0 THEN
    vn_embargo := 0;
    vn_neto    := vn_bruto - vn_salud - vn_pension - vn_libranza;
    IF vn_neto < 0 THEN
      vn_libranza := 0;
      vn_neto     := vn_bruto - vn_salud - vn_pension;
    END IF;
  END IF;

  INSERT INTO LIQUIDACION (id_liquidacion, id_empleado, id_quincena,
    salario_base_q, bruto, deduccion_salud, deduccion_pension,
    embargo, libranzas, neto, fecha_liquidacion)
  VALUES (SEQ_LIQUIDACION.NEXTVAL, p_id_empleado, p_id_quincena,
    fn_salario_base_q(p_id_empleado, p_id_quincena),
    vn_bruto, vn_salud, vn_pension, vn_embargo, vn_libranza, vn_neto, SYSDATE);

  COMMIT;

EXCEPTION
  WHEN NO_DATA_FOUND THEN
    RAISE_APPLICATION_ERROR(-20001, 'Empleado no encontrado: ' || p_id_empleado);
END;
/
```

---

## Punto 5 — Compound Trigger

Actúa en INSERT sobre `LIQUIDACION`: valida neto negativo antes de insertar, actualiza libranzas y genera log del lote.

```sql
CREATE OR REPLACE TRIGGER trg_liquidacion_control
FOR INSERT ON LIQUIDACION
COMPOUND TRIGGER

  vb_ajustado BOOLEAN := FALSE;

  BEFORE EACH ROW IS
  BEGIN
    IF :NEW.salario_base_q < 0 THEN
      RAISE_APPLICATION_ERROR(-20010, 'Salario base no puede ser negativo');
    END IF;
    IF :NEW.neto < 0 THEN
      :NEW.embargo := 0;
      :NEW.neto    := :NEW.bruto - :NEW.deduccion_salud - :NEW.deduccion_pension - :NEW.libranzas;
      IF :NEW.neto < 0 THEN
        :NEW.libranzas := 0;
        :NEW.neto      := :NEW.bruto - :NEW.deduccion_salud - :NEW.deduccion_pension;
      END IF;
      :NEW.total_deducciones := :NEW.deduccion_salud + :NEW.deduccion_pension
                               + :NEW.embargo + :NEW.libranzas;
      vb_ajustado := TRUE;
    END IF;
  END BEFORE EACH ROW;

  AFTER EACH ROW IS
  BEGIN
    IF vb_ajustado THEN
      INSERT INTO LOG_NOMINA (id_log, fecha_log, operacion, detalle, id_empleado)
      VALUES (SEQ_LOG.NEXTVAL, SYSTIMESTAMP, 'ALERTA_NETO_NEGATIVO',
              'Neto negativo ajustado para empleado ' || :NEW.id_empleado, :NEW.id_empleado);
    END IF;
    UPDATE LIBRANZAS
    SET saldo_pendiente = saldo_pendiente - :NEW.libranzas
    WHERE id_empleado = :NEW.id_empleado AND estado = 'ACTIVA';
    UPDATE LIBRANZAS SET estado = 'PAGADA'
    WHERE id_empleado = :NEW.id_empleado AND saldo_pendiente <= 0;
  END AFTER EACH ROW;

  AFTER STATEMENT IS
  BEGIN
    INSERT INTO LOG_NOMINA (id_log, fecha_log, operacion, detalle)
    VALUES (SEQ_LOG.NEXTVAL, SYSTIMESTAMP, 'INSERT_LIQUIDACION',
            'Lote procesado a las ' || TO_CHAR(SYSTIMESTAMP, 'HH24:MI:SS.FF3'));
  END AFTER STATEMENT;

END trg_liquidacion_control;
/
```

---

## Punto 8 — Auditoría con AUTONOMOUS_TRANSACTION

```sql
PROCEDURE sp_log_nomina(
  p_operacion       VARCHAR2,
  p_detalle         VARCHAR2,
  p_empleados_ok    NUMBER DEFAULT 0,
  p_empleados_error NUMBER DEFAULT 0,
  p_monto_total     NUMBER DEFAULT 0
) IS
  PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
  INSERT INTO LOG_NOMINA (id_log, fecha_log, operacion, detalle,
    empleados_ok, empleados_error, monto_total, usuario)
  VALUES (SEQ_LOG.NEXTVAL, SYSTIMESTAMP, p_operacion, p_detalle,
    p_empleados_ok, p_empleados_error, p_monto_total, USER);
  COMMIT;
END sp_log_nomina;
```

**Por qué `AUTONOMOUS_TRANSACTION`:** si la transacción principal hace `ROLLBACK` por un error en la liquidación masiva, los registros del log se perderían. Con esta pragma, el log vive en su propia transacción independiente y siempre queda evidencia de lo que se intentó hacer.

---

## Validación de Resultados Clave

| Empleado | Tipo | Bruto esperado | Neto esperado |
|----------|------|---------------|---------------|
| 1001 — Carlos Méndez | PLANTA, Bogotá | $1,423,958.33 | $1,115,041.67 |
| 1003 — Pedro Suárez | TEMPORAL, Bogotá | $1,695,625.00 | — |
| 1019 — Héctor Díaz | SERVICIOS, Bogotá | $2,892,500.00 | — |
| 1016 — Sandra Mejía | TEMPORAL, 0 horas | $0.00 | $0.00 |

---

## Aprendizaje

Este taller integró todos los conceptos del curso en un solo sistema coherente. Lo más desafiante fue el compound trigger, ya que requiere coordinar lógica `BEFORE` y `AFTER` sin crear problemas de mutating table. El `PRAGMA AUTONOMOUS_TRANSACTION` fue revelador: entender que el log debe sobrevivir a un `ROLLBACK` de la transacción principal cambia completamente cómo se piensa la auditoría. El `BULK COLLECT + FORALL` con `SAVE EXCEPTIONS` permitió procesar todos los empleados activos en una sola operación masiva, tolerando fallos individuales sin detener el lote completo.