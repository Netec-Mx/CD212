# Solución de referencia — Práctica 1

## Análisis de casos de uso NoSQL vs Relacional

> Esta solución es un ejemplo de referencia. Otras respuestas pueden ser correctas si están justificadas técnicamente.

---

## Tarea 1. Analizar el problema relacional de ViajeYA

### 1.1 Síntomas identificados

**Rendimiento**

- Las búsquedas tardan entre 4 y 8 segundos.
- El servidor principal alcanza 95 % de CPU.
- La consulta revisa aproximadamente 45 millones de filas.
- Los JOINs incrementan el costo de ejecución.

**Cambios en el modelo**

- Las migraciones requieren ventanas de mantenimiento.
- Agregar nuevos alojamientos puede requerir columnas o tablas nuevas.
- `amenities` almacenado como `TEXT` limita las consultas estructuradas.

**Consistencia**

- Las réplicas pueden mostrar datos ligeramente desactualizados.
- Una lectura podría no reflejar inmediatamente el último cambio.

### 1.2 Análisis del modelo relacional

La tabla `hotels` almacena los datos generales del hotel. `rooms` relaciona cada habitación con un hotel mediante `hotel_id`. `availability` guarda una fila por habitación y fecha, por lo que puede crecer rápidamente.

Ejemplo:

```text
500,000 habitaciones × 365 fechas = 182,500,000 filas
```

Guardar `amenities` como texto dificulta buscar campos específicos como `wifi`, `pool` o `parking`, validar tipos y mantener un formato consistente.

### 1.3 Interpretación del plan

`Seq Scan on availability` indica que el motor revisa secuencialmente una gran cantidad de filas. Revisar 45 millones de filas consume CPU, memoria, lectura de disco y tiempo. Los JOINs entre `hotels`, `rooms` y `availability` agregan más comparaciones y combinaciones de datos.

### 1.4 Tabla completada

| Síntoma observado | Causa raíz probable | Categoría |
|---|---|---|
| La búsqueda tarda entre 4 y 8 segundos | La consulta revisa millones de filas y realiza varios JOINs | Escala |
| Las migraciones requieren mantenimiento | El esquema es rígido y debe modificarse para nuevos atributos | Variedad |
| El servidor opera al 95 % de CPU | Las consultas pesadas concentran procesamiento en el nodo principal | Velocidad |
| Las réplicas muestran datos desactualizados | Existe retraso de replicación y consistencia eventual | Velocidad |

**Respuestas**

1. `Seq Scan` significa que se recorren secuencialmente muchas filas.
2. Revisar 45 millones de filas aumenta consumo y latencia.
3. `amenities` como texto no permite consultar propiedades estructuradas.
4. `glamping` podría requerir migraciones o tablas adicionales para sus atributos.

---

## Tarea 2. Documento JSON desnormalizado

```json
{
  "type": "hotel",
  "id": "hotel_001",
  "name": "Hotel Mediterráneo",
  "location": {
    "city": "Barcelona",
    "country": "España"
  },
  "stars": 4,
  "amenities": {
    "wifi": true,
    "pool": true,
    "parking": false,
    "breakfast_included": true,
    "air_conditioning": true
  },
  "rooms": [
    {
      "room_id": "room_001",
      "type": "standard",
      "capacity": 2,
      "price_per_night": 120.0,
      "available_dates": [
        "2024-12-20",
        "2024-12-21",
        "2024-12-22"
      ]
    },
    {
      "room_id": "room_002",
      "type": "glamping",
      "capacity": 4,
      "price_per_night": 180.0,
      "tent_material": "canvas",
      "max_wind_speed": 70,
      "eco_certification": true,
      "available_dates": [
        "2024-12-20",
        "2024-12-23",
        "2024-12-24"
      ]
    }
  ]
}
```

### Justificación

- `location` agrupa ciudad y país.
- `amenities` permite consultar cada amenidad como campo booleano.
- `rooms` integra las habitaciones dentro del hotel.
- `available_dates` evita consultar una tabla separada.
- `glamping` puede agregar atributos propios sin modificar las demás habitaciones.

### Reflexión

1. Se eliminan los JOINs `hotels → rooms` y `rooms → availability`.
2. Un objeto JSON permite consultar amenidades específicas.
3. El esquema flexible admite atributos particulares.
4. Un documento demasiado grande aumenta costo de transferencia, lectura y actualización.

| Criterio | Estado |
|---|---|
| JSON válido | Completado |
| Datos generales | Completado |
| Amenidades como objeto | Completado |
| Habitaciones como arreglo | Completado |
| Disponibilidad integrada | Completado |
| Tipo con atributos propios | Completado |

---

## Tarea 3. Clasificación de escenarios NoSQL

| Escenario | Tipo NoSQL elegido | Justificación |
|---|---|---|
| A. Carrito de compras | Clave-Valor | Se recupera por ID de usuario, requiere baja latencia y expiración |
| B. Catálogo de productos | Documental | Cada categoría tiene atributos diferentes y requiere consultas por campos |
| C. Telemetría IoT | Columnar | Recibe escrituras masivas y consultas por sensor y rango de tiempo |
| D. Red social profesional | Grafos | Necesita recorrer relaciones entre personas, empresas e industrias |

### Explicación

**Carrito:** patrón `user_id → carrito`, ideal para clave-valor.

**Catálogo:** los documentos permiten esquemas diferentes por categoría.

**IoT:** el modelo columnar soporta grandes volúmenes y consultas por rango.

**Red social:** un grafo representa nodos y relaciones de forma natural.

---

## Tarea 4. Decisiones CAP

| Escenario | Clasificación | Justificación |
|---|---|---|
| Transferencia bancaria | CP | Debe evitar saldos inconsistentes y dobles débitos |
| Feed de red social | AP | Es aceptable una demora breve si el servicio continúa disponible |
| Últimos asientos de un vuelo | CP | Debe evitar vender el mismo asiento dos veces |

### ¿Por qué no elegir siempre CP?

Porque CP puede reducir la disponibilidad durante una partición de red. En feeds sociales, recomendaciones, catálogos o métricas no críticas puede ser preferible responder con datos ligeramente desactualizados en lugar de dejar de responder.

---

## Validación final

| Entregable | Estado |
|---|---|
| Tabla de análisis relacional | Completado |
| Documento JSON desnormalizado | Completado |
| Clasificación NoSQL | Completado |
| Clasificación CAP | Completado |

## Resultado final

La solución identifica problemas de escala, rigidez y costo de consulta. El documento JSON reduce JOINs y permite atributos flexibles. Los escenarios NoSQL se clasifican por patrón de acceso y las decisiones CAP se justifican por el costo de una inconsistencia temporal.
