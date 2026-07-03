# Ejecución de consultas analíticas con SQL++ Analytics

## Metadatos

| Campo | Detalle |
|---|---|
| **Duración estimada** | 50 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (Apply) |
| **Servicio principal** | Couchbase Analytics Service |
| **Dataset requerido** | travel-sample |

---

## Descripción General

Este laboratorio introduce el servicio Analytics de Couchbase como solución para cargas de trabajo OLAP que deben operar de forma aislada respecto a las operaciones transaccionales OLTP del servicio Query. Los estudiantes configurarán un entorno Analytics completo —creando un Dataverse, Datasets y Links— sobre el bucket `travel-sample`, ejecutarán consultas analíticas SQL++ de complejidad creciente, y aprenderán a monitorear su ejecución tanto desde la Web Console como mediante el REST API y el shell `cbas`. Al finalizar, comprenderán cuándo y por qué elegir Analytics sobre el servicio Query estándar.

---

## Objetivos de Aprendizaje

Al completar este laboratorio, el estudiante será capaz de:

- [ ] Explicar las diferencias arquitectónicas entre el servicio Query (OLTP) y el servicio Analytics (OLAP) de Couchbase, incluyendo el rol del protocolo DCP en la sincronización de shadow datasets.
- [ ] Configurar un entorno Analytics funcional creando un Dataverse, Datasets y el Link Local sobre el bucket `travel-sample`.
- [ ] Ejecutar consultas SQL++ analíticas con `COUNT`, `GROUP BY`, `AVG`, `JOIN` y subconsultas sobre los datasets configurados.
- [ ] Interpretar la estructura completa de una respuesta REST del Analytics Service, incluyendo los campos `results`, `metrics`, `status`, `errors` y `warnings`.
- [ ] Utilizar el shell `cbas` y el REST API con `curl` para ejecutar y automatizar consultas analíticas desde la línea de comandos.

---

## Prerrequisitos

### Conocimientos previos

- Conocimiento sólido de SQL++ `SELECT` con `JOIN`, `GROUP BY`, funciones de agregación (`COUNT`, `SUM`, `AVG`) y subconsultas.
- Comprensión de la estructura de documentos en el bucket `travel-sample` (colecciones `airports`, `hotels`, `routes`, `landmarks`).
- Familiaridad con el uso básico de `curl` y la Web Console de Couchbase.
- Haber completado los labs anteriores de la secuencia (especialmente los labs de SQL++ del servicio Query).

### Acceso y software requerido

| Componente | Versión mínima | Notas |
|---|---|---|
| Couchbase Server | 7.6.x | Con servicio Analytics habilitado |
| RAM asignada a Analytics | 4 GB | Configurar en Server Settings |
| Bucket `travel-sample` | Cargado y activo | ~31,000 documentos |
| Navegador web | Chrome 110+ / Firefox 110+ | Para Web Console |
| `curl` | 7.x o superior | Para ejercicios REST API |
| `cbas` shell | Incluido con Couchbase 7.6.x | Analytics Shell |

---

## Entorno de Laboratorio

### Verificación del servicio Analytics

Antes de iniciar, confirma que el servicio Analytics está activo en tu nodo Couchbase.

**Desde la Web Console:**
1. Navega a `http://localhost:8091` e inicia sesión (usuario: `Administrator`, contraseña: `password` o la configurada en tu entorno).
2. Ve a **Servers** → selecciona tu nodo → verifica que **Analytics** aparece en la columna de servicios activos.

**Desde la línea de comandos:**
```bash
curl -s -u Administrator:password \
  http://localhost:8091/pools/default \
  | python3 -m json.tool | grep -i analytics
```

### Verificación del bucket travel-sample

```bash
curl -s -u Administrator:password \
  http://localhost:8091/pools/default/buckets/travel-sample \
  | python3 -m json.tool | grep '"name"'
```

Si el bucket no existe, cárgalo desde **Settings → Sample Buckets → travel-sample → Load Sample Data**.

### Verificación del endpoint Analytics

```bash
curl -s -u Administrator:password \
  http://localhost:8095/analytics/cluster \
  | python3 -m json.tool
```

> **Nota:** El servicio Analytics escucha en el puerto `8095` (HTTP) y `18095` (HTTPS). Si ves `"state": "ACTIVE"`, el servicio está listo.

---

## Pasos del Laboratorio

---

### Paso 1: Explorar la interfaz Analytics en la Web Console

**Objetivo:** Familiarizarse con el editor Analytics de la Web Console antes de ejecutar comandos, identificando sus componentes principales.

#### Instrucciones

1. Abre tu navegador y navega a `http://localhost:8091`.
2. Inicia sesión con tus credenciales de administrador.
3. En el menú lateral izquierdo, haz clic en **Analytics**.
4. Observa los componentes del editor:
   - **Panel izquierdo (Insights):** muestra los Dataverses y Datasets configurados.
   - **Editor central:** área para escribir y ejecutar consultas SQL++ Analytics.
   - **Panel inferior (Results):** muestra resultados, métricas y errores.
5. En el editor, escribe la siguiente consulta de exploración y presiona el botón **Execute** (▶):

```sql
SELECT VALUE dv
FROM Metadata.`Dataverse` dv;
```

6. Observa los resultados: verás el Dataverse `Default` que existe por defecto en todo clúster Analytics.

