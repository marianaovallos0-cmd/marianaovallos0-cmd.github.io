---
title: Relational Database / Base de Datos Relacional
description: PostgreSQL schema with 12 entities, functions, triggers, stored procedures and views.
author: Mariana Ovallos Romero
---

## Contexto

Base de datos diseñada en PostgreSQL para gestionar la plataforma de paseadores de perros. El modelo cubre el ciclo completo del servicio: desde el registro de usuarios hasta la calificación mutua al finalizar un paseo.

---

## Entidades del Sistema

| Entidad | Descripción |
|---------|-------------|
| `rol` | Catálogo de roles: ADMIN, USUARIO, PASEADOR |
| `usuario` | Entidad central con reputación automática |
| `usuario_rol` | Relación N:M entre usuario y rol |
| `paseador` | Extiende usuario (1:1), agrega disponibilidad |
| `dueno` | Extiende usuario (1:1), vinculado a dirección |
| `direccion` | Dirección con coordenadas GPS |
| `perro` | Mascota del dueño con validaciones de edad y peso |
| `disponibilidad` | Franjas horarias del paseador por día |
| `solicitud` | Pedido de paseo con estados controlados |
| `paseo` | Servicio confirmado con precio, distancia y ruta |
| `paseo_perro` | N:M entre paseo y perro (máximo 5) |
| `calificacion` | Evaluación mutua solo al finalizar el paseo |

---

## Creación de Tablas

```sql
CREATE TABLE usuario (
    id_usuario BIGSERIAL PRIMARY KEY,
    correo VARCHAR(100) UNIQUE NOT NULL,
    contrasena TEXT NOT NULL,
    telefono VARCHAR(20),
    primer_nombre VARCHAR(30) NOT NULL,
    segundo_nombre VARCHAR(30),
    primer_apellido VARCHAR(30) NOT NULL,
    segundo_apellido VARCHAR(30),
    foto_perfil TEXT,
    reputacion NUMERIC(3,2) DEFAULT 0,
    activo BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE paseo (
    id_paseo BIGSERIAL PRIMARY KEY,
    estado estado_paseo DEFAULT 'SOLICITADO',
    fecha_inicio TIMESTAMP,
    fecha_fin TIMESTAMP,
    precio NUMERIC(10,2) CHECK (precio > 0),
    distancia_km NUMERIC(6,2),
    ruta TEXT,
    id_solicitud BIGINT UNIQUE NOT NULL,
    id_paseador BIGINT NOT NULL,
    FOREIGN KEY(id_solicitud) REFERENCES solicitud(id_solicitud),
    FOREIGN KEY(id_paseador) REFERENCES paseador(id_usuario)
);

CREATE TABLE calificacion (
    id_calificacion BIGSERIAL PRIMARY KEY,
    puntaje puntuacion NOT NULL,
    comentario TEXT,
    fecha TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    id_paseo BIGINT NOT NULL,
    id_emisor BIGINT NOT NULL,
    id_receptor BIGINT NOT NULL,
    FOREIGN KEY(id_paseo) REFERENCES paseo(id_paseo) ON DELETE CASCADE,
    FOREIGN KEY(id_emisor) REFERENCES usuario(id_usuario),
    FOREIGN KEY(id_receptor) REFERENCES usuario(id_usuario)
);
```

---

## Dominios Personalizados

En lugar de tablas de catálogo separadas, se usaron dominios PostgreSQL para garantizar valores controlados:

```sql
CREATE DOMAIN estado_paseo AS VARCHAR(20)
    CHECK (VALUE IN ('SOLICITADO','ACEPTADO','EN_CURSO','FINALIZADO','CANCELADO'));

CREATE DOMAIN estado_solicitud AS VARCHAR(20)
    CHECK (VALUE IN ('PENDIENTE','ACEPTADA','RECHAZADA','CANCELADA'));

CREATE DOMAIN puntuacion AS NUMERIC(2,1)
    CHECK (VALUE >= 0 AND VALUE <= 5);

CREATE DOMAIN dia_semana AS VARCHAR(10)
    CHECK (VALUE IN ('LUNES','MARTES','MIERCOLES','JUEVES','VIERNES','SABADO','DOMINGO'));
```

---

## Funciones

### fn_registrar_usuario — Registro con roles

