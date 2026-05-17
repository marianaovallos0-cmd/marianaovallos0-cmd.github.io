---
title: Workshop — MQL MongoDB / Taller — MQL MongoDB
description: 30 exercises on CRUD operations, aggregation pipelines and advanced operators over sample_airbnb.
author: Mariana Ovallos Romero, Daniel Rodriguez
---

## Contexto

Taller de MQL (MongoDB Query Language) desarrollado sobre la base de datos `sample_airbnb`. El objetivo fue consolidar operaciones CRUD, consultas con filtros y proyecciones, y pipelines de agregación. Docente: Christian Felipe Duarte Arévalo.

---

## Operaciones de Inserción

**Ejercicio 1 — insertOne básico**

```javascript
use("sample_airbnb");
db.listingsAndReviews.insertOne({
  name: "Littel Cabin retreat",
  summary: "Pequeña cabaña acogedora y agradable",
  property_type: "Cabin",
  price: NumberDecimal('150.00')
})
```

**Ejercicio 2 — insertOne con objeto anidado**

```javascript
use("sample_airbnb");
db.listingsAndReviews.insertOne({
  name: "Departamento en el centro",
  property_type: "Apartment",
  price: NumberDecimal('95.00'),
  address: {
    country: "Colombia",
    market: "Bogotá"
  },
  amenities: ["Wifi", "Kitchen"]
})
```

Para acceder a campos anidados en consultas se usa notación de punto entre comillas: `"address.country"`.

**Ejercicio 3 — insertMany**

```javascript
use("sample_airbnb");
db.listingsAndReviews.insertMany([
  { name: "Studio calle 76", price: NumberDecimal("80.00") },
  { name: "Casa minimalista", price: NumberDecimal("70.00") }
])
```

**Ejercicio 5 — Validación de unicidad del \_id**

```javascript
db.listingsAndReviews.insertOne({
  _id: ObjectId("6a02854ce5ff92ec173a1791"),
  name: "Caso duplicado"
})
// Error: E11000 duplicate key error
```

MongoDB obliga a que el `_id` sea único en cada colección. Intentar insertar un documento con un `_id` ya existente genera el error `E11000 duplicate key error`.

---

## Consultas con find

**Ejercicio 6 — Filtro por país con proyección**

```javascript
db.listingsAndReviews.find(
  { "address.country": "Brazil" },
  { name: 1, price: 1, _id: 0 }
)
```

**Ejercicio 7 — Mayor que ($gt)**

```javascript
db.listingsAndReviews.find(
  { beds: { $gt: 5 } },
  { name: 1, number_of_reviews: 1, _id: 0 }
)
```

**Ejercicio 9 — $and con objeto anidado**

```javascript
db.listingsAndReviews.find(
  { $and: [
    { property_type: "Apartment" },
    { "address.market": "New York" }
  ]},
  { property_type: 1, "address.market": 1 }
)
```

**Ejercicio 10 — $in**

```javascript
db.listingsAndReviews.find(
  { property_type: { $in: ["House", "Condominium"] } },
  { property_type: 1 }
)
```

**Ejercicio 11 — Regex sin distinción de mayúsculas**

```javascript
db.listingsAndReviews.find(
  { name: { $regex: "^Luxury", $options: "i" } },
  { name: 1 }
)
```

**Ejercicio 13 — $size exacto en array**

```javascript
db.listingsAndReviews.find(
  { amenities: { $size: 20 } },
  { name: 1, amenities: 1 }
)
```

`$size` no se puede combinar con operadores de comparación; solo acepta igualdad exacta.

**Ejercicio 14 — sort + limit**

```javascript
db.listingsAndReviews.find(
  {},
  { name: 1, price: 1, "address.country": 1, _id: 0 }
).sort({ price: -1 }).limit(10)
```

---

## Operaciones de Actualización

**Ejercicio 17 — $set + $push simultáneo**