#### Salida esperada

```json
[
  {
    "DataverseName": "Default",
    "Pending": false,
    "Timestamp": "..."
  }
]
```

#### Verificación

En el panel de resultados, confirma que el campo `"DataverseName"` tiene el valor `"Default"`. Esto confirma que el servicio Analytics está operativo y puede responder consultas.

---

### Paso 2: Crear el Dataverse TravelAnalytics

**Objetivo:** Crear el namespace analítico `TravelAnalytics` que contendrá todos los Datasets de este laboratorio.

#### Instrucciones

1. En el editor Analytics de la Web Console, escribe y ejecuta:

```sql
CREATE DATAVERSE TravelAnalytics IF NOT EXISTS;
```

2. Verifica que el Dataverse fue creado consultando los metadatos:

```sql
SELECT VALUE dv.DataverseName
FROM Metadata.`Dataverse` dv
WHERE dv.DataverseName = "TravelAnalytics";
```

3. Establece `TravelAnalytics` como el Dataverse activo para la sesión:

```sql
USE TravelAnalytics;
```

> **Concepto clave:** Un **Dataverse** en Analytics es equivalente a un schema o namespace en bases de datos relacionales. Permite agrupar Datasets relacionados y aislar objetos entre distintos equipos o proyectos analíticos.

#### Salida esperada

Después del `CREATE DATAVERSE`:
```
Results: []
Status: success
```

Después del `SELECT`:
```json
["TravelAnalytics"]
```

#### Verificación

En el panel **Insights** (lateral izquierdo de la Web Console Analytics), actualiza la vista. Deberías ver `TravelAnalytics` listado como un Dataverse disponible.

---

### Paso 3: Crear Datasets sobre travel-sample

**Objetivo:** Definir cuatro shadow datasets (`airports`, `hotels`, `routes`, `landmarks`) que sincronizarán datos desde el bucket `travel-sample` usando el Link Local predeterminado.

#### Instrucciones

1. Asegúrate de estar en el contexto `TravelAnalytics` (ejecuta `USE TravelAnalytics;` si es necesario).

2. Crea el Dataset para **airports**:

```sql
USE TravelAnalytics;

CREATE DATASET airports ON `travel-sample`.inventory.airport
  WHERE type = "airport";
```

3. Crea el Dataset para **hotels**:

```sql
CREATE DATASET hotels ON `travel-sample`.inventory.hotel
  WHERE type = "hotel";
```

4. Crea el Dataset para **routes**:

```sql
CREATE DATASET routes ON `travel-sample`.inventory.route
  WHERE type = "route";
```

5. Crea el Dataset para **landmarks**:

```sql
CREATE DATASET landmarks ON `travel-sample`.inventory.landmark
  WHERE type = "landmark";
```

6. Verifica que los cuatro Datasets fueron creados:

```sql
USE TravelAnalytics;

SELECT VALUE ds.DatasetName
FROM Metadata.`Dataset` ds
WHERE ds.DataverseName = "TravelAnalytics"
ORDER BY ds.DatasetName;
```

> **Concepto clave:** Un **Dataset** en Analytics es un *shadow dataset*: una copia de los datos operacionales que se mantiene sincronizada mediante el protocolo **DCP (Database Change Protocol)**. Esta sincronización es automática, continua y en tiempo casi real. No es necesario ejecutar ningún proceso ETL manual.

> **Nota sobre el Link Local:** Al crear un Dataset sobre un bucket local (en el mismo clúster), Analytics usa automáticamente el **Link Local** predeterminado. No es necesario declararlo explícitamente en este caso.

#### Salida esperada

```json
["airports", "hotels", "landmarks", "routes"]
```

#### Verificación

Ejecuta una consulta rápida de conteo para confirmar que los datos se están sincronizando:

```sql
USE TravelAnalytics;

SELECT
  (SELECT VALUE COUNT(*) FROM airports)[0]   AS total_airports,
  (SELECT VALUE COUNT(*) FROM hotels)[0]     AS total_hotels,
  (SELECT VALUE COUNT(*) FROM routes)[0]     AS total_routes,
  (SELECT VALUE COUNT(*) FROM landmarks)[0]  AS total_landmarks;
```

Valores de referencia aproximados del dataset `travel-sample`:

| Dataset | Documentos esperados |
|---|---|
| airports | ~1,968 |
| hotels | ~917 |
| routes | ~24,024 |
| landmarks | ~4,495 |

> Si los conteos son 0, espera 10-15 segundos y vuelve a ejecutar. La sincronización DCP puede tardar unos momentos en completarse tras la creación del Dataset.

---

### Paso 4: Consultas analíticas de exploración

**Objetivo:** Ejecutar consultas SQL++ analíticas básicas con `COUNT`, `GROUP BY` y `AVG` para familiarizarse con la sintaxis y el comportamiento de Analytics.

#### Instrucciones

**Consulta 4.1 — Distribución de aeropuertos por país:**

```sql
USE TravelAnalytics;

SELECT a.country,
       COUNT(*) AS total_aeropuertos
FROM airports a
GROUP BY a.country
ORDER BY total_aeropuertos DESC
LIMIT 10;
```

**Consulta 4.2 — Promedio de calificación de hoteles por ciudad (top 10 ciudades con más hoteles):**

