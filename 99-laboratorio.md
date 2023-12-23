# Laboratorio MongoDB

**Pregunta**. Si montaras un sitio real, ¿Qué posibles problemas pontenciales les ves a como está almacenada la información?

```md
El documento es demasiado grande. Por ejemplo, el objeto de reviews ocupa aproximadamente de la línea 151 a la 662 en el primer registro. Aquí cabría aplicar el subset pattern para optimizar el espacio del Working Set. Además, están las review scores, que deberían ir con un computed pattern para no malgastar ciclos de CPU. También podría venir bien el patrón de schema versioning al cambiar, quizá, frecuentemente el precio.
```

## Obligatorio

Esta es la parte mínima que tendrás que entregar para superar este laboratorio.

### Consultas

- Saca en una consulta cuantos alojamientos hay en España.

```js
db.listingsAndReviews.countDocuments({ "address.country": "Spain" });
```

- Lista los 10 primeros:
  - Ordenados por precio de forma ascendente.
  - Sólo muestra: nombre, precio, camas y la localidad (`address.market`).

```js
db.listingsAndReviews
  .find(
    { "address.country": "Spain" },
    { _id: 0, name: 1, price: 1, beds: 1, "address.market": 1 }
  )
  .sort({ price: 1 })
  .limit(10);
```

### Filtrando

- Queremos viajar cómodos, somos 4 personas y queremos:
  - 4 camas.
  - Dos cuartos de baño o más.
  - Sólo muestra: nombre, precio, camas y baños.

```js
db.listingsAndReviews.find(
  {
    $and: [{ beds: 4 }, { bathrooms: { $gte: 2 } }],
  },
  { _id: 0, name: 1, price: 1, beds: 1, bathrooms: 1 }
);
```

- Aunque estamos de viaje no queremos estar desconectados, así que necesitamos que el alojamiento también tenga conexión wifi. A los requisitos anteriores, hay que añadir que el alojamiento tenga wifi.
  - Sólo muestra: nombre, precio, camas, baños y servicios (`amenities`).

```js
db.listingsAndReviews.find(
  {
    $and: [
      { beds: 4 },
      { bathrooms: { $gte: 2 } },
      {
        amenities: {
          $all: ["Wifi"],
        },
      },
    ],
  },
  { _id: 0, name: 1, price: 1, beds: 1, bathrooms: 1, amenities: 1 }
);
```

- Y bueno, un amigo trae a su perro, así que tenemos que buscar alojamientos que permitan mascota (_Pets allowed_).
  - Sólo muestra: nombre, precio, camas, baños y servicios (`amenities`).

```js
db.listingsAndReviews.find(
  {
    $and: [
      { beds: 4 },
      { bathrooms: { $gte: 2 } },
      {
        amenities: {
          $all: ["Wifi", "Pets allowed"],
        },
      },
    ],
  },
  { _id: 0, name: 1, price: 1, beds: 1, bathrooms: 1, amenities: 1 }
);
```

- Estamos entre ir a Barcelona o a Portugal, los dos destinos nos valen. Pero queremos que el precio nos salga baratito (50 $), y que tenga buen rating de reviews (campo `review_scores.review_scores_rating` igual o superior a 88).
  - Sólo muestra: nombre, precio, camas, baños, rating y localidad.

```js
db.listingsAndReviews.find(
  {
    $or: [{ "address.country": "Portugal" }, { "address.market": "Barcelona" }],
    $and: [
      { beds: 4 },
      { bathrooms: { $gte: 2 } },
      {
        amenities: {
          $all: ["Wifi", "Pets allowed"],
        },
      },
      { price: 50.0 },
      { "review_scores.review_scores_rating": { $gte: 88 } },
    ],
  },
  {
    _id: 0,
    name: 1,
    price: 1,
    beds: 1,
    bathrooms: 1,
    "review_scores.review_scores_rating": 1,
    "address.market": 1,
  }
);
```

- También queremos que el huésped sea un superhost (`host.host_is_superhost`) y que no tengamos que pagar depósito de seguridad (`security_deposit`).
  - Sólo muestra: nombre, precio, camas, baños, rating, si el huésped es superhost, depósito de seguridad y localidad.

```js
db.listingsAndReviews.find(
  {
    $or: [{ "address.country": "Portugal" }, { "address.market": "Barcelona" }],
    $and: [
      { beds: 4 },
      { bathrooms: { $gte: 2 } },
      {
        amenities: {
          $all: ["Wifi", "Pets allowed"],
        },
      },
      { price: 50.0 },
      { "review_scores.review_scores_rating": { $gte: 88 } },
      { "host.host_is_superhost": true },
      { security_deposit: 0.0 },
    ],
  },
  {
    _id: 0,
    name: 1,
    price: 1,
    beds: 1,
    bathrooms: 1,
    "review_scores.review_scores_rating": 1,
    host_is_superhost: 1,
    security_deposit: 1,
    "address.market": 1,
  }
);
```

### Agregaciones

- Queremos mostrar los alojamientos que hay en España, con los siguientes campos:
  - Nombre.
  - Localidad (no queremos mostrar un objeto, sólo el string con la localidad).
  - Precio

```js
db.listingsAndReviews.aggregate([
  { $match: { "address.country": "Spain" } },
  { $project: { _id: 0, name: 1, localidad: "$address.market", price: 1 } },
]);
```

- Queremos saber cuantos alojamientos hay disponibles por pais.

```js
db.listingsAndReviews.aggregate([
  {
    $group: {
      _id: "$address.country",
      total_alojamientos: { $sum: 1 },
    },
  },
]);
```

## Opcional

- Queremos saber el precio medio de alquiler de airbnb en España.

```js
db.listingsAndReviews.aggregate([
  { $match: { "address.country": "Spain" } },
  {
    $group: {
      _id: { country: "$address.country" },
      mediaPrecio: { $avg: "$price" },
    },
  },
]);
```

- ¿Y si quisieramos hacer como el anterior, pero sacarlo por paises?

```js
db.listingsAndReviews.aggregate([
  {
    $group: {
      _id: { country: "$address.country" },
      mediaPrecio: { $avg: "$price" },
    },
  },
]);
```

- Repite los mismos pasos pero agrupando también por numero de habitaciones.

```js
db.listingsAndReviews.aggregate([
  {
    $group: {
      _id: { country: "$address.country", numBedrooms: "$bedrooms" },
      mediaPrecio: { $avg: "$price" },
    },
  },
]);
```

## Desafio

Queremos mostrar el top 5 de alojamientos más caros en España, con los siguentes campos:

- Nombre.
- Precio.
- Número de habitaciones
- Número de camas
- Número de baños
- Ciudad.
- Servicios, pero en vez de un array, un string con todos los servicios incluidos.

```js
db.listingsAndReviews.aggregate([
  {
    $match: {
      "address.country": "Spain",
    },
  },
  {
    $sort: {
      price: -1,
    },
  },
  {
    $project: {
      _id: 0,
      name: 1,
      price: 1,
      bedrooms: 1,
      beds: 1,
      bathrooms: 1,
      "address.market": 1,
      amenities: {
        $reduce: {
          input: "$amenities",
          initialValue: "",
          in: { $concat: ["$$value", " - ", "$$this"] },
        },
      },
    },
  },
  { $limit: 5 },
]);
```