```javascript
db.listingsAndReviews.updateOne(
  { name: "Copacabana Apartment Posto 6" },
  {
    $set: { property_type: "Casa gigante" },
    $push: { amenities: "Smart Tv" }
  }
)
```

**Ejercicio 18 — $inc en updateMany**

```javascript
db.listingsAndReviews.updateMany(
  { "address.country": "Spain" },
  { $inc: { number_of_reviews: 1 } }
)
// matchedCount: 633, modifiedCount: 633
```

**Ejercicio 19 — $pull en updateMany**

```javascript
db.listingsAndReviews.updateMany(
  { amenities: "Cable TV" },
  { $pull: { amenities: "Cable TV" } }
)
// matchedCount: 1735, modifiedCount: 1735
```

**Ejercicio 20 — upsert: true**

```javascript
db.listingsAndReviews.updateOne(
  { name: "Apartamentico nuevo" },
  { $set: { price: NumberDecimal("1500.00"), "address.country": "United States" } },
  { upsert: true }
)
// upsertedCount: 1 — no existía, se creó automáticamente
```

---

## Pipelines de Agregación

**Ejercicio 22 — Precio promedio por país**

```javascript
db.listingsAndReviews.aggregate([
  { $group: { _id: "$address.country", avg_precio: { $avg: "$price" } } },
  { $sort: { avg_precio: -1 } }
])
```

Hong Kong lidera con el promedio más alto, seguido por Brasil y China.

**Ejercicio 23 — Top 5 amenities más comunes**

```javascript
db.listingsAndReviews.aggregate([
  { $unwind: "$amenities" },
  { $group: { _id: "$amenities", frecuency: { $sum: 1 } } },
  { $sort: { frecuency: -1 } },
  { $limit: 5 }
])
// Resultado: Wifi (5304), Essentials (5048), Kitchen (4952), TV (4295), Hangers (4226)
```

`$unwind` descompone el array `amenities` en documentos individuales; sin él no se podría contar por elemento.

**Ejercicio 27 — $bucket para rangos de precio**

```javascript
db.listingsAndReviews.aggregate([
  { $bucket: {
    groupBy: "$price",
    boundaries: [0, 101, 301, 1001],
    default: "1000+",
    output: { cantidad: { $sum: 1 } }
  }}
])
```

`$bucket` requiere límites exactos y que todos los documentos caigan dentro de algún rango o del `default`. El tipado `Decimal128` del campo `price` puede causar problemas con operadores numéricos estándar.

**Ejercicio 30 — Análisis por año de última review**

```javascript
db.listingsAndReviews.aggregate([
  { $match: { last_review: { $exists: true, $ne: null } } },
  { $group: { _id: { año: { $year: "$last_review" } }, total: { $sum: 1 } } },
  { $sort: { "_id.año": 1 } }
])
```

---

## Dificultades encontradas

| Situación | Solución |
|-----------|----------|
| Campos ausentes en algunos documentos | Usar `$exists` para filtrar antes de operar |
| `$size` no acepta operadores de comparación | Solo funciona con igualdad exacta |
| `price` usa `Decimal128` en vez de Number | Precaución en operaciones aritméticas y conversiones con `$toInt` |
| `$unwind` duplica documentos | Esperado: es su función; se agrupa después |
| `$bucket` requiere límites precisos | Definir `default` para valores fuera de rango |
| Diferencia entre `find()` y `aggregate()` | Para pipelines múltiples, siempre `aggregate()` |

---

## Aprendizaje

La diferencia fundamental entre SQL y MongoDB no es solo de sintaxis sino de modelo mental: en MongoDB los datos pueden estar anidados dentro del mismo documento, lo que elimina JOINs pero requiere conocer operadores como `$unwind` para trabajar con arrays. El pipeline de agregación es secuencial — cada etapa recibe la salida de la anterior — lo que obliga a pensar en el orden correcto de `$match` (primero, para reducir documentos) antes de `$group` o `$unwind`. El uso de `upsert: true` fue especialmente útil para operaciones donde no se sabe si el documento ya existe.