```sql
USE TravelAnalytics;

SELECT h.city,
       COUNT(*)            AS total_hoteles,
       AVG(h.reviews[0].ratings.Overall) AS avg_rating_general
FROM hotels h
WHERE h.city IS NOT NULL
  AND ARRAY_LENGTH(h.reviews) > 0
GROUP BY h.city
ORDER BY total_hoteles DESC
LIMIT 10;
```

**Consulta 4.3 — Rutas por aerolínea (top 10 aerolíneas con más rutas):**

```sql
USE TravelAnalytics;

SELECT r.airline,
       COUNT(*) AS total_rutas,
       COUNT(DISTINCT r.sourceairport) AS aeropuertos_origen_distintos
FROM routes r
GROUP BY r.airline
ORDER BY total_rutas DESC
LIMIT 10;
```

**Consulta 4.4 — Landmarks por actividad (top 5 tipos de actividad):**

```sql
USE TravelAnalytics;

SELECT l.activity,
       COUNT(*) AS total_landmarks
FROM landmarks l
WHERE l.activity IS NOT NULL
GROUP BY l.activity
ORDER BY total_landmarks DESC
LIMIT 5;
```

#### Salida esperada (Consulta 4.1 — ejemplo parcial)

```json
[
  { "country": "United States", "total_aeropuertos": 1845 },
  { "country": "United Kingdom", "total_aeropuertos": 48 },
  { "country": "France", "total_aeropuertos": 22 },
  ...
]
```

#### Verificación

En el panel de resultados, revisa el campo **Metrics** (o **Status**). Deberías ver algo similar a:

```json
{
  "elapsedTime": "245.123ms",
  "executionTime": "210.456ms",
  "resultCount": 10,
  "resultSize": 1234,
  "processedObjects": 1968
}
```

Anota los tiempos de ejecución. Los compararás con el servicio Query en el siguiente paso.

---

### Paso 5: Consultas con JOIN entre Datasets

**Objetivo:** Ejecutar consultas analíticas que relacionen múltiples Datasets, observando cómo Analytics maneja JOINs sobre grandes volúmenes de datos sin impactar el servicio Query.

#### Instrucciones

**Consulta 5.1 — Rutas con información del aeropuerto de origen:**

```sql
USE TravelAnalytics;

SELECT r.airline,
       r.sourceairport,
       r.destinationairport,
       a.airportname,
       a.city   AS ciudad_origen,
       a.country AS pais_origen
FROM routes r
JOIN airports a ON r.sourceairport = a.faa
WHERE r.airline = "AA"
ORDER BY r.destinationairport
LIMIT 15;
```

**Consulta 5.2 — Hoteles y landmarks en la misma ciudad:**

```sql
USE TravelAnalytics;

SELECT h.name   AS nombre_hotel,
       h.city,
       h.country,
       l.name   AS nombre_landmark,
       l.activity
FROM hotels h
JOIN landmarks l
  ON h.city = l.city
 AND h.country = l.country
WHERE h.country = "United States"
  AND l.activity = "Eat/Drink"
ORDER BY h.city, h.name
LIMIT 20;
```

**Consulta 5.3 — Estadísticas de rutas por aeropuerto (JOIN con agregación):**

```sql
USE TravelAnalytics;

SELECT a.airportname,
       a.city,
       a.country,
       COUNT(r.id) AS rutas_salientes,
       COUNT(DISTINCT r.airline) AS aerolineas_distintas
FROM airports a
JOIN routes r ON a.faa = r.sourceairport
WHERE a.country = "United States"
GROUP BY a.airportname, a.city, a.country
HAVING COUNT(r.id) > 50
ORDER BY rutas_salientes DESC
LIMIT 10;
```

#### Salida esperada (Consulta 5.3 — ejemplo parcial)

```json
[
  {
    "airportname": "Hartsfield Jackson Atlanta Intl",
    "city": "Atlanta",
    "country": "United States",
    "rutas_salientes": 342,
    "aerolineas_distintas": 12
  },
  ...
]
```

#### Verificación

Observa en el panel Metrics el campo `processedObjects`. Para la Consulta 5.1, este número debería ser significativamente mayor que el número de resultados devueltos, reflejando el escaneo completo del dataset `routes` (~24,000 documentos). Esto ilustra por qué Analytics es preferible para este tipo de operaciones: el escaneo pesado ocurre en el nodo Analytics sin afectar el servicio Query.

---

### Paso 6: Consultas avanzadas con subconsultas y agregaciones múltiples

**Objetivo:** Escribir consultas analíticas complejas que combinen subconsultas, agregaciones anidadas y filtros avanzados.

#### Instrucciones

**Consulta 6.1 — Aeropuertos con más de la media de rutas salientes:**

```sql
USE TravelAnalytics;

WITH avg_rutas AS (
  SELECT VALUE AVG(conteo)
  FROM (
    SELECT COUNT(*) AS conteo
    FROM routes r
    GROUP BY r.sourceairport
  ) subq
)
SELECT a.airportname,
       a.city,
       a.country,
       COUNT(r.id) AS rutas_salientes
FROM airports a
JOIN routes r ON a.faa = r.sourceairport
GROUP BY a.airportname, a.city, a.country
HAVING COUNT(r.id) > (SELECT VALUE avg_rutas FROM avg_rutas)[0]
ORDER BY rutas_salientes DESC
LIMIT 15;
```

