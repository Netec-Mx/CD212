---
layout: lab
title: "Práctica 4: Creación y validación de índices en Couchbase"
permalink: /lab4/lab4/
images_base: /labs/lab4/img
duration: "70 minutos"
objective:
  - Preparar el directorio de trabajo de la práctica 4 y validar que Couchbase Server continúe disponible.
  - Inventariar los índices existentes mediante la colección del sistema system:indexes.
  - Crear y validar índices secundarios simples, compuestos, parciales y de arreglos.
  - Crear un índice con construcción diferida y observar sus estados deferred, building y online.
  - Crear un índice particionado y comprender sus limitaciones en un clúster de nodo único.
  - Explorar los índices desde la Web Console y validar capacidades disponibles en Couchbase Server Enterprise Edition.
prerequisites:
  - Haber completado la Práctica 3.
  - Tener Docker Desktop en ejecución.
  - Tener activo el contenedor couchbase-lab creado con la imagen couchbase/server:enterprise-7.6.2.
  - Tener cargado el bucket travel-sample.
  - Tener acceso a la Web Console en http://localhost:8091.
  - Utilizar Visual Studio Code con Git Bash como terminal integrada.
introduction:
  - En esta práctica trabajarás con el Index Service de Couchbase Server Enterprise Edition sobre el dataset travel-sample. Primero revisarás los índices existentes y después crearás índices secundarios simples, compuestos, parciales y de arreglos. También practicarás la construcción diferida, crearás un índice particionado y revisarás sus propiedades desde system:indexes y la Web Console. Finalmente, validarás características de Enterprise Edition dentro del entorno actual de nodo único.
slug: lab4
lab_number: 4
final_result: >
  Al finalizar la práctica habrás creado y validado distintos tipos de índices sobre las collections airline, airport y route del bucket travel-sample. Podrás identificar su definición, estado y propósito desde system:indexes y la Web Console, comprenderás el flujo de construcción diferida y reconocerás el alcance real de un índice particionado dentro de un clúster de nodo único.
notes:
  - Todos los comandos de terminal deben ejecutarse desde Git Bash dentro de Visual Studio Code.
  - No elimines ni detengas el contenedor couchbase-lab al finalizar la práctica.
  - Los índices creados en esta práctica utilizan el prefijo idx_lab4_ para evitar conflictos con índices de prácticas anteriores.
  - El entorno utiliza Couchbase Server Enterprise Edition en un nodo único. Plasma y los índices particionados pueden validarse directamente; las réplicas, el movimiento y la distribución física de índices se revisan de forma conceptual porque requieren varios nodos del Index Service.
references: []
prev: /lab3/lab3/
next: /lab5/lab5/
---

---
<!-- Aquí comienzan las instrucciones paso a paso de la práctica -->

## 📁 Preparación del directorio de trabajo

En esta práctica conservarás el directorio raíz creado anteriormente y únicamente crearás el subdirectorio correspondiente a `lab4`.

### 🗂️ Crear y abrir el subdirectorio de la práctica

- {% include step_label.html %} Abre **Docker Desktop** y espera a que el motor esté en ejecución.
- {% include step_label.html %} Abre **Visual Studio Code**.
- {% include step_label.html %} En VS Code, abre el directorio raíz del curso:

  ```text
  C:\LABS\couchbase-nosql
  ```

- {% include step_label.html %} Abre una terminal integrada desde **Terminal → New Terminal**.
- {% include step_label.html %} Verifica que el perfil seleccionado sea **Git Bash**.
- {% include step_label.html %} Crea el subdirectorio de la práctica:

  ```bash
  mkdir -p /c/LABS/couchbase-nosql/lab4
  ```

- {% include step_label.html %} Cambia al subdirectorio:

  ```bash
  cd /c/LABS/couchbase-nosql/lab4
  ```

- {% include step_label.html %} Confirma tu ubicación:

  ```bash
  pwd
  ```

**Salida esperada:**

```text
/c/LABS/couchbase-nosql/lab4
```

