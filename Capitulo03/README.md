# Consultas básicas y filtrado de documentos con SQL++

## Metadatos

| Atributo        | Valor                                      |
|-----------------|--------------------------------------------|
| **Duración**    | 80 minutos                                 |
| **Complejidad** | Alta                                       |
| **Nivel Bloom** | Aplicar (Apply)                            |
| **Dataset**     | `travel-sample` (inventory scope)          |
| **Herramienta** | Query Editor (Web Console) / `cbq`         |

---

## Descripción General

En este laboratorio aplicarás SQL++ (N1QL) para consultar el dataset `travel-sample` de Couchbase, avanzando desde selecciones básicas hasta operaciones avanzadas como `UNNEST`, funciones de cadena/fecha y `JOIN` entre colecciones. Trabajarás con las colecciones `airline`, `airport`, `hotel`, `route` y `landmark` del scope `inventory`, construyendo progresivamente consultas más complejas que reflejan casos de uso reales en aplicaciones distribuidas. El laboratorio está dividido en **9 secciones** que cubren todos los conceptos del módulo de forma secuencial.

---

## Objetivos de Aprendizaje

- [ ] Escribir consultas SQL++ con `SELECT`, `FROM`, `WHERE`, `LIMIT` y `ORDER BY` para recuperar y filtrar documentos del dataset `travel-sample`.
- [ ] Aplicar aliasing de campos y colecciones, concatenación de strings y selección por document key con `USE KEYS`.
- [ ] Crear índices primarios y secundarios, verificar su uso con `EXPLAIN` y optimizar consultas de filtrado.
- [ ] Construir consultas con rangos (`BETWEEN`, comparadores), funciones de agregación (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`) y `GROUP BY` con `HAVING`.
- [ ] Indexar y consultar arreglos JSON con `UNNEST` y `CREATE INDEX` en campos de arreglo, y unir colecciones con `JOIN`.

---

## Prerrequisitos

### Conocimientos
- Haber completado el **Lab 02-00-01** (Couchbase Server instalado con `travel-sample` cargado y verificado).
- Experiencia previa con SQL relacional: `SELECT`, `JOIN`, `GROUP BY`, funciones de agregación.
- Comprensión básica de estructuras JSON anidadas (objetos, arreglos, campos opcionales).

### Acceso y Herramientas
- Couchbase Server 7.6.x en ejecución (nodo único).
- Web Console accesible en `http://localhost:8091` (usuario: `Administrator`).
- Dataset `travel-sample` cargado en el bucket `travel-sample` con el scope `inventory` y las colecciones: `airline`, `airport`, `hotel`, `route`, `landmark`.
- Servicio **Query** habilitado en el nodo.
- (Opcional) `cbq` disponible para ejecución desde terminal.

---

## Entorno de Laboratorio

### Requisitos de Hardware

| Recurso        | Mínimo              | Recomendado         |
|----------------|---------------------|---------------------|
| RAM            | 8 GB disponibles    | 16 GB               |
| CPU            | 4 núcleos x86_64    | 8 núcleos           |
| Almacenamiento | 20 GB libres (SSD)  | 50 GB SSD           |
| Red            | localhost funcional | localhost funcional |
| Pantalla       | 1280×768            | 1920×1080           |

### Requisitos de Software

| Software              | Versión mínima          |
|-----------------------|-------------------------|
| Couchbase Server      | 7.6.x (CE o Enterprise) |
| Navegador web         | Chrome/Firefox/Edge 110+|
| `cbq` (opcional)      | Incluido con CB 7.6.x   |
| `curl` (opcional)     | 7.x o superior          |

### Verificación del Entorno

Antes de comenzar, ejecuta las siguientes verificaciones desde el **Query Editor** (menú **Query** en la Web Console):

```sql
-- Verificar que el bucket travel-sample está disponible
SELECT RAW COUNT(*) FROM `travel-sample`.inventory.airline;

-- Verificar todas las colecciones del scope inventory
SELECT RAW ARRAY_AGG(DISTINCT c.`collection_name`)
FROM system:keyspaces AS c
WHERE c.`bucket_name` = 'travel-sample' AND c.`scope_name` = 'inventory';
```

**Resultado esperado de la primera consulta:** un número entre 180 y 200 (aerolíneas en el dataset).  
**Resultado esperado de la segunda consulta:** `["airline","airport","hotel","landmark","route"]`

> ⚠️ **Si el dataset no está cargado:** Ve a **Settings → Sample Buckets** en la Web Console y carga `travel-sample`. Espera 2-3 minutos hasta que el indicador muestre "Loaded".

---

## Instrucciones Paso a Paso

---

### Sección 1: SELECT Básico, LIMIT y Proyección de Campos

**Objetivo:** Familiarizarse con la sintaxis fundamental de SQL++ usando `SELECT *`, proyecciones y paginación con `LIMIT`/`OFFSET` sobre la colección `airline`.

#### Paso 1.1 — Exploración inicial con SELECT *

**Instrucciones:**

1. Abre la Web Console en `http://localhost:8091` e inicia sesión.
2. Navega a la sección **Query** en el menú lateral.
3. En el Query Editor, escribe y ejecuta la siguiente consulta:

```sql
-- Recuperar los primeros 5 documentos completos de la colección airline
SELECT *
FROM `travel-sample`.inventory.airline
LIMIT 5;
```

**Resultado esperado:**  
5 objetos JSON, cada uno con una clave `airline` que envuelve los campos del documento (`id`, `type`, `name`, `iata`, `icao`, `callsign`, `country`).

```json
[
  {
    "airline": {
      "callsign": "MILE-AIR",
      "country": "United States",
      "iata": "Q5",
      "icao": "MLA",
      "id": 10,
      "name": "40-Mile Air",
      "type": "airline"
    }
  },
  ...
]
```

**Verificación:** Confirma que el panel de resultados muestra exactamente 5 objetos y que el campo `"type": "airline"` está presente en todos ellos.

---

#### Paso 1.2 — Proyección de campos específicos

**Instrucciones:**

1. Modifica la consulta para recuperar únicamente `name`, `country` e `iata` de las primeras 10 aerolíneas:

```sql
-- Proyección parcial: solo nombre, país y código IATA
SELECT name, country, iata
FROM `travel-sample`.inventory.airline
LIMIT 10;
```

**Resultado esperado:**  
10 objetos JSON con exactamente tres campos cada uno, **sin** la envoltura `"airline"`:

```json
[
  { "country": "United States", "iata": "Q5", "name": "40-Mile Air" },
  { "country": "United States", "iata": "TQ", "name": "Aircraft Guaranty Corp" },
  ...
]
```

**Verificación:** Confirma que los objetos resultado contienen **solo** los tres campos solicitados.

---

#### Paso 1.3 — Paginación con LIMIT y OFFSET

**Instrucciones:**

1. Ejecuta las tres consultas de paginación siguientes **una por una** y compara los resultados:

```sql
-- Página 1: aerolíneas 1-5
SELECT name, country
FROM `travel-sample`.inventory.airline
LIMIT 5
OFFSET 0;
```

```sql
-- Página 2: aerolíneas 6-10
SELECT name, country
FROM `travel-sample`.inventory.airline
LIMIT 5
OFFSET 5;
```

```sql
-- Página 3: aerolíneas 11-15
SELECT name, country
FROM `travel-sample`.inventory.airline
LIMIT 5
OFFSET 10;
```

**Resultado esperado:**  
Cada consulta devuelve 5 aerolíneas distintas. Los registros de la página 2 son diferentes a los de la página 1, y los de la página 3 son diferentes a ambas.

**Verificación:** Compara los nombres devueltos entre las tres páginas. No debe haber duplicados entre ellas.

> 💡 **Fórmula de paginación:** `OFFSET = (número_de_página - 1) × LIMIT`

---

### Sección 2: Aliasing (AS), Concatenación y USE KEYS

**Objetivo:** Usar `AS` para renombrar campos y colecciones en los resultados, concatenar strings con `||` y `CONCAT()`, y recuperar documentos por su clave primaria con `USE KEYS`.

#### Paso 2.1 — Aliasing de campos con AS

**Instrucciones:**

```sql
-- Renombrar campos en el resultado usando AS
SELECT name        AS nombre_aerolinea,
       country     AS pais,
       iata        AS codigo_iata,
       callsign    AS indicativo
FROM `travel-sample`.inventory.airline
LIMIT 8;
```

**Resultado esperado:**  
Los campos en el resultado usan los nombres en español definidos con `AS`.

```json
[
  {
    "codigo_iata": "Q5",
    "indicativo": "MILE-AIR",
    "nombre_aerolinea": "40-Mile Air",
    "pais": "United States"
  },
  ...
]
```

**Verificación:** Confirma que ningún campo del resultado usa los nombres originales en inglés (`name`, `country`, etc.).

---

#### Paso 2.2 — Concatenación de strings

**Instrucciones:**

1. Usa el operador `||` para concatenar campos de texto:

```sql
-- Concatenación con operador ||
SELECT name || ' (' || iata || ')' AS aerolinea_con_codigo,
       country
FROM `travel-sample`.inventory.airline
WHERE iata IS NOT MISSING
LIMIT 10;
```

2. Luego prueba la función `CONCAT()`:

```sql
-- Concatenación con función CONCAT()
SELECT CONCAT(name, ' - ', country) AS descripcion_completa,
       icao
FROM `travel-sample`.inventory.airline
LIMIT 10;
```

**Resultado esperado (primera consulta):**
```json
[
  { "aerolinea_con_codigo": "40-Mile Air (Q5)", "country": "United States" },
  ...
]
```

**Verificación:** Confirma que el formato `Nombre (IATA)` se genera correctamente en todos los resultados.

---

#### Paso 2.3 — Selección por document key con USE KEYS

**Instrucciones:**

1. Primero identifica la clave de un documento de aerolínea. En `travel-sample`, las claves siguen el patrón `airline_<id>`. Ejecuta:

```sql
-- Recuperar un documento específico por su document key
SELECT *
FROM `travel-sample`.inventory.airline
USE KEYS "airline_10";
```

2. Recupera múltiples documentos con un arreglo de keys:

```sql
-- Recuperar múltiples documentos por sus keys
SELECT name, country, iata
FROM `travel-sample`.inventory.airline
USE KEYS ["airline_10", "airline_10123", "airline_10226"];
```

**Resultado esperado (primera consulta):**  
Un único documento correspondiente a la aerolínea con `id: 10` ("40-Mile Air").

**Resultado esperado (segunda consulta):**  
Exactamente 3 documentos con los campos `name`, `country` e `iata`.

**Verificación:** Compara el `id` del documento devuelto con el número en la key (`airline_10` → `"id": 10`).

---

### Sección 3: CREATE PRIMARY INDEX, CREATE INDEX y WHERE Básico

**Objetivo:** Crear índices primarios y secundarios para habilitar y optimizar consultas de filtrado.

#### Paso 3.1 — Crear un índice primario

**Instrucciones:**

> ⚠️ El dataset `travel-sample` ya incluye índices primarios creados automáticamente. Si al ejecutar las consultas anteriores obtuviste resultados, los índices ya existen. Este paso es para practicar la sintaxis.

```sql
-- Verificar si ya existe un índice primario en airline
SELECT name, state
FROM system:indexes
WHERE keyspace_id = 'airline'
  AND bucket_id = 'travel-sample'
  AND scope_id = 'inventory';
```

Si no existe ningún índice primario, créalo:

```sql
-- Crear índice primario en la colección airline (solo si no existe)
CREATE PRIMARY INDEX `idx_airline_primary`
ON `travel-sample`.inventory.airline;
```

**Resultado esperado:**  
El comando devuelve sin errores. En `system:indexes` aparece el nuevo índice con `"state": "online"`.

**Verificación:**
```sql
SELECT name, state, `using`
FROM system:indexes
WHERE keyspace_id = 'airline' AND bucket_id = 'travel-sample';
```

---

#### Paso 3.2 — Crear un índice secundario

**Instrucciones:**

```sql
-- Crear índice secundario sobre el campo 'country' en airline
CREATE INDEX `idx_airline_country`
ON `travel-sample`.inventory.airline(country);
```

```sql
-- Crear índice secundario compuesto sobre 'country' y 'name'
CREATE INDEX `idx_airline_country_name`
ON `travel-sample`.inventory.airline(country, name);
```

**Resultado esperado:**  
Ambos índices se crean sin errores y aparecen en `system:indexes` con estado `"online"`.

---

#### Paso 3.3 — Consultas con WHERE básico

**Instrucciones:**

```sql
-- Filtrar aerolíneas de un país específico
SELECT name, iata, callsign
FROM `travel-sample`.inventory.airline
WHERE country = 'United States'
LIMIT 10;
```

```sql
-- Filtrar aerolíneas con código IATA que empieza por 'A'
SELECT name, iata, country
FROM `travel-sample`.inventory.airline
WHERE iata LIKE 'A%'
ORDER BY name
LIMIT 10;
```

**Resultado esperado (primera consulta):**  
Aerolíneas cuyo campo `country` es exactamente `'United States'`. Deben aparecer aproximadamente 150+ aerolíneas en el dataset (limitado a 10).

**Verificación:**  
Todos los registros devueltos deben tener `"country": "United States"`.

---

### Sección 4: BETWEEN, Comparadores, ORDER BY y EXPLAIN

**Objetivo:** Usar operadores de rango y ordenamiento, y analizar el plan de ejecución de consultas con `EXPLAIN`.

#### Paso 4.1 — Consultas con comparadores y BETWEEN

**Instrucciones:**

```sql
-- Aeropuertos con altitud mayor a 5000 pies
SELECT airportname, city, country, geo.alt AS altitud_pies
FROM `travel-sample`.inventory.airport
WHERE geo.alt > 5000
ORDER BY geo.alt DESC
LIMIT 10;
```

```sql
-- Aeropuertos con altitud entre 1000 y 3000 pies (BETWEEN)
SELECT airportname, city, geo.alt AS altitud_pies
FROM `travel-sample`.inventory.airport
WHERE geo.alt BETWEEN 1000 AND 3000
ORDER BY geo.alt ASC
LIMIT 15;
```

```sql
-- Aeropuertos en países específicos con altitud menor a 100 pies
SELECT airportname, country, geo.alt
FROM `travel-sample`.inventory.airport
WHERE country IN ('France', 'United Kingdom', 'Germany')
  AND geo.alt < 100
ORDER BY country ASC, airportname ASC
LIMIT 20;
```

**Resultado esperado (primera consulta):**  
Aeropuertos ordenados de mayor a menor altitud, todos con `altitud_pies > 5000`. El aeropuerto de mayor altitud en el dataset suele estar en Bolivia o Perú.

**Verificación:** Confirma que todos los valores de `altitud_pies` son mayores a 5000 y que están en orden descendente.

---

#### Paso 4.2 — ORDER BY con múltiples campos

**Instrucciones:**

```sql
-- Ordenar aeropuertos por país ASC y luego por nombre ASC
SELECT airportname, city, country
FROM `travel-sample`.inventory.airport
WHERE country IN ('France', 'United States', 'United Kingdom')
ORDER BY country ASC, airportname ASC
LIMIT 20;
```

**Resultado esperado:**  
Los aeropuertos aparecen agrupados primero por país en orden alfabético, y dentro de cada país, ordenados por nombre de aeropuerto.

---

#### Paso 4.3 — Análisis con EXPLAIN

**Instrucciones:**

1. Primero crea un índice sobre `geo.alt` para optimizar las consultas de rango:

```sql
CREATE INDEX `idx_airport_alt`
ON `travel-sample`.inventory.airport(geo.alt);
```

2. Analiza el plan de ejecución de la consulta de altitud:

```sql
EXPLAIN
SELECT airportname, city, geo.alt
FROM `travel-sample`.inventory.airport
WHERE geo.alt > 5000
ORDER BY geo.alt DESC
LIMIT 10;
```

**Resultado esperado:**  
El plan de ejecución (`#operator`) debe mostrar un `IndexScan3` que referencia el índice `idx_airport_alt`, **no** un `PrimaryScan`. Esto confirma que el índice está siendo utilizado.

**Verificación:**  
Busca en el JSON del plan el campo `"index"` — debe contener `"idx_airport_alt"`.

> 💡 **Lectura del plan EXPLAIN:** Busca los operadores `IndexScan3` (uso de índice secundario) vs `PrimaryScan` (escaneo completo). Un `IndexScan3` indica que la consulta está optimizada.

---

### Sección 5: COUNT, DISTINCT, LIKE y Funciones de Agregación

**Objetivo:** Usar funciones de agregación (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`), `DISTINCT` y el operador `LIKE` con wildcards.

#### Paso 5.1 — COUNT y DISTINCT

**Instrucciones:**

```sql
-- Contar total de aerolíneas en el dataset
SELECT COUNT(*) AS total_aerolineas
FROM `travel-sample`.inventory.airline;
```

```sql
-- Contar países únicos representados en airline
SELECT COUNT(DISTINCT country) AS paises_unicos
FROM `travel-sample`.inventory.airline;
```

```sql
-- Listar países únicos en orden alfabético
SELECT DISTINCT country
FROM `travel-sample`.inventory.airline
ORDER BY country ASC;
```

**Resultado esperado:**
- Total de aerolíneas: ~187
- Países únicos: ~36
- Lista de países en orden alfabético.

**Verificación:** El `COUNT(DISTINCT country)` debe coincidir con el número de filas devueltas por la consulta `DISTINCT country`.

---

#### Paso 5.2 — LIKE con wildcards

**Instrucciones:**

```sql
-- Aerolíneas cuyo nombre contiene 'Air' (case-sensitive)
SELECT name, country, iata
FROM `travel-sample`.inventory.airline
WHERE name LIKE '%Air%'
ORDER BY name
LIMIT 15;
```

```sql
-- Aeropuertos cuyo nombre empieza con 'San'
SELECT airportname, city, country
FROM `travel-sample`.inventory.airport
WHERE airportname LIKE 'San%'
ORDER BY airportname
LIMIT 10;
```

```sql
-- Aeropuertos con código FAA de exactamente 3 caracteres que empieza con 'L'
SELECT airportname, faa, city
FROM `travel-sample`.inventory.airport
WHERE faa LIKE 'L__'
ORDER BY faa
LIMIT 10;
```

**Resultado esperado (primera consulta):**  
Aerolíneas como "40-Mile Air", "Air Berlin", "Air Canada", "Air France", etc.

**Verificación:** Todos los nombres en el resultado deben contener la subcadena `Air` (mayúscula A).

---

#### Paso 5.3 — SUM, AVG, MIN, MAX sobre campos numéricos

**Instrucciones:**

```sql
-- Estadísticas de altitud de aeropuertos
SELECT COUNT(*)           AS total_aeropuertos,
       AVG(geo.alt)       AS altitud_promedio,
       MIN(geo.alt)       AS altitud_minima,
       MAX(geo.alt)       AS altitud_maxima,
       SUM(geo.alt)       AS suma_altitudes
FROM `travel-sample`.inventory.airport
WHERE geo.alt IS NOT MISSING;
```

**Resultado esperado:**  
Un único objeto con las cinco métricas calculadas. La altitud promedio suele estar alrededor de 800-1200 pies.

**Verificación:** Confirma que `altitud_minima` ≤ `altitud_promedio` ≤ `altitud_maxima`.

---

### Sección 6: IS MISSING, IS NULL, GROUP BY y HAVING

**Objetivo:** Manejar campos opcionales con `IS MISSING` e `IS NULL`, y construir agregaciones con `GROUP BY` filtradas con `HAVING`.

#### Paso 6.1 — IS MISSING e IS NULL

**Instrucciones:**

```sql
-- Aerolíneas sin código IATA (campo faltante o nulo)
SELECT name, country, iata
FROM `travel-sample`.inventory.airline
WHERE iata IS MISSING OR iata IS NULL
LIMIT 10;
```

```sql
-- Aerolíneas que SÍ tienen código IATA
SELECT COUNT(*) AS con_iata
FROM `travel-sample`.inventory.airline
WHERE iata IS NOT MISSING AND iata IS NOT NULL;
```

```sql
-- Aeropuertos sin coordenadas geográficas completas
SELECT airportname, city
FROM `travel-sample`.inventory.airport
WHERE geo IS MISSING OR geo.lat IS MISSING OR geo.lon IS MISSING
LIMIT 10;
```

**Resultado esperado (primera consulta):**  
Aerolíneas que no tienen el campo `iata` en su documento JSON, o que lo tienen con valor `null`.

> 💡 **Diferencia clave:** `IS MISSING` detecta campos que no existen en el documento JSON. `IS NULL` detecta campos que existen pero tienen valor `null`. En SQL++, ambas condiciones son distintas y deben manejarse por separado.

---

#### Paso 6.2 — GROUP BY

**Instrucciones:**

```sql
-- Contar aerolíneas por país
SELECT country,
       COUNT(*) AS total_aerolineas
FROM `travel-sample`.inventory.airline
GROUP BY country
ORDER BY total_aerolineas DESC
LIMIT 10;
```

```sql
-- Contar aeropuertos por país
SELECT country,
       COUNT(*)       AS total_aeropuertos,
       AVG(geo.alt)   AS altitud_promedio
FROM `travel-sample`.inventory.airport
GROUP BY country
ORDER BY total_aeropuertos DESC
LIMIT 10;
```

**Resultado esperado (primera consulta):**  
Lista de países con su conteo de aerolíneas. Estados Unidos debe aparecer en el primer lugar con ~150 aerolíneas.

---

#### Paso 6.3 — HAVING para filtrar grupos

**Instrucciones:**

```sql
-- Países con más de 5 aerolíneas
SELECT country,
       COUNT(*) AS total_aerolineas
FROM `travel-sample`.inventory.airline
GROUP BY country
HAVING COUNT(*) > 5
ORDER BY total_aerolineas DESC;
```

```sql
-- Países con más de 50 aeropuertos y altitud promedio mayor a 500 pies
SELECT country,
       COUNT(*)     AS total_aeropuertos,
       AVG(geo.alt) AS altitud_promedio
FROM `travel-sample`.inventory.airport
GROUP BY country
HAVING COUNT(*) > 50 AND AVG(geo.alt) > 500
ORDER BY total_aeropuertos DESC;
```

**Resultado esperado (primera consulta):**  
Solo los países con más de 5 aerolíneas. Deben aparecer aproximadamente 5-8 países.

**Verificación:** Todos los valores de `total_aerolineas` deben ser mayores a 5.

---

### Sección 7: UNNEST para Arreglos y Array Indexes

**Objetivo:** Descomponer arreglos JSON con `UNNEST` y crear índices optimizados para campos de arreglo.

#### Paso 7.1 — Explorar la estructura de documentos con arreglos

**Instrucciones:**

1. Primero examina la estructura de un documento `route` para entender sus arreglos:

```sql
-- Ver la estructura completa de un documento route
SELECT *
FROM `travel-sample`.inventory.route
LIMIT 1;
```

2. Observa el campo `schedule`, que es un arreglo de objetos con días y horarios de vuelo.

```sql
-- Ver solo el campo schedule de una ruta
SELECT r.airline, r.destinationairport, r.schedule
FROM `travel-sample`.inventory.route AS r
LIMIT 1;
```

**Resultado esperado:**  
Un documento `route` con campos como `airline`, `sourceairport`, `destinationairport`, `distance`, y un arreglo `schedule` con múltiples objetos `{day, flight, utc}`.

---

#### Paso 7.2 — UNNEST para descomponer arreglos

**Instrucciones:**

```sql
-- Descomponer el arreglo schedule de rutas usando UNNEST
SELECT r.airline,
       r.sourceairport,
       r.destinationairport,
       s.day    AS dia,
       s.flight AS vuelo,
       s.utc    AS hora_utc
FROM `travel-sample`.inventory.route AS r
UNNEST r.schedule AS s
WHERE r.sourceairport = 'SFO'
LIMIT 15;
```

```sql
-- Contar vuelos por día de la semana desde SFO
SELECT s.day        AS dia_semana,
       COUNT(*)     AS total_vuelos
FROM `travel-sample`.inventory.route AS r
UNNEST r.schedule AS s
WHERE r.sourceairport = 'SFO'
GROUP BY s.day
ORDER BY s.day ASC;
```

**Resultado esperado (primera consulta):**  
Filas individuales para cada elemento del arreglo `schedule`, con los datos de la ruta repetidos en cada fila.

**Resultado esperado (segunda consulta):**  
7 filas (días 0-6) con el conteo de vuelos programados para cada día desde SFO.

**Verificación:** El total de filas de la primera consulta debe ser igual a la suma de todos los valores `total_vuelos` de la segunda consulta (para el mismo filtro `sourceairport = 'SFO'`).

---

#### Paso 7.3 — CREATE INDEX en campos de arreglo

**Instrucciones:**

```sql
-- Crear índice sobre el campo flight dentro del arreglo schedule
CREATE INDEX `idx_route_schedule_flight`
ON `travel-sample`.inventory.route
    (ALL ARRAY s.flight FOR s IN schedule END);
```

```sql
-- Verificar que el índice existe
SELECT name, state
FROM system:indexes
WHERE keyspace_id = 'route'
  AND bucket_id = 'travel-sample'
  AND name = 'idx_route_schedule_flight';
```

```sql
-- Consulta que usa el array index
SELECT r.airline, r.sourceairport, r.destinationairport
FROM `travel-sample`.inventory.route AS r
WHERE ANY s IN r.schedule SATISFIES s.flight = 'AF138' END
LIMIT 10;
```

**Resultado esperado:**  
El índice se crea con estado `"online"`. La consulta `ANY...SATISFIES` devuelve las rutas que tienen el vuelo `AF138` en su schedule.

**Verificación:** Ejecuta `EXPLAIN` sobre la última consulta y confirma que usa `idx_route_schedule_flight`.

```sql
EXPLAIN
SELECT r.airline, r.sourceairport, r.destinationairport
FROM `travel-sample`.inventory.route AS r
WHERE ANY s IN r.schedule SATISFIES s.flight = 'AF138' END;
```

---

### Sección 8: Funciones de String, Fecha y Tipo

**Objetivo:** Aplicar funciones integradas de SQL++ para manipular strings, fechas y tipos de datos.

#### Paso 8.1 — Funciones de string

**Instrucciones:**

```sql
-- UPPER, LOWER, LENGTH, SUBSTR sobre campos de aerolíneas
SELECT name,
       UPPER(name)          AS nombre_mayusculas,
       LOWER(country)       AS pais_minusculas,
       LENGTH(name)         AS longitud_nombre,
       SUBSTR(name, 0, 5)   AS primeros_5_chars
FROM `travel-sample`.inventory.airline
LIMIT 10;
```

```sql
-- Buscar aerolíneas y normalizar presentación
SELECT UPPER(iata)                              AS codigo_iata,
       INITCAP(LOWER(name))                     AS nombre_formateado,
       LTRIM(RTRIM(country))                    AS pais_limpio,
       POSITION(name, 'Air')                    AS posicion_air
FROM `travel-sample`.inventory.airline
WHERE name LIKE '%Air%'
  AND iata IS NOT MISSING
LIMIT 10;
```

**Resultado esperado (primera consulta):**  
Cada fila muestra el nombre original, su versión en mayúsculas, el país en minúsculas, la longitud del nombre y los primeros 5 caracteres.

---

#### Paso 8.2 — Funciones de fecha

**Instrucciones:**

```sql
-- Obtener la fecha y hora actual en diferentes formatos
SELECT NOW_STR()                          AS fecha_actual_iso,
       NOW_MILLIS()                       AS timestamp_millis,
       DATE_PART_STR(NOW_STR(), 'year')   AS anio_actual,
       DATE_PART_STR(NOW_STR(), 'month')  AS mes_actual,
       DATE_PART_STR(NOW_STR(), 'day')    AS dia_actual;
```

```sql
-- Convertir timestamp a string legible
SELECT STR_TO_MILLIS('2024-01-15T10:30:00Z')  AS millis_fecha,
       MILLIS_TO_STR(1705316200000)            AS fecha_desde_millis,
       DATE_DIFF_STR('2025-12-31', '2025-01-01', 'day') AS dias_restantes_2025;
```

**Resultado esperado (primera consulta):**  
Un objeto con la fecha actual en formato ISO 8601, el timestamp en milisegundos y los componentes de fecha separados.

---

#### Paso 8.3 — Funciones de tipo (TOSTRING, TONUMBER, TYPE)

**Instrucciones:**

```sql
-- Conversión de tipos y verificación
SELECT name,
       id,
       TOSTRING(id)          AS id_como_string,
       TYPE(id)              AS tipo_id,
       TYPE(name)            AS tipo_name,
       TYPE(geo)             AS tipo_geo
FROM `travel-sample`.inventory.airport
LIMIT 5;
```

```sql
-- Concatenar campo numérico con string usando TOSTRING
SELECT airportname || ' (ID: ' || TOSTRING(id) || ')' AS descripcion,
       faa,
       TONUMBER(faa)   AS faa_como_numero
FROM `travel-sample`.inventory.airport
WHERE faa IS NOT MISSING
LIMIT 10;
```

**Resultado esperado (primera consulta):**  
Cada fila muestra el tipo de dato de cada campo: `"number"` para `id`, `"string"` para `name`, `"object"` para `geo`.

**Verificación:** Confirma que `TOSTRING(id)` convierte el entero a string (aparece entre comillas en el JSON resultado).

---

### Sección 9: JOIN entre Colecciones

**Objetivo:** Unir documentos de diferentes colecciones usando `JOIN` con `ON KEYS` para enriquecer los resultados.

#### Paso 9.1 — JOIN básico entre route y airline

**Instrucciones:**

1. Primero entiende la relación: los documentos `route` tienen un campo `airlineid` que contiene la document key de la aerolínea correspondiente (ej: `"airline_10"`).

```sql
-- Verificar la estructura de la relación
SELECT r.airline, r.airlineid, r.sourceairport, r.destinationairport
FROM `travel-sample`.inventory.route AS r
LIMIT 3;
```

2. Ahora realiza el JOIN:

```sql
-- JOIN entre route y airline para obtener detalles de la aerolínea
SELECT r.sourceairport,
       r.destinationairport,
       r.distance,
       a.name          AS nombre_aerolinea,
       a.country       AS pais_aerolinea,
       a.callsign      AS indicativo
FROM `travel-sample`.inventory.route AS r
JOIN `travel-sample`.inventory.airline AS a
  ON KEYS r.airlineid
WHERE r.sourceairport = 'SFO'
LIMIT 15;
```

**Resultado esperado:**  
Rutas que salen de SFO enriquecidas con los datos completos de la aerolínea operadora (nombre, país, indicativo).

**Verificación:** Confirma que el campo `nombre_aerolinea` está poblado en todos los registros (no `null` ni `missing`).

---

#### Paso 9.2 — JOIN con filtros adicionales

**Instrucciones:**

```sql
-- Rutas de aerolíneas francesas con distancia mayor a 5000 km
SELECT r.sourceairport,
       r.destinationairport,
       r.distance,
       a.name    AS aerolinea,
       a.iata    AS codigo_iata
FROM `travel-sample`.inventory.route AS r
JOIN `travel-sample`.inventory.airline AS a
  ON KEYS r.airlineid
WHERE a.country = 'France'
  AND r.distance > 5000
ORDER BY r.distance DESC
LIMIT 15;
```

```sql
-- Contar rutas por aerolínea (JOIN + GROUP BY)
SELECT a.name          AS aerolinea,
       a.country       AS pais,
       COUNT(*)        AS total_rutas,
       AVG(r.distance) AS distancia_promedio_km
FROM `travel-sample`.inventory.route AS r
JOIN `travel-sample`.inventory.airline AS a
  ON KEYS r.airlineid
GROUP BY a.name, a.country
HAVING COUNT(*) > 100
ORDER BY total_rutas DESC
LIMIT 10;
```

**Resultado esperado (primera consulta):**  
Rutas operadas por aerolíneas francesas con distancia superior a 5000 km, ordenadas de mayor a menor distancia.

**Resultado esperado (segunda consulta):**  
Aerolíneas con más de 100 rutas registradas, con su distancia promedio. Las grandes aerolíneas (United, Delta, American) deben aparecer en los primeros lugares.

---

#### Paso 9.3 — JOIN con tres colecciones

**Instrucciones:**

```sql
-- JOIN entre route, airline y airport para obtener información completa
SELECT r.distance,
       a.name                AS aerolinea,
       src.airportname       AS aeropuerto_origen,
       src.city              AS ciudad_origen,
       dst.airportname       AS aeropuerto_destino,
       dst.city              AS ciudad_destino
FROM `travel-sample`.inventory.route AS r
JOIN `travel-sample`.inventory.airline AS a
  ON KEYS r.airlineid
JOIN `travel-sample`.inventory.airport AS src
  ON KEYS ('airport_' || TOSTRING(src.id))
WHERE r.sourceairport = src.faa
  AND r.destinationairport = 'CDG'
  AND a.country = 'United States'
ORDER BY r.distance ASC
LIMIT 10;
```

> ⚠️ **Nota:** Los JOINs con tres colecciones pueden ser costosos sin índices adecuados. Si la consulta tarda más de 30 segundos, cancélala y continúa con el paso de verificación.

**Alternativa más eficiente:**

```sql
-- Versión optimizada con filtros en los campos indexados
SELECT r.sourceairport    AS origen,
       r.destinationairport AS destino,
       r.distance,
       a.name             AS aerolinea
FROM `travel-sample`.inventory.route AS r
JOIN `travel-sample`.inventory.airline AS a
  ON KEYS r.airlineid
WHERE r.destinationairport = 'CDG'
  AND a.country = 'United States'
ORDER BY r.distance ASC
LIMIT 10;
```

**Resultado esperado:**  
Rutas de aerolíneas estadounidenses hacia el aeropuerto Charles de Gaulle (CDG), ordenadas por distancia.

---

## Validación y Pruebas

Ejecuta las siguientes consultas de validación para confirmar que todas las secciones se completaron correctamente:

```sql
-- Validación 1: Verificar índices creados en este laboratorio
SELECT name, keyspace_id, state
FROM system:indexes
WHERE bucket_id = 'travel-sample'
  AND name IN (
    'idx_airline_primary',
    'idx_airline_country',
    'idx_airline_country_name',
    'idx_airport_alt',
    'idx_route_schedule_flight'
  )
ORDER BY name;
```

**Resultado esperado:** 5 filas, todas con `"state": "online"`.

```sql
-- Validación 2: Confirmar que GROUP BY funciona correctamente
SELECT country, COUNT(*) AS total
FROM `travel-sample`.inventory.airline
GROUP BY country
HAVING COUNT(*) > 10
ORDER BY total DESC;
```

**Resultado esperado:** Entre 3 y 5 países con más de 10 aerolíneas. Estados Unidos debe encabezar la lista.

```sql
-- Validación 3: Confirmar que UNNEST funciona
SELECT COUNT(*) AS total_segmentos_schedule
FROM `travel-sample`.inventory.route AS r
UNNEST r.schedule AS s
WHERE r.sourceairport = 'SFO';
```

**Resultado esperado:** Un número mayor a 100 (cantidad de segmentos de horario para rutas desde SFO).

```sql
-- Validación 4: Confirmar que JOIN funciona
SELECT COUNT(*) AS total_rutas_con_aerolinea
FROM `travel-sample`.inventory.route AS r
JOIN `travel-sample`.inventory.airline AS a
  ON KEYS r.airlineid;
```

**Resultado esperado:** Un número cercano al total de rutas en el dataset (~17,000+).

---

## Solución de Problemas

### Problema 1: Error "No index available" al ejecutar consultas con WHERE

**Síntoma:**  
Al ejecutar una consulta con filtro `WHERE` sobre la colección `airline` o `airport`, Couchbase devuelve el error:
```
"No index available on keyspace `travel-sample`.inventory.airline that matches your query."
```

**Causa:**  
No existe un índice primario ni un índice secundario que cubra el campo utilizado en la cláusula `WHERE`. Couchbase requiere al menos un índice primario para realizar escaneos completos, o un índice secundario para optimizar filtros específicos.

**Solución:**

1. Verifica los índices existentes:
```sql
SELECT name, state, keyspace_id
FROM system:indexes
WHERE bucket_id = 'travel-sample'
ORDER BY keyspace_id, name;
```

2. Si no existe un índice primario para la colección afectada, créalo:
```sql
-- Reemplaza 'airline' con la colección que causa el error
CREATE PRIMARY INDEX ON `travel-sample`.inventory.airline;
```

3. Para consultas frecuentes sobre campos específicos, crea un índice secundario:
```sql
CREATE INDEX idx_airline_country ON `travel-sample`.inventory.airline(country);
```

4. Espera a que el índice alcance el estado `"online"` antes de reintentar la consulta.

---

### Problema 2: UNNEST devuelve menos resultados de los esperados o resultados vacíos

**Síntoma:**  
La consulta con `UNNEST r.schedule AS s` devuelve 0 filas o significativamente menos filas que las esperadas, aunque se sabe que los documentos `route` tienen el campo `schedule`.

**Causa:**  
`UNNEST` en SQL++ realiza por defecto un **INNER JOIN** implícito: si el campo `schedule` está ausente (`MISSING`), es `null`, o es un arreglo vacío `[]` en algún documento, ese documento es **excluido** completamente del resultado. Esto puede reducir drásticamente el número de resultados.

**Solución:**

1. Verifica cuántos documentos tienen el campo `schedule` poblado:
```sql
SELECT COUNT(*) AS con_schedule,
       (SELECT RAW COUNT(*) FROM `travel-sample`.inventory.route) AS total_rutas
FROM `travel-sample`.inventory.route
WHERE schedule IS NOT MISSING AND ARRAY_LENGTH(schedule) > 0;
```

2. Si necesitas incluir documentos sin `schedule`, usa `LEFT UNNEST` (outer join):
```sql
SELECT r.airline, r.sourceairport, r.destinationairport,
       s.day AS dia, s.flight AS vuelo
FROM `travel-sample`.inventory.route AS r
LEFT UNNEST r.schedule AS s
WHERE r.sourceairport = 'SFO'
LIMIT 20;
```

3. Con `LEFT UNNEST`, los documentos sin `schedule` aparecen en el resultado con `dia` y `vuelo` como `null`.

---

## Limpieza del Entorno

Ejecuta las siguientes instrucciones para eliminar los índices creados durante el laboratorio y dejar el entorno en su estado original:

```sql
-- Eliminar índices secundarios creados en el laboratorio
DROP INDEX `idx_airline_country`
  ON `travel-sample`.inventory.airline;

DROP INDEX `idx_airline_country_name`
  ON `travel-sample`.inventory.airline;

DROP INDEX `idx_airport_alt`
  ON `travel-sample`.inventory.airport;

DROP INDEX `idx_route_schedule_flight`
  ON `travel-sample`.inventory.route;
```

```sql
-- Eliminar el índice primario solo si fue creado manualmente en este lab
-- (omitir si ya existía antes del laboratorio)
DROP INDEX `idx_airline_primary`
  ON `travel-sample`.inventory.airline;
```

```sql
-- Verificar que los índices del lab han sido eliminados
SELECT COUNT(*) AS indices_restantes
FROM system:indexes
WHERE bucket_id = 'travel-sample'
  AND name IN (
    'idx_airline_primary',
    'idx_airline_country',
    'idx_airline_country_name',
    'idx_airport_alt',
    'idx_route_schedule_flight'
  );
```

**Resultado esperado:** `"indices_restantes": 0`

> ⚠️ **No elimines** los índices que venían pre-cargados con `travel-sample` (como `def_inventory_airline_primary`). Solo elimina los índices que creaste explícitamente en este laboratorio.

---

## Resumen

En este laboratorio aplicaste los conceptos fundamentales de SQL++ sobre el dataset `travel-sample` de Couchbase, cubriendo el ciclo completo desde consultas básicas hasta operaciones avanzadas:

| Sección | Concepto Practicado | Colección(es) Usada(s) |
|---------|--------------------|-----------------------|
| 1 | `SELECT *`, proyección, `LIMIT`, `OFFSET` | `airline` |
| 2 | `AS` aliasing, `\|\|`, `CONCAT()`, `USE KEYS` | `airline` |
| 3 | `CREATE PRIMARY INDEX`, `CREATE INDEX`, `WHERE` | `airline` |
| 4 | `BETWEEN`, comparadores, `ORDER BY`, `EXPLAIN` | `airport` |
| 5 | `COUNT`, `DISTINCT`, `LIKE`, `SUM`, `AVG`, `MIN`, `MAX` | `airline`, `airport` |
| 6 | `IS MISSING`, `IS NULL`, `GROUP BY`, `HAVING` | `airline`, `airport` |
| 7 | `UNNEST`, array index con `ALL ARRAY` | `route` |
| 8 | `UPPER`, `LOWER`, `SUBSTR`, `NOW_STR`, `TOSTRING` | `airline`, `airport` |
| 9 | `JOIN ... ON KEYS`, multi-colección | `route`, `airline`, `airport` |

### Conceptos Clave Aprendidos

- **Proyección vs SELECT \*:** Seleccionar solo los campos necesarios reduce el tamaño de la respuesta y mejora el rendimiento.
- **Índices son obligatorios:** Sin índices, Couchbase rechaza consultas con `WHERE` sobre colecciones (a menos que exista un índice primario).
- **EXPLAIN es tu aliado:** Siempre verifica el plan de ejecución para confirmar que tus índices están siendo utilizados (`IndexScan3` vs `PrimaryScan`).
- **UNNEST = INNER JOIN implícito:** Documentos con arreglos vacíos o ausentes son excluidos; usa `LEFT UNNEST` si necesitas incluirlos.
- **JOIN ON KEYS:** En Couchbase, los JOINs se realizan sobre document keys, lo que los hace muy eficientes comparado con JOINs por valor.

### Recursos Adicionales

- [SQL++ Language Reference — Couchbase Docs](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/index.html)
- [CREATE INDEX en SQL++](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/createindex.html)
- [UNNEST Clause Reference](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/from.html#unnest)
- [JOIN en SQL++ — Couchbase Docs](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/join.html)
- [Funciones integradas de SQL++](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/functions.html)
- [Couchbase Query Workbench — Guía de uso](https://docs.couchbase.com/server/current/tools/query-workbench.html)

---
