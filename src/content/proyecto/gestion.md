---
title: Project Management / Gestión del Proyecto
description: Scrum methodology, Gantt chart, RACI matrix and risk management.
author: Mariana Ovallos Romero
---

## Metodología

El proyecto se gestionó bajo **Scrum adaptado** para equipos académicos. Se definieron sprints semanales con entregables claros por fase, reuniones de revisión al inicio de cada clase y un tablero de tareas compartido.

---

## Equipo y Roles

| Integrante | Rol principal |
|-----------|--------------|
| Mariana Ovallos | Base de datos, documentación, Fronted React |
| Jhonatan Lara | Base de datos, integración MV |
| Daniel Rodriguez | Backend Spring Boot, NoSQL / MongoDB |

---

## Fases del Proyecto

| Fase | Entregable | Estado |
|------|-----------|--------|
| Fase 1 | Propuesta, modelo E-R, normalización, gestión | Completada |
| Fase 2 | Análisis SO/RDBMS, implementación BD relacional | Completada |
| Fase 3 | MongoDB, integración, aplicación web | Completada  |

---

## Gestión de Riesgos

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|-------------|---------|-----------|
| Incompatibilidad entre Spring Boot y PostgreSQL | Media | Alto | Uso de Spring Data JPA con drivers certificados |
| Pérdida de avance por falta de control de versiones | Alta | Alto | GitHub con ramas por feature |
| Inconsistencia de datos entre BD relacional y MongoDB | Media | Medio | Definición clara de qué datos van en cada motor |
| Falta de tiempo para implementar frontend | Alta | Medio | Priorizar backend y BD; frontend como demostración básica |
| Errores en lógica de triggers | Media | Alto | Pruebas unitarias por cada trigger antes de integrar |

---

## Control de Versiones

El proyecto usa GitHub con ramas separadas por módulo. Los commits reflejan el avance real de cada integrante, con mensajes descriptivos que siguen la convención `feat:`, `fix:`, `docs:`.

---

## Aprendizaje

La gestión formal del proyecto demostró que la planificación previa ahorra tiempo de desarrollo. La matriz RACI evitó ambigüedad sobre quién era responsable de cada entregable, y el cronograma Gantt permitió detectar con anticipación los riesgos de tiempo en la integración.