> **IMPORTANTE:** Conserva el directorio raíz `C:\LABS\couchbase-nosql`. En las prácticas siguientes solo crearás un nuevo subdirectorio dentro de esta ubicación.
{: .lab-note .important .compact}

---

## 🧭 Tarea 1. Validar Couchbase e inventariar índices existentes

En esta tarea confirmarás que el entorno continúa disponible y consultarás `system:indexes` para conocer el estado inicial de los índices.

### Tarea 1.1. Verificar que el contenedor está activo

- {% include step_label.html %} Ejecuta:

  {%raw%}
  ```bash
  docker ps --filter "name=couchbase-lab" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
  ```
  {%endraw%}

**Salida esperada:**

```text
NAMES           STATUS
couchbase-lab   Up ...
```

- {% include step_label.html %} Si no aparece activo, inícialo:

  ```bash
  docker start couchbase-lab
  ```

- {% include step_label.html %} Valida la Web Console:

  ```bash
  curl -s -o /dev/null -w "Web Console: HTTP %{http_code}\n" \
    http://localhost:8091/ui/index.html
  ```

**Salida esperada:**

```text
Web Console: HTTP 200
```

### Tarea 1.2. Validar que el contenedor utiliza Enterprise Edition

- {% include step_label.html %} Ejecuta:

  {%raw%}
  ```bash
  docker inspect couchbase-lab --format "Imagen activa: {{.Config.Image}}"
  ```
  {%endraw%}

**Salida esperada:**

```text
Imagen activa: couchbase/server:enterprise-7.6.2
```

> **IMPORTANTE:** Si aparece `couchbase/server:community-7.6.2`, el contenedor anterior de Community Edition continúa activo. Debes volver a la Práctica 2 y recrearlo con la imagen Enterprise antes de continuar, porque los índices particionados son una capacidad de Enterprise Edition.
{: .lab-note .important .compact}

### Tarea 1.3. Abrir el Query Editor

- {% include step_label.html %} Abre `http://localhost:8091`.
- {% include step_label.html %} Inicia sesión con `Administrator` y `Password123!`.
- {% include step_label.html %} Abre **Query**.

### Tarea 1.4. Inventariar los índices existentes

- {% include step_label.html %} Ejecuta:

  ```sql
  SELECT name,
         bucket_id,
         scope_id,
         keyspace_id,
         state,
         is_primary,
         `index_key`,
         condition
  FROM system:indexes
  WHERE bucket_id = "travel-sample"
    AND scope_id = "inventory"
  ORDER BY keyspace_id, name;
  ```

**Resultado esperado:** una lista de índices de las collections del scope `inventory`.

### Tarea 1.5. Resumir índices por collection

- {% include step_label.html %} Ejecuta:

  ```sql
  SELECT keyspace_id AS collection_name,
         COUNT(*) AS total_indices,
         ARRAY_AGG(name) AS nombres
  FROM system:indexes
  WHERE bucket_id = "travel-sample"
    AND scope_id = "inventory"
  GROUP BY keyspace_id
  ORDER BY keyspace_id;
  ```

**Validación:** deben aparecer collections como `airline`, `airport`, `hotel` y `route`.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

{% include support-prompt.html task="tarea1" %}

---

## 🧱 Tarea 2. Crear índices simple, compuesto, parcial y de arreglo

En esta tarea crearás cuatro tipos de índices con nombres exclusivos para la práctica 4.

### Tarea 2.1. Crear un índice secundario simple

- {% include step_label.html %} Ejecuta:

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab4_airline_country
  ON `travel-sample`.inventory.airline(country);
  ```

- {% include step_label.html %} Valida:

  ```sql
  SELECT name, state, `index_key`
  FROM system:indexes
  WHERE name = "idx_lab4_airline_country";
  ```

- {% include step_label.html %} Prueba:

  ```sql
  SELECT name, iata, country
  FROM `travel-sample`.inventory.airline
  WHERE country = "United States"
  ORDER BY name
  LIMIT 10;
  ```

### Tarea 2.2. Crear un índice compuesto

- {% include step_label.html %} Ejecuta:

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab4_airline_country_callsign
  ON `travel-sample`.inventory.airline(country, callsign);
  ```