```sql
CREATE OR REPLACE FUNCTION fn_registrar_usuario(
    p_correo VARCHAR, p_contrasena VARCHAR, p_telefono VARCHAR,
    p_primer_nombre VARCHAR, p_segundo_nombre VARCHAR,
    p_primer_apellido VARCHAR, p_segundo_apellido VARCHAR,
    p_foto_perfil VARCHAR, p_roles BIGINT[]
)
RETURNS SETOF usuario LANGUAGE plpgsql AS
$$
DECLARE
    v_id_usuario BIGINT;
    v_rol_id BIGINT;
BEGIN
    IF EXISTS(SELECT 1 FROM usuario WHERE correo = p_correo) THEN
        RAISE EXCEPTION 'El correo ya está registrado';
    END IF;

    INSERT INTO usuario(correo, contrasena, telefono, primer_nombre,
        segundo_nombre, primer_apellido, segundo_apellido, foto_perfil,
        reputacion, activo, created_at, updated_at)
    VALUES(p_correo, p_contrasena, p_telefono, p_primer_nombre,
        p_segundo_nombre, p_primer_apellido, p_segundo_apellido,
        p_foto_perfil, 0, true, NOW(), NOW())
    RETURNING id_usuario INTO v_id_usuario;

    IF p_roles IS NOT NULL THEN
        FOREACH v_rol_id IN ARRAY p_roles LOOP
            INSERT INTO usuario_rol(id_usuario, id_rol)
            VALUES(v_id_usuario, v_rol_id);
        END LOOP;
    END IF;

    RETURN QUERY SELECT * FROM usuario WHERE id_usuario = v_id_usuario;
END;
$$;
```

### fn_validar_max_perros — Límite de 5 perros por paseo

```sql
CREATE OR REPLACE FUNCTION fn_validar_max_perros()
RETURNS TRIGGER LANGUAGE plpgsql AS
$$
DECLARE
    cantidad INTEGER;
BEGIN
    SELECT COUNT(*) INTO cantidad
    FROM paseo_perro WHERE id_paseo = NEW.id_paseo;

    IF cantidad >= 5 THEN
        RAISE EXCEPTION 'Máximo 5 perros por paseo';
    END IF;

    RETURN NEW;
END;
$$;
```

---

## Procedimientos Almacenados

### sp_aceptar_solicitud — Acepta solicitud y crea el paseo

```sql
CREATE OR REPLACE PROCEDURE sp_aceptar_solicitud(p_solicitud BIGINT)
LANGUAGE plpgsql AS
$$
DECLARE
    v_paseador BIGINT;
BEGIN
    UPDATE solicitud SET estado = 'ACEPTADA'
    WHERE id_solicitud = p_solicitud;

    SELECT id_paseador INTO v_paseador
    FROM solicitud WHERE id_solicitud = p_solicitud;

    INSERT INTO paseo(estado, id_solicitud, id_paseador)
    VALUES('ACEPTADO', p_solicitud, v_paseador);
END;
$$;
```

---

## Triggers

### trg_validar_paseo_activo — Impide dos paseos simultáneos

```sql
CREATE TRIGGER trg_validar_paseo_activo
BEFORE INSERT OR UPDATE ON paseo
FOR EACH ROW
WHEN (NEW.estado = 'EN_CURSO')
EXECUTE FUNCTION fn_validar_paseo_activo();
```

### trg_actualizar_reputacion — Recalcula reputación tras calificación

```sql
CREATE TRIGGER trg_actualizar_reputacion
AFTER INSERT ON calificacion
FOR EACH ROW
EXECUTE FUNCTION fn_actualizar_reputacion();
```

**Por qué triggers:** la reputación y las restricciones de negocio se aplican de forma transparente en la capa de datos, sin depender de la aplicación. Así aunque se inserte directamente en la base, las reglas siempre se cumplen.

---

## Vistas

### vw_historial_paseos

```sql
CREATE OR REPLACE VIEW vw_historial_paseos AS
SELECT
    p.id_paseo, p.estado, p.fecha_inicio, p.fecha_fin, p.precio,
    u_dueno.primer_nombre || ' ' || u_dueno.primer_apellido AS dueno,
    u_paseador.primer_nombre || ' ' || u_paseador.primer_apellido AS paseador
FROM paseo p
INNER JOIN solicitud s ON s.id_solicitud = p.id_solicitud
INNER JOIN dueno d ON d.id_usuario = s.id_dueno
INNER JOIN usuario u_dueno ON u_dueno.id_usuario = d.id_usuario
INNER JOIN paseador pa ON pa.id_usuario = p.id_paseador
INNER JOIN usuario u_paseador ON u_paseador.id_usuario = pa.id_usuario;
```

### mv_ranking_paseadores — Vista materializada

```sql
CREATE MATERIALIZED VIEW mv_ranking_paseadores AS
SELECT
    p.id_usuario,
    u.primer_nombre,
    ROUND(AVG(c.puntaje), 2) AS reputacion
FROM paseador p
INNER JOIN usuario u ON u.id_usuario = p.id_usuario
LEFT JOIN calificacion c ON c.id_receptor = p.id_usuario
GROUP BY p.id_usuario, u.primer_nombre;
```

La vista materializada permite consultar el ranking de paseadores con alto rendimiento sin recalcular el promedio en cada consulta.

---

## Aprendizaje

Este proyecto integró todos los conceptos del curso en un sistema real. Lo más valioso fue entender cómo los triggers y funciones en la capa de base de datos garantizan las reglas de negocio independientemente de la aplicación que las consuma. La vista materializada fue una decisión de diseño consciente: el ranking de paseadores se consulta frecuentemente pero se actualiza poco, haciéndola ideal para materialización.