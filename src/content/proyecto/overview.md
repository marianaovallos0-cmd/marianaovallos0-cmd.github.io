---
title: Overview / Visión General
description: Dog walking platform connecting owners and walkers in real time.
author: Mariana Ovallos Romero
---

## El Proyecto

**Delta** es una plataforma digital que conecta dueños de mascotas con paseadores disponibles, permitiendo solicitar, gestionar y hacer seguimiento de paseos en tiempo real.

---

## Problema que resuelve

Actualmente los dueños de perros enfrentan procesos informales y sin garantías para encontrar paseadores confiables. No existe seguimiento en tiempo real, ni control de calidad del servicio, ni mecanismos de confianza entre usuarios desconocidos.

---

## Solución

Un sistema web completo con:

- Registro de usuarios con roles (dueño, paseador, admin)
- Gestión de solicitudes con estados controlados
- Seguimiento GPS del paseo en tiempo real
- Sistema de calificaciones mutuas post-paseo
- Reputación calculada automáticamente por la base de datos

---

## Stack Tecnológico

| Capa | Tecnología |
|------|-----------|
| Backend | Spring Boot (Java) |
| Base de datos relacional | PostgreSQL |
| Base de datos NoSQL | MongoDB |
| Frontend | React |
| Virtualización | Linux/Debian |
| Control de versiones | Git / GitHub |

---

## Repositorios

- **Backend:** [github.com/DnzoCoud/paseadores-back](https://github.com/DnzoCoud/paseadores-back)
- **Frontend:** [github.com/marianaovallos0-cmd/plataforma_paseadores_front](https://github.com/marianaovallos0-cmd/plataforma_paseadores_front)

## Participantes

- [Daniel Rodriguez](https://github.com/DnzoCoud)
- [Jhonatan Lara](https://github.com/jlaragUEB)
- [Mariana Ovallos](https://github.com/marianaovallos0-cmd)


---

## Estado actual

El modelo relacional está completamente implementado en PostgreSQL con 12 entidades, dominios personalizados, índices, funciones, triggers y vistas. El backend expone endpoints REST con autenticación JWT. El frontend y la integración con MongoDB están en desarrollo.