- {% include step_label.html %} Prueba:

  ```sql
  SELECT name, country, callsign
  FROM `travel-sample`.inventory.airline
  WHERE country = "United States"
    AND callsign IS NOT MISSING
    AND callsign IS NOT NULL
  ORDER BY callsign
  LIMIT 10;
  ```

### Tarea 2.3. Crear un índice parcial

- {% include step_label.html %} Ejecuta:

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab4_airline_us_name
  ON `travel-sample`.inventory.airline(name)
  WHERE country = "United States";
  ```

- {% include step_label.html %} Prueba:

  ```sql
  SELECT name, iata, country
  FROM `travel-sample`.inventory.airline
  WHERE country = "United States"
    AND name LIKE "A%"
  ORDER BY name
  LIMIT 10;
  ```

### Tarea 2.4. Crear un índice de arreglo

- {% include step_label.html %} Ejecuta:

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab4_route_schedule_day
  ON `travel-sample`.inventory.route(
    DISTINCT ARRAY s.day FOR s IN schedule END
  );
  ```

- {% include step_label.html %} Prueba:

  ```sql
  SELECT sourceairport, destinationairport, airline
  FROM `travel-sample`.inventory.route
  WHERE ANY s IN schedule SATISFIES s.day = 1 END
  LIMIT 10;
  ```

### Tarea 2.5. Validar los cuatro índices

- {% include step_label.html %} Ejecuta:

  ```sql
  SELECT name,
         keyspace_id AS collection_name,
         state,
         `index_key`,
         condition
  FROM system:indexes
  WHERE bucket_id = "travel-sample"
    AND scope_id = "inventory"
    AND name IN [
      "idx_lab4_airline_country",
      "idx_lab4_airline_country_callsign",
      "idx_lab4_airline_us_name",
      "idx_lab4_route_schedule_day"
    ]
  ORDER BY name;
  ```

**Resultado esperado:** 4 filas con estado `online`.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

{% include support-prompt.html task="tarea2" %}

---

## ⏳ Tarea 3. Crear y construir un índice diferido

En esta tarea crearás un índice con construcción diferida y observarás su cambio de estado.

### Tarea 3.1. Crear el índice sin construirlo

- {% include step_label.html %} Ejecuta:

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab4_airport_city_deferred
  ON `travel-sample`.inventory.airport(city, country)
  WITH {"defer_build": true};
  ```

### Tarea 3.2. Verificar el estado deferred

- {% include step_label.html %} Ejecuta:

  ```sql
  SELECT name, state, `index_key`
  FROM system:indexes
  WHERE name = "idx_lab4_airport_city_deferred";
  ```

**Salida esperada:** `state = "deferred"`.

### Tarea 3.3. Construir el índice

- {% include step_label.html %} Ejecuta:

  ```sql
  BUILD INDEX ON `travel-sample`.inventory.airport(
    idx_lab4_airport_city_deferred
  );
  ```

### Tarea 3.4. Observar la transición de estados

- {% include step_label.html %} Ejecuta varias veces:

  ```sql
  SELECT name, state
  FROM system:indexes
  WHERE name = "idx_lab4_airport_city_deferred";
  ```

**Estados esperados:**

```text
deferred → building → online
```

> **NOTA:** En un dataset pequeño, `building` puede durar muy poco y podrías observar directamente `online`.
{: .lab-note .info .compact}

### Tarea 3.5. Probar el índice

- {% include step_label.html %} Ejecuta:

  ```sql
  SELECT airportname, city, country
  FROM `travel-sample`.inventory.airport
  WHERE city = "San Francisco"
    AND country = "United States"
  ORDER BY airportname
  LIMIT 10;
  ```

**Validación:** el índice debe estar `online` y la consulta debe ejecutarse sin errores.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

{% include support-prompt.html task="tarea3" %}

---

## 🧩 Tarea 4. Crear y validar un índice particionado

En esta tarea crearás un índice particionado por hash sobre la collection `route`.

### Tarea 4.1. Crear el índice particionado

- {% include step_label.html %} Ejecuta:

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab4_route_source_partitioned
  ON `travel-sample`.inventory.route(
    sourceairport,
    destinationairport
  )
  PARTITION BY HASH(sourceairport)
  WITH {
    "defer_build": false,
    "num_partition": 8
  };
  ```

### Tarea 4.2. Validar su definición

- {% include step_label.html %} Ejecuta:

  ```sql
  SELECT
      name,
      state,
      `partition`,
      `index_key`
  FROM system:indexes
  WHERE name = "idx_lab4_route_source_partitioned";
  ```

**Validación:** el índice debe aparecer `online` y el campo `partition` debe indicar particionado por hash.

### Tarea 4.3. Probar una consulta compatible

- {% include step_label.html %} Ejecuta:

  ```sql
  SELECT sourceairport, destinationairport, airline, distance
  FROM `travel-sample`.inventory.route
  WHERE sourceairport = "SFO"
  ORDER BY destinationairport
  LIMIT 10;
  ```

### Tarea 4.4. Revisar el plan con EXPLAIN

- {% include step_label.html %} Ejecuta:

  ```sql
  EXPLAIN
  SELECT sourceairport, destinationairport
  FROM `travel-sample`.inventory.route
  WHERE sourceairport = "SFO"
  ORDER BY destinationairport
  LIMIT 10;
  ```

- {% include step_label.html %} Busca en el resultado:

  ```text
  idx_lab4_route_source_partitioned
  ```

> **NOTA:** Couchbase puede elegir otro índice compatible si estima que es más conveniente. Confirma primero que el índice particionado esté `online`.
{: .lab-note .info .compact}

### Tarea 4.5. Comprender la limitación del nodo único

```text
Clúster actual:
1 nodo Couchbase
1 Index Service
8 particiones lógicas
```

En este laboratorio todas las particiones residen dentro del mismo nodo. En un clúster multi-nodo podrían distribuirse físicamente entre varios nodos del Index Service.