**Consulta 6.2 — Distribución de hoteles por rango de calificación:**

```sql
USE TravelAnalytics;

SELECT
  CASE
    WHEN overall_avg >= 4.5 THEN "Excelente (4.5-5.0)"
    WHEN overall_avg >= 4.0 THEN "Muy Bueno (4.0-4.4)"
    WHEN overall_avg >= 3.0 THEN "Bueno (3.0-3.9)"
    WHEN overall_avg >= 2.0 THEN "Regular (2.0-2.9)"
    ELSE "Bajo (< 2.0)"
  END AS rango_calificacion,
  COUNT(*) AS cantidad_hoteles
FROM (
  SELECT h.name,
         AVG(rev.ratings.Overall) AS overall_avg
  FROM hotels h
  UNNEST h.reviews AS rev
  WHERE rev.ratings.Overall IS NOT NULL
  GROUP BY h.name
) hotel_ratings
GROUP BY
  CASE
    WHEN overall_avg >= 4.5 THEN "Excelente (4.5-5.0)"
    WHEN overall_avg >= 4.0 THEN "Muy Bueno (4.0-4.4)"
    WHEN overall_avg >= 3.0 THEN "Bueno (3.0-3.9)"
    WHEN overall_avg >= 2.0 THEN "Regular (2.0-2.9)"
    ELSE "Bajo (< 2.0)"
  END
ORDER BY cantidad_hoteles DESC;
```

**Consulta 6.3 — Análisis completo: aerolíneas con cobertura en múltiples países:**

```sql
USE TravelAnalytics;

SELECT r.airline,
       COUNT(DISTINCT src.country)  AS paises_origen,
       COUNT(DISTINCT dst.country)  AS paises_destino,
       COUNT(*)                     AS total_rutas,
       COUNT(DISTINCT r.sourceairport) AS aeropuertos_origen
FROM routes r
JOIN airports src ON r.sourceairport = src.faa
JOIN airports dst ON r.destinationairport = dst.faa
GROUP BY r.airline
HAVING COUNT(DISTINCT src.country) >= 3
ORDER BY paises_origen DESC, total_rutas DESC
LIMIT 10;
```

#### Salida esperada (Consulta 6.2 — ejemplo)

```json
[
  { "rango_calificacion": "Muy Bueno (4.0-4.4)", "cantidad_hoteles": 312 },
  { "rango_calificacion": "Excelente (4.5-5.0)", "cantidad_hoteles": 287 },
  { "rango_calificacion": "Bueno (3.0-3.9)", "cantidad_hoteles": 198 },
  { "rango_calificacion": "Regular (2.0-2.9)", "cantidad_hoteles": 45 },
  { "rango_calificacion": "Bajo (< 2.0)", "cantidad_hoteles": 12 }
]
```

#### Verificación

Compara los tiempos de ejecución de las tres consultas. La Consulta 6.3 (doble JOIN + múltiples DISTINCT) debería ser la más lenta. Anota el `elapsedTime` de cada una. Esta comparación ilustra cómo la complejidad de la consulta afecta el tiempo de procesamiento analítico.

---

### Paso 7: Monitorear consultas analíticas desde la Web Console

**Objetivo:** Utilizar el panel de monitoreo de Analytics en la Web Console para observar el estado de las consultas en ejecución, tiempos de respuesta y uso de recursos.

#### Instrucciones

1. En la Web Console, navega a **Analytics** en el menú lateral.
2. Ejecuta la siguiente consulta que tiene un tiempo de procesamiento más largo (no pongas LIMIT para forzar un escaneo completo):

```sql
USE TravelAnalytics;

SELECT r.airline,
       r.sourceairport,
       r.destinationairport,
       src.city AS ciudad_origen,
       dst.city AS ciudad_destino
FROM routes r
JOIN airports src ON r.sourceairport = src.faa
JOIN airports dst ON r.destinationairport = dst.faa
WHERE src.country = "United States";
```

3. Mientras la consulta se ejecuta, abre una nueva pestaña del navegador y navega a:
   ```
   http://localhost:8095/analytics/admin/active_requests
   ```
4. Observa las consultas activas. Verás información como:
   - `clientContextID`: identificador único de la consulta
   - `statement`: el texto de la consulta
   - `elapsedTime`: tiempo transcurrido
   - `state`: estado actual (`running`, `completed`)

5. Regresa a la pestaña de la Web Console y espera a que la consulta termine.
6. En el panel de resultados, haz clic en la pestaña **Plan** (si está disponible) para ver el plan de ejecución de la consulta analítica.

#### Salida esperada (endpoint active_requests)

```json
{
  "activeRequests": [
    {
      "clientContextID": "analytics-1234-abcd",
      "statement": "USE TravelAnalytics; SELECT r.airline ...",
      "elapsedTime": "1.234s",
      "state": "running"
    }
  ]
}
```

#### Verificación

Una vez completada la consulta, verifica en el panel **Metrics** de la Web Console:

```json
{
  "elapsedTime": "...",
  "executionTime": "...",
  "resultCount": ...,
  "resultSize": ...,
  "processedObjects": ...
}
```

Confirma que `processedObjects` es mayor que `resultCount`, lo que indica que Analytics procesó más documentos de los que devolvió (filtrado durante el JOIN).

---

### Paso 8: Interpretar la estructura de una respuesta REST de Analytics

**Objetivo:** Analizar la estructura completa de una respuesta del Analytics REST API, identificando los campos `results`, `metrics`, `status`, `errors` y `warnings`.

#### Instrucciones

1. Abre una terminal y ejecuta la siguiente consulta via REST API:

```bash
curl -s -u Administrator:password \
  -H "Content-Type: application/json" \
  -X POST http://localhost:8095/analytics/service \
  -d '{
    "statement": "USE TravelAnalytics; SELECT a.country, COUNT(*) AS total FROM airports a GROUP BY a.country ORDER BY total DESC LIMIT 5;",
    "pretty": true,
    "client_context_id": "lab14-test-001"
  }' | python3 -m json.tool
```

2. Observa la estructura completa de la respuesta. Identifica cada sección:

```json
{
  "requestID": "...",
  "clientContextID": "lab14-test-001",
  "signature": {
    "*": "*"
  },
  "results": [
    { "country": "United States", "total": 1845 },
    ...
  ],
  "status": "success",
  "metrics": {
    "elapsedTime": "...",
    "executionTime": "...",
    "resultCount": 5,
    "resultSize": ...,
    "processedObjects": 1968,
    "errorCount": 0,
    "warningCount": 0
  }
}
```

3. Ahora ejecuta una consulta con un error intencional para observar la estructura del campo `errors`:

```bash
curl -s -u Administrator:password \
  -H "Content-Type: application/json" \
  -X POST http://localhost:8095/analytics/service \
  -d '{
    "statement": "USE TravelAnalytics; SELECT * FROM tabla_inexistente;",
    "pretty": true
  }' | python3 -m json.tool
```

4. Observa el campo `errors` en la respuesta:

```json
{
  "requestID": "...",
  "status": "fatal",
  "errors": [
    {
      "code": 24045,
      "msg": "Cannot find dataset with name tabla_inexistente ..."
    }
  ],
  "metrics": {
    "elapsedTime": "...",
    "executionTime": "...",
    "resultCount": 0,
    "resultSize": 0,
    "processedObjects": 0,
    "errorCount": 1
  }
}
```

#### Referencia de campos de la respuesta Analytics

| Campo | Descripción |
|---|---|
| `requestID` | UUID único generado por el servidor para esta solicitud |
| `clientContextID` | ID enviado por el cliente (útil para correlacionar logs) |
| `signature` | Esquema inferido de los resultados |
| `results` | Array con los documentos resultado de la consulta |
| `status` | `success`, `running`, `errors`, `fatal`, `timeout`, `stopped` |
| `metrics.elapsedTime` | Tiempo total desde recepción hasta respuesta |
| `metrics.executionTime` | Tiempo de ejecución pura de la consulta |
| `metrics.resultCount` | Número de documentos en `results` |
| `metrics.processedObjects` | Total de documentos escaneados/procesados |
| `errors` | Array de errores (vacío si `status` = `success`) |
| `warnings` | Array de advertencias no fatales |

#### Verificación

Confirma que puedes identificar correctamente cada campo en la respuesta. En particular, verifica que:
- Una consulta exitosa tiene `"status": "success"` y `"errors": []`.
- Una consulta fallida tiene `"status": "fatal"` y el array `errors` contiene al menos un objeto con `code` y `msg`.

---

### Paso 9: Usar el Analytics Shell (cbas)

**Objetivo:** Ejecutar consultas analíticas desde la línea de comandos usando el shell `cbas`, y crear un script analítico reutilizable.

#### Instrucciones

1. Abre una terminal en tu sistema. Localiza el shell `cbas`:

```bash
# En instalación estándar de Couchbase Server en Linux:
/opt/couchbase/bin/cbas

# En macOS (instalación típica):
/Applications/Couchbase\ Server.app/Contents/Resources/couchbase-core/bin/cbas

# Verificar disponibilidad:
which cbas || find /opt/couchbase -name "cbas" 2>/dev/null
```

2. Conéctate al servidor Analytics:

```bash
cbas -u Administrator -p password -s http://localhost:8095
```

3. Una vez en el prompt de `cbas` (`cbas>`), ejecuta las siguientes consultas:

```sql
USE TravelAnalytics;
```

```sql
SELECT COUNT(*) AS total_aeropuertos FROM airports;
```

```sql
SELECT a.country, COUNT(*) AS total
FROM airports a
GROUP BY a.country
ORDER BY total DESC
LIMIT 5;
```

4. Sal del shell con:
```
\quit
```

5. Ahora crea un archivo de script analítico. Abre tu editor de texto y crea el archivo `analytics_report.sqlpp`:

```sql
-- analytics_report.sqlpp
-- Script analítico: Reporte de cobertura aérea por país
-- Uso: cbas -u Administrator -p password -s http://localhost:8095 -f analytics_report.sqlpp

USE TravelAnalytics;

-- 1. Resumen general
SELECT "=== RESUMEN GENERAL ===" AS seccion;

SELECT
  (SELECT VALUE COUNT(*) FROM airports)[0]   AS total_aeropuertos,
  (SELECT VALUE COUNT(*) FROM routes)[0]     AS total_rutas,
  (SELECT VALUE COUNT(*) FROM hotels)[0]     AS total_hoteles,
  (SELECT VALUE COUNT(*) FROM landmarks)[0]  AS total_landmarks;

-- 2. Top 5 países con más aeropuertos
SELECT "=== TOP 5 PAISES - AEROPUERTOS ===" AS seccion;

SELECT a.country, COUNT(*) AS total_aeropuertos
FROM airports a
GROUP BY a.country
ORDER BY total_aeropuertos DESC
LIMIT 5;

-- 3. Top 5 aerolíneas con más rutas
SELECT "=== TOP 5 AEROLINEAS - RUTAS ===" AS seccion;

SELECT r.airline, COUNT(*) AS total_rutas
FROM routes r
GROUP BY r.airline
ORDER BY total_rutas DESC
LIMIT 5;
```

6. Ejecuta el script desde la línea de comandos:

```bash
cbas -u Administrator -p password \
     -s http://localhost:8095 \
     -f analytics_report.sqlpp
```

#### Salida esperada

```
USE TravelAnalytics;
OK

SELECT "=== RESUMEN GENERAL ===" AS seccion;
[ { "seccion": "=== RESUMEN GENERAL ===" } ]

SELECT (SELECT VALUE COUNT(*) FROM airports)[0] ...
[ { "total_aeropuertos": 1968, "total_rutas": 24024, "total_hoteles": 917, "total_landmarks": 4495 } ]

...
```

#### Verificación

Confirma que el script se ejecutó completamente sin errores. Todos los bloques `SELECT` deben devolver resultados con `OK` o datos JSON. Si algún bloque falla, revisa la sintaxis del archivo `.sqlpp`.

---

### Paso 10: Consumir el Analytics REST API con curl para automatización

**Objetivo:** Automatizar la ejecución de consultas analíticas usando el REST API de Analytics, simulando un flujo de integración programática.

#### Instrucciones

1. **Consulta con parámetros y formato de salida controlado:**

```bash
curl -s -u Administrator:password \
  -H "Content-Type: application/json" \
  -X POST http://localhost:8095/analytics/service \
  -d '{
    "statement": "USE TravelAnalytics; SELECT r.airline, COUNT(*) AS rutas FROM routes r GROUP BY r.airline ORDER BY rutas DESC LIMIT 5;",
    "format": "application/json",
    "timeout": "30s",
    "client_context_id": "lab14-automation-001"
  }'
```

2. **Extraer solo el array de resultados con Python:**

```bash
curl -s -u Administrator:password \
  -H "Content-Type: application/json" \
  -X POST http://localhost:8095/analytics/service \
  -d '{
    "statement": "USE TravelAnalytics; SELECT a.country, COUNT(*) AS total FROM airports a GROUP BY a.country ORDER BY total DESC LIMIT 3;"
  }' | python3 -c "
import json, sys
response = json.load(sys.stdin)
print(f'Status: {response[\"status\"]}')
print(f'Tiempo de ejecución: {response[\"metrics\"][\"executionTime\"]}')
print(f'Documentos procesados: {response[\"metrics\"][\"processedObjects\"]}')
print('Resultados:')
for row in response['results']:
    print(f'  {row}')
"
```

3. **Verificar el estado del clúster Analytics:**

```bash
curl -s -u Administrator:password \
  http://localhost:8095/analytics/cluster \
  | python3 -m json.tool
```

4. **Consultar el historial de solicitudes completadas:**

```bash
curl -s -u Administrator:password \
  http://localhost:8095/analytics/admin/completed_requests \
  | python3 -m json.tool | head -50
```

5. **Script Bash completo de automatización** — Crea el archivo `run_analytics.sh`:

```bash
#!/bin/bash
# run_analytics.sh — Ejecuta una consulta analítica y verifica el resultado

CB_HOST="localhost"
CB_PORT="8095"
CB_USER="Administrator"
CB_PASS="password"

QUERY="USE TravelAnalytics; SELECT a.country, COUNT(*) AS total FROM airports a GROUP BY a.country ORDER BY total DESC LIMIT 3;"

echo "Ejecutando consulta analítica..."
RESPONSE=$(curl -s -u "${CB_USER}:${CB_PASS}" \
  -H "Content-Type: application/json" \
  -X POST "http://${CB_HOST}:${CB_PORT}/analytics/service" \
  -d "{\"statement\": \"${QUERY}\", \"client_context_id\": \"bash-script-$(date +%s)\"}")

STATUS=$(echo "$RESPONSE" | python3 -c "import json,sys; print(json.load(sys.stdin)['status'])")
ELAPSED=$(echo "$RESPONSE" | python3 -c "import json,sys; print(json.load(sys.stdin)['metrics']['elapsedTime'])")

echo "Estado: $STATUS"
echo "Tiempo transcurrido: $ELAPSED"

if [ "$STATUS" = "success" ]; then
  echo "Resultados:"
  echo "$RESPONSE" | python3 -c "
import json, sys
data = json.load(sys.stdin)
for row in data['results']:
    print(f\"  {row['country']}: {row['total']} aeropuertos\")
"
else
  echo "ERROR en la consulta:"
  echo "$RESPONSE" | python3 -m json.tool
fi
```

```bash
chmod +x run_analytics.sh
./run_analytics.sh
```

#### Salida esperada

```
Ejecutando consulta analítica...
Estado: success
Tiempo transcurrido: 123.456ms
Resultados:
  United States: 1845 aeropuertos
  United Kingdom: 48 aeropuertos
  France: 22 aeropuertos
```

#### Verificación

Confirma que el script devuelve `Estado: success` y muestra los tres países con más aeropuertos. Si el script falla, verifica que `curl`, `python3` y los permisos del archivo son correctos.

---

## Validación y Pruebas Finales

Ejecuta las siguientes verificaciones para confirmar que el laboratorio está completo:

### Verificación 1: Dataverse y Datasets creados

```sql
USE TravelAnalytics;

SELECT VALUE {
  "dataverse": (SELECT VALUE COUNT(*) FROM Metadata.`Dataverse` WHERE DataverseName = "TravelAnalytics")[0],
  "datasets": (SELECT VALUE COUNT(*) FROM Metadata.`Dataset` WHERE DataverseName = "TravelAnalytics")[0]
};
```

**Resultado esperado:** `[{ "dataverse": 1, "datasets": 4 }]`

### Verificación 2: Todos los datasets tienen datos

```sql
USE TravelAnalytics;

SELECT
  "airports"   AS dataset, (SELECT VALUE COUNT(*) FROM airports)[0]   AS count UNION ALL
SELECT
  "hotels"     AS dataset, (SELECT VALUE COUNT(*) FROM hotels)[0]     AS count UNION ALL
SELECT
  "routes"     AS dataset, (SELECT VALUE COUNT(*) FROM routes)[0]     AS count UNION ALL
SELECT
  "landmarks"  AS dataset, (SELECT VALUE COUNT(*) FROM landmarks)[0]  AS count;
```

**Resultado esperado:** Los cuatro datasets deben tener conteos mayores que 0.

### Verificación 3: JOIN entre datasets funcional

```sql
USE TravelAnalytics;

SELECT COUNT(*) AS rutas_con_aeropuerto_origen
FROM routes r
JOIN airports a ON r.sourceairport = a.faa;
```

**Resultado esperado:** Un número mayor que 0 (típicamente > 20,000).

### Verificación 4: REST API responde correctamente

```bash
curl -s -u Administrator:password \
  -H "Content-Type: application/json" \
  -X POST http://localhost:8095/analytics/service \
  -d '{"statement": "USE TravelAnalytics; SELECT VALUE COUNT(*) FROM airports;"}' \
  | python3 -c "
import json, sys
r = json.load(sys.stdin)
assert r['status'] == 'success', f'Estado inesperado: {r[\"status\"]}'
assert r['results'][0] > 0, 'No se encontraron aeropuertos'
print(f'PASS: REST API funcional. Aeropuertos: {r[\"results\"][0]}')
"
```

**Resultado esperado:** `PASS: REST API funcional. Aeropuertos: 1968` (o similar).

---

## Resolución de Problemas

### Problema 1: Los Datasets muestran COUNT(*) = 0 después de crearlos

**Síntoma:** Al ejecutar `SELECT VALUE COUNT(*) FROM airports;` inmediatamente después de crear el Dataset, el resultado es `[0]` o el Dataset parece vacío.

**Causa:** La sincronización inicial de datos mediante el protocolo DCP (Database Change Protocol) no es instantánea. Cuando se crea un Dataset en Analytics, Couchbase debe transferir todos los documentos existentes desde el bucket operacional al shadow dataset. Este proceso puede tardar entre 15 segundos y varios minutos dependiendo del volumen de datos y la carga del sistema.

**Solución:**
1. Espera al menos 30 segundos y vuelve a ejecutar el conteo.
2. Si persiste, verifica el estado de sincronización desde la Web Console: **Analytics → Insights → [nombre del Dataset]** → observa el indicador de sincronización.
3. Alternativamente, consulta el estado del Link Local:
   ```sql
   SELECT VALUE lnk
   FROM Metadata.`Link` lnk
   WHERE lnk.DataverseName = "TravelAnalytics";
   ```
4. Si el problema continúa, verifica que el bucket `travel-sample` tiene documentos en la colección correspondiente:
   ```bash
   curl -s -u Administrator:password \
     "http://localhost:8093/query/service" \
     -d 'statement=SELECT COUNT(*) AS total FROM `travel-sample`.inventory.airport'
   ```
5. Si el bucket tiene datos pero el Dataset sigue vacío después de 2 minutos, intenta eliminar y recrear el Dataset:
   ```sql
   USE TravelAnalytics;
   DROP DATASET airports IF EXISTS;
   CREATE DATASET airports ON `travel-sample`.inventory.airport WHERE type = "airport";
   ```

---

### Problema 2: Error "Cannot connect to Analytics service" al usar curl o cbas

**Síntoma:** Al ejecutar `curl http://localhost:8095/analytics/service` se obtiene `Connection refused` o `curl: (7) Failed to connect`. El shell `cbas` tampoco puede conectarse.

**Causa:** El servicio Analytics no está habilitado en el nodo Couchbase, o no se le asignó suficiente memoria RAM durante la configuración del clúster. El servicio Analytics requiere un mínimo de 1 GB de RAM para iniciar (se recomiendan 4 GB). También puede ocurrir si el puerto 8095 está bloqueado por el firewall local.

**Solución:**
1. Verifica si el servicio Analytics está habilitado en el nodo:
   ```bash
   curl -s -u Administrator:password \
     http://localhost:8091/pools/default/nodeServices \
     | python3 -m json.tool | grep -i "cbas\|analytics\|8095"
   ```
2. Si el servicio no aparece, habilítalo desde la Web Console:
   - Navega a **Servers → [tu nodo] → Edit** → marca el checkbox **Analytics** → **Save**.
   - **Advertencia:** Cambiar los servicios de un nodo requiere reiniciarlo. Sigue las instrucciones del asistente de rebalanceo.
3. Verifica la memoria asignada al servicio Analytics:
   - Ve a **Settings → Analytics** → confirma que la memoria asignada es al menos 1024 MB (recomendado: 4096 MB).
4. Si el servicio está habilitado pero no responde, verifica que el proceso está corriendo:
   ```bash
   # En Linux:
   ps aux | grep cbas
   # O verifica el log:
   tail -100 /opt/couchbase/var/lib/couchbase/logs/analytics.log | grep -i "error\|started\|listening"
   ```
5. Verifica que el puerto no está bloqueado:
   ```bash
   nc -zv localhost 8095
   ```
6. Si usas Docker, verifica que el puerto 8095 está mapeado en el contenedor:
   ```bash
   docker ps --format "table {{.Names}}\t{{.Ports}}" | grep couchbase
   ```
   Si no está mapeado, recrea el contenedor incluyendo `-p 8095:8095` en el comando `docker run`.

---

## Limpieza del Entorno

Después de completar el laboratorio, puedes optar por mantener los Datasets para labs posteriores o eliminarlos para liberar recursos.

### Opción A: Mantener el entorno (recomendado si hay labs posteriores)

Los Datasets y el Dataverse `TravelAnalytics` pueden mantenerse activos. El consumo de recursos es mínimo cuando no hay consultas en ejecución.

### Opción B: Eliminar los recursos creados en este lab

```sql
-- Eliminar datasets individuales
USE TravelAnalytics;
DROP DATASET airports IF EXISTS;
DROP DATASET hotels IF EXISTS;
DROP DATASET routes IF EXISTS;
DROP DATASET landmarks IF EXISTS;

-- Eliminar el Dataverse completo (elimina todos los objetos dentro de él)
DROP DATAVERSE TravelAnalytics IF EXISTS;
```

### Eliminar archivos creados localmente

```bash
# Eliminar scripts creados durante el lab
rm -f analytics_report.sqlpp
rm -f run_analytics.sh
```

### Verificar limpieza

```sql
SELECT VALUE COUNT(*)
FROM Metadata.`Dataverse`
WHERE DataverseName = "TravelAnalytics";
```

**Resultado esperado después de la limpieza:** `[0]`

---

## Resumen

En este laboratorio aplicaste los conceptos fundamentales del servicio Analytics de Couchbase en un entorno real con el dataset `travel-sample`. Los puntos clave que practicaste fueron:

| Habilidad | Actividad realizada |
|---|---|
| **Arquitectura Analytics** | Exploraste la separación OLTP/OLAP y el rol del protocolo DCP en la sincronización de shadow datasets |
| **Configuración del entorno** | Creaste el Dataverse `TravelAnalytics` y cuatro Datasets usando el Link Local predeterminado |
| **Consultas analíticas básicas** | Ejecutaste queries con `COUNT`, `GROUP BY`, `AVG` y `UNNEST` sobre datasets individuales |
| **JOINs analíticos** | Relacionaste múltiples Datasets en consultas con agregaciones complejas |
| **Subconsultas y CTEs** | Usaste `WITH` (CTEs) y subconsultas anidadas para análisis multietapa |
| **Monitoreo** | Observaste queries activas via Web Console y el endpoint `/analytics/admin/active_requests` |
| **Respuesta REST** | Interpretaste los campos `results`, `metrics`, `status`, `errors` y `warnings` de la API |
| **cbas shell** | Ejecutaste consultas interactivas y scripts `.sqlpp` desde la línea de comandos |
| **REST API con curl** | Automatizaste consultas analíticas con scripts Bash y procesamiento Python |

### Diferencias clave recordadas: Analytics vs Query

| Característica | Servicio Query | Servicio Analytics |
|---|---|---|
| **Puerto** | 8093 | 8095 |
| **Carga de trabajo** | OLTP (baja latencia) | OLAP (alta complejidad) |
| **Aislamiento** | Comparte recursos con Data Service | Nodo independiente |
| **Acceso a datos** | Directo al bucket | Via shadow datasets (DCP) |
| **Latencia típica** | Milisegundos | Segundos a minutos |
| **Namespace** | Bucket → Scope → Collection | Dataverse → Dataset |
| **Shell** | `cbq` | `cbas` |

### Recursos Adicionales

- [Documentación oficial: Couchbase Analytics Service](https://docs.couchbase.com/server/current/analytics/introduction.html)
- [SQL++ for Analytics — Referencia del lenguaje](https://docs.couchbase.com/server/current/analytics/sql-pp-reference.html)
- [Analytics REST API Reference](https://docs.couchbase.com/server/current/rest-api/rest-analytics.html)
- [Database Change Protocol (DCP)](https://docs.couchbase.com/server/current/learn/clusters-and-availability/database-change-protocol.html)
- [Workload Isolation en Couchbase Analytics](https://www.couchbase.com/blog/couchbase-analytics-workload-isolation/)
- [Apache AsterixDB — Motor subyacente de Analytics](https://asterixdb.apache.org/)

---
*Lab 14-00-01 — Couchbase Analytics Service — SQL++ for Analytics*