> **IMPORTANTE:** El índice está particionado lógicamente, pero el entorno actual no demuestra distribución física ni alta disponibilidad.
{: .lab-note .important .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

{% include support-prompt.html task="tarea4" %}

---

## 📈 Tarea 5. Explorar índices y validar Enterprise Edition

En esta tarea revisarás los índices desde la Web Console, generarás actividad y validarás las capacidades de indexación disponibles en Enterprise Edition.

### Tarea 5.1. Abrir la sección Indexes

- {% include step_label.html %} En la Web Console, abre **Indexes**.
- {% include step_label.html %} Localiza los índices con prefijo `idx_lab4_`.
- {% include step_label.html %} Identifica nombre, keyspace, estado, campos indexados, condición parcial, nodo y métricas visibles.

> **NOTA:** Las columnas exactas pueden variar según la edición y versión.
{: .lab-note .info .compact}

### Tarea 5.2. Generar uso sobre un índice

- {% include step_label.html %} Regresa a **Query** y ejecuta entre 3 y 5 veces:

  ```sql
  SELECT name, country, callsign
  FROM `travel-sample`.inventory.airline
  WHERE country = "United States"
  ORDER BY callsign
  LIMIT 20;
  ```

- {% include step_label.html %} Regresa a **Indexes** y revisa si las métricas reflejan nuevas solicitudes o actividad.

### Tarea 5.3. Consultar la configuración del Index Service

- {% include step_label.html %} En Git Bash, ejecuta:

  ```bash
  curl -s -u Administrator:Password123! \
    http://localhost:8091/settings/indexes \
    | python -m json.tool
  ```

**Salida esperada:** un objeto JSON con la configuración del Index Service. En Enterprise Edition, el modo de almacenamiento estándar debe indicar `plasma`.

### Tarea 5.4. Validar capacidades de Enterprise Edition

| Capacidad | Estado en el laboratorio |
|---|---|
| Índices secundarios | Disponible y validado |
| Índices compuestos | Disponible y validado |
| Índices parciales | Disponible y validado |
| Índices de arreglos | Disponible y validado |
| Build diferido | Disponible y validado |
| Índices particionados | Disponible; las 8 particiones permanecen en el nodo único |
| Réplicas de índices | Disponible en Enterprise, pero requiere varios nodos para demostrar alta disponibilidad |
| Movimiento de índices | Disponible en Enterprise, pero requiere varios nodos del Index Service |
| Plasma | Disponible y verificable mediante la configuración del Index Service |
| Memory Optimized Indexes | Disponible en Enterprise, pero no se cambia en esta práctica |

### Tarea 5.5. Consolidar los índices creados

- {% include step_label.html %} Ejecuta:

  ```sql
  SELECT
      name,
      keyspace_id AS collection_name,
      state,
      `index_key`,
      condition,
      `partition`
  FROM system:indexes
  WHERE bucket_id = "travel-sample"
    AND scope_id = "inventory"
    AND name LIKE "idx_lab4_%"
  ORDER BY collection_name, name;
  ```

**Resultado esperado:** 6 índices, todos en estado `online`.

### Tarea 5.6. Ejecutar validación final desde Git Bash

- {% include step_label.html %} Ejecuta:

  ```bash
  curl -s -u Administrator:Password123! \
    http://localhost:8093/query/service \
    --data-urlencode 'statement=SELECT name, keyspace_id, state FROM system:indexes WHERE bucket_id = "travel-sample" AND scope_id = "inventory" AND name LIKE "idx_lab4_%" ORDER BY name' \
    | python -m json.tool | grep -E '"name"|"keyspace_id"|"state"|"status"'
  ```

**Salida esperada:** nombres de los índices, `state: online` y `status: success`.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

{% include support-prompt.html task="tarea5" %}

---

## 🛠️ Resolución de problemas

### Problema 1. El índice ya existe

Verifica su definición:

```sql
SELECT name, keyspace_id, `index_key`, condition, state
FROM system:indexes
WHERE name LIKE "idx_lab4_%"
ORDER BY name;
```

### Problema 2. El índice permanece en deferred

```sql
BUILD INDEX ON `travel-sample`.inventory.airport(
  idx_lab4_airport_city_deferred
);
```

### Problema 3. El índice permanece en building

Espera unos segundos y revisa nuevamente:

```sql
SELECT name, state
FROM system:indexes
WHERE name LIKE "idx_lab4_%"
ORDER BY name;
```

### Problema 4. EXPLAIN no usa el índice esperado

Verifica que esté `online`:

```sql
SELECT name, state, `index_key`
FROM system:indexes
WHERE name = "idx_lab4_route_source_partitioned";
```

Después prueba:

```sql
EXPLAIN
SELECT sourceairport, destinationairport
FROM `travel-sample`.inventory.route
WHERE sourceairport = "SFO";
```

### Problema 5. La creación del índice particionado indica que es una función Enterprise

**Síntoma:** aparece un mensaje similar a `Index Partitioning is not supported in non-Enterprise Edition`.

**Causa probable:** el contenedor activo todavía fue creado con la imagen Community Edition.

**Solución:**

{%raw%}
```bash
docker inspect couchbase-lab   --format "Imagen activa: {{.Config.Image}}"
```
{%endraw%}

La salida debe ser:

```text
Imagen activa: couchbase/server:enterprise-7.6.2
```

Si aparece la imagen Community, vuelve a la Práctica 2 y recrea el contenedor con Enterprise Edition. Después repite la configuración del clúster y la carga de `travel-sample`.

### Problema 6. python -m json.tool falla

```bash
python --version
python3 --version
```

Si `python3` funciona, reemplaza `python -m json.tool` por `python3 -m json.tool`.