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

- {% include step_label.html %} Abre **Docker Desktop** y confirma que el motor indique estado activo, porque `couchbase-lab` depende del daemon local para ejecutar todos sus servicios.
- {% include step_label.html %} Abre **Visual Studio Code** y espera su carga completa, ya que utilizarás el Explorador y la terminal integrada durante toda la práctica.
- {% include step_label.html %} Abre `C:\LABS\couchbase-nosql` en Visual Studio Code para mantener visibles los archivos del curso y trabajar desde la ruta esperada.

  ```text
  C:\LABS\couchbase-nosql
  ```

- {% include step_label.html %} Selecciona **Terminal → New Terminal** en Visual Studio Code para abrir la consola integrada desde la que ejecutarás las operaciones de la práctica.
- {% include step_label.html %} Comprueba en el selector del panel Terminal que **Git Bash** sea el perfil activo, porque los comandos utilizan sintaxis y rutas compatibles con Bash.
- {% include step_label.html %} Crea el subdirectorio de la práctica desde Git Bash para crear de forma idempotente el directorio `/c/LABS/couchbase-nosql/lab4` donde se organizarán los archivos de esta práctica.

  ```bash
  mkdir -p /c/LABS/couchbase-nosql/lab4
  ```

- {% include step_label.html %} Cambia al subdirectorio desde Git Bash para cambiar la ubicación activa a `/c/LABS/couchbase-nosql/lab4` y evitar operaciones posteriores desde un directorio incorrecto.

  ```bash
  cd /c/LABS/couchbase-nosql/lab4
  ```

- {% include step_label.html %} Confirma tu ubicación desde Git Bash para mostrar la ruta activa y confirmar que Git Bash está ubicado en el subdirectorio asignado a esta práctica.

  ```bash
  pwd
  ```

**Salida esperada:**

Para validar `crear y abrir el subdirectorio de trabajo`, verifica la referencia siguiente y confirma que la respuesta permita mostrar la ruta activa y confirmar que Git Bash está ubicado en el subdirectorio asignado a esta práctica; detente si aparece un error.

```text
/c/LABS/couchbase-nosql/lab4
```

> **IMPORTANTE:** Conserva el directorio raíz `C:\LABS\couchbase-nosql`. En las prácticas siguientes solo crearás un nuevo subdirectorio dentro de esta ubicación.
{: .lab-note .important .compact}

---

## 🧭 Tarea 1. Validar Couchbase e inventariar índices existentes

En esta tarea confirmarás que el entorno continúa disponible y consultarás `system:indexes` para conocer el estado inicial de los índices.

### Tarea 1.1. Verificar que el contenedor está activo

- {% include step_label.html %} Consulta en Git Bash el estado de `couchbase-lab` y confirma que aparezca activo antes de utilizar Web Console, Query e Index Service.

  {%raw%}
  ```bash
  docker ps --filter "name=couchbase-lab" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
  ```
  {%endraw%}

**Salida esperada:**

Para validar `Verificar que el contenedor está activo`, verifica la referencia siguiente y confirma que la respuesta permita consultar los contenedores y comprobar que `couchbase-lab` permanezca activo antes de utilizar sus servicios; detente si aparece un error.

```text
NAMES           STATUS
couchbase-lab   Up ...
```

- {% include step_label.html %} Inicia `couchbase-lab` solamente si está detenido y espera la confirmación de Docker antes de acceder a los servicios del clúster.

  ```bash
  docker start couchbase-lab
  ```

- {% include step_label.html %} Solicita la Web Console por el puerto 8091 y confirma HTTP 200 como evidencia de que el servicio administrativo está disponible.

  ```bash
  curl -s -o /dev/null -w "Web Console: HTTP %{http_code}\n" \
    http://localhost:8091/ui/index.html
  ```

**Salida esperada:**

Para validar `Verificar que el contenedor está activo`, verifica la referencia siguiente y confirma que la respuesta permita solicitar la Web Console por el puerto 8091 e interpretar el código HTTP como prueba de disponibilidad; detente si aparece un error.

```text
Web Console: HTTP 200
```

### Tarea 1.2. Validar que el contenedor utiliza Enterprise Edition

- {% include step_label.html %} Inspecciona la imagen configurada en `couchbase-lab` y verifica que sea exactamente `couchbase/server:enterprise-7.6.2`.

  {%raw%}
  ```bash
  docker inspect couchbase-lab --format "Imagen activa: {{.Config.Image}}"
  ```
  {%endraw%}

**Salida esperada:**

Para validar `Validar que el contenedor utiliza Enterprise Edition`, verifica la referencia siguiente y confirma que la respuesta permita consultar la configuración de `couchbase-lab` y verificar que utiliza la imagen Enterprise 7.6.2 requerida; detente si aparece un error.

```text
Imagen activa: couchbase/server:enterprise-7.6.2
```

> **IMPORTANTE:** Si aparece `couchbase/server:community-7.6.2`, el contenedor anterior de Community Edition continúa activo. Debes volver a la Práctica 2 y recrearlo con la imagen Enterprise antes de continuar, porque los índices particionados son una capacidad de Enterprise Edition.
{: .lab-note .important .compact}

### Tarea 1.3. Abrir el Query Editor

- {% include step_label.html %} Abre `http://localhost:8091` en el navegador para ingresar a Web Console y utilizar Query Workbench durante la práctica.
- {% include step_label.html %} Inicia sesión en la Web Console con `Administrator` y `Password123!` para acceder al clúster mediante las credenciales administrativas definidas.
- {% include step_label.html %} Selecciona **Query** en la navegación lateral de la Web Console para abrir Query Workbench y ejecutar las sentencias SQL++ de la práctica.

### Tarea 1.4. Inventariar los índices existentes

- {% include step_label.html %} Consulta `system:indexes` en Query Workbench para inventariar nombre, keyspace, claves, condición y estado de los índices existentes.

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

Para validar `Inventariar los índices existentes`, una lista de índices de las collections del scope `inventory`. y confirma que la respuesta permita inventariar en `system:indexes` los índices del bucket, scope y collections especificados por los filtros; detente si aparece un error.

### Tarea 1.5. Resumir índices por collection

- {% include step_label.html %} Agrupa `system:indexes` por colección para contar los índices y reunir sus nombres antes de crear las definiciones de esta práctica.

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

Para validar `Resumir índices por collection`, deben aparecer collections como `airline`, `airport`, `hotel` y `route`. y confirma que la respuesta permita agrupar los registros de `system:indexes` por collection y comparar la cantidad y los nombres disponibles; detente si aparece un error.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

{% include support-prompt.html task="tarea1" %}

---

## 🧱 Tarea 2. Crear índices simple, compuesto, parcial y de arreglo

En esta tarea crearás cuatro tipos de índices con nombres exclusivos para la práctica 4.

### Tarea 2.1. Crear un índice secundario simple

- {% include step_label.html %} Crea `idx_lab4_airline_country` sobre `airline.country` para soportar búsquedas exactas de aerolíneas filtradas por país.

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab4_airline_country
  ON `travel-sample`.inventory.airline(country);
  ```

- {% include step_label.html %} Consulta `system:indexes` y confirma que `idx_lab4_airline_country` tenga la clave esperada y alcance el estado `online`.

  ```sql
  SELECT name, state, `index_key`
  FROM system:indexes
  WHERE name = "idx_lab4_airline_country";
  ```

- {% include step_label.html %} Consulta aerolíneas de `United States` para comprobar que el índice simple atienda el predicado exacto definido sobre `country`.

  ```sql
  SELECT name, iata, country
  FROM `travel-sample`.inventory.airline
  WHERE country = "United States"
  ORDER BY name
  LIMIT 10;
  ```

### Tarea 2.2. Crear un índice compuesto

- {% include step_label.html %} Crea `idx_lab4_airline_country_callsign` con claves ordenadas para cubrir consultas que filtren país y distintivo de llamada.

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab4_airline_country_callsign
  ON `travel-sample`.inventory.airline(country, callsign);
  ```

- {% include step_label.html %} Consulta `name`, `country` y `callsign` con los predicados establecidos para validar el uso funcional del índice compuesto.

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

- {% include step_label.html %} Crea `idx_lab4_airline_us_name` con condición parcial para indexar nombres únicamente de aerolíneas de `United States`.

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab4_airline_us_name
  ON `travel-sample`.inventory.airline(name)
  WHERE country = "United States";
  ```

- {% include step_label.html %} Consulta aerolíneas estadounidenses cuyo nombre comienza con `A` para verificar que el predicado satisface la condición parcial.

  ```sql
  SELECT name, iata, country
  FROM `travel-sample`.inventory.airline
  WHERE country = "United States"
    AND name LIKE "A%"
  ORDER BY name
  LIMIT 10;
  ```

### Tarea 2.4. Crear un índice de arreglo

- {% include step_label.html %} Crea `idx_lab4_route_schedule_day` con una clave de arreglo para indexar individualmente los valores `day` incluidos en `schedule`.

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab4_route_schedule_day
  ON `travel-sample`.inventory.route(
    DISTINCT ARRAY s.day FOR s IN schedule END
  );
  ```

- {% include step_label.html %} Consulta rutas con alguna salida en el día 1 mediante `ANY ... SATISFIES` para validar la correspondencia con la clave de arreglo.

  ```sql
  SELECT sourceairport, destinationairport, airline
  FROM `travel-sample`.inventory.route
  WHERE ANY s IN schedule SATISFIES s.day = 1 END
  LIMIT 10;
  ```

### Tarea 2.5. Validar los cuatro índices

- {% include step_label.html %} Consulta conjuntamente los cuatro índices en `system:indexes` y confirma claves, condiciones, keyspaces y estado `online`.

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

Para validar `Validar los cuatro índices`, 4 filas con estado `online`. y confirma que la respuesta permita consultar `system:indexes` y verificar conjuntamente los índices enumerados en el filtro `name IN`; detente si aparece un error.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

{% include support-prompt.html task="tarea2" %}

---

## ⏳ Tarea 3. Crear y construir un índice diferido

En esta tarea crearás un índice con construcción diferida y observarás su cambio de estado.

### Tarea 3.1. Crear el índice sin construirlo

- {% include step_label.html %} Crea `idx_lab4_airport_city_deferred` con `defer_build` para separar el registro de la definición y la construcción física.

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab4_airport_city_deferred
  ON `travel-sample`.inventory.airport(city, country)
  WITH {"defer_build": true};
  ```

### Tarea 3.2. Verificar el estado deferred

- {% include step_label.html %} Consulta `system:indexes` y verifica que `idx_lab4_airport_city_deferred` figure registrado con estado `deferred`.

  ```sql
  SELECT name, state, `index_key`
  FROM system:indexes
  WHERE name = "idx_lab4_airport_city_deferred";
  ```

**Salida esperada:** `state = "deferred"`.

Para validar `Verificar el estado deferred`, `state = "deferred"`. y confirma que la respuesta permita consultar `system:indexes` y confirmar la definición y el estado del índice `idx_lab4_airport_city_deferred`; detente si aparece un error.

### Tarea 3.3. Construir el índice

- {% include step_label.html %} Inicia con `BUILD INDEX` la construcción física del índice diferido para que Index Service procese los documentos del keyspace.

  ```sql
  BUILD INDEX ON `travel-sample`.inventory.airport(
    idx_lab4_airport_city_deferred
  );
  ```

### Tarea 3.4. Observar la transición de estados

- {% include step_label.html %} Consulta periódicamente `system:indexes` y observa la transición de `deferred` o `building` hasta alcanzar el estado `online`.

  ```sql
  SELECT name, state
  FROM system:indexes
  WHERE name = "idx_lab4_airport_city_deferred";
  ```

**Estados esperados:**

Para validar `Observar la transición de estados`, verifica la referencia siguiente y confirma que la respuesta permita consultar `system:indexes` y confirmar la definición y el estado del índice `idx_lab4_airport_city_deferred`; detente si aparece un error.

```text
deferred → building → online
```

> **NOTA:** En un dataset pequeño, `building` puede durar muy poco y podrías observar directamente `online`.
{: .lab-note .info .compact}

### Tarea 3.5. Probar el índice

- {% include step_label.html %} Consulta aeropuertos de San Francisco en Estados Unidos para comprobar el índice diferido después de quedar disponible.

  ```sql
  SELECT airportname, city, country
  FROM `travel-sample`.inventory.airport
  WHERE city = "San Francisco"
    AND country = "United States"
  ORDER BY airportname
  LIMIT 10;
  ```

**Validación:** el índice debe estar `online` y la consulta debe ejecutarse sin errores.

Para validar `Probar el índice`, el índice debe estar `online` y la consulta debe ejecutarse sin errores. y confirma que la respuesta permita consultar ``travel-sample`.inventory.airport` con el filtro `city = "San Francisco" AND country = "United States"` y comprobar las filas y el orden obtenidos; detente si aparece un error.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

{% include support-prompt.html task="tarea3" %}

---

## 🧩 Tarea 4. Crear y validar un índice particionado

En esta tarea crearás un índice particionado por hash sobre la collection `route`.

### Tarea 4.1. Crear el índice particionado

- {% include step_label.html %} Crea `idx_lab4_route_source_partitioned` con partición HASH sobre `sourceairport` para estudiar su definición en Enterprise Edition.

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

- {% include step_label.html %} Consulta `system:indexes` y confirma las claves, la expresión de partición y el estado del índice particionado recién creado.

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

Para validar `Validar su definición`, el índice debe aparecer `online` y el campo `partition` debe indicar particionado por hash. y confirma que la respuesta permita consultar `system:indexes` y confirmar la definición y el estado del índice `idx_lab4_route_source_partitioned`; detente si aparece un error.

### Tarea 4.3. Probar una consulta compatible

- {% include step_label.html %} Consulta las rutas cuyo aeropuerto de origen es `SFO` para probar un predicado compatible con la clave del índice particionado.

  ```sql
  SELECT sourceairport, destinationairport, airline, distance
  FROM `travel-sample`.inventory.route
  WHERE sourceairport = "SFO"
  ORDER BY destinationairport
  LIMIT 10;
  ```

### Tarea 4.4. Revisar el plan con EXPLAIN

- {% include step_label.html %} Obtén el plan con `EXPLAIN` e identifica el índice seleccionado, sus spans y los operadores empleados por Query Service.

  ```sql
  EXPLAIN
  SELECT sourceairport, destinationairport
  FROM `travel-sample`.inventory.route
  WHERE sourceairport = "SFO"
  ORDER BY destinationairport
  LIMIT 10;
  ```

- {% include step_label.html %} Localiza `idx_lab4_route_source_partitioned` en el plan y confirma que Query Service eligió la definición esperada para el filtro.

  ```text
  idx_lab4_route_source_partitioned
  ```

> **NOTA:** Couchbase puede elegir otro índice compatible si estima que es más conveniente. Confirma primero que el índice particionado esté `online`.
{: .lab-note .info .compact}

### Tarea 4.5. Comprender la limitación del nodo único

- {% include step_label.html %} Registra que la partición se observa lógicamente, pero un clúster de un nodo no permite evaluar distribución ni tolerancia a fallos.

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

- {% include step_label.html %} Selecciona **Indexes** en la navegación lateral de la Web Console para examinar la definición, el estado y las métricas del Index Service.
- {% include step_label.html %} En **Indexes**, filtra o recorre la lista hasta localizar los nombres con prefijo `idx_lab4_` y confirma que correspondan con esta práctica.
- {% include step_label.html %} En la tabla de **Indexes**, revisa nombre, keyspace, estado, campos, condición, nodo y métricas para interpretar cada índice creado.

> **NOTA:** Las columnas exactas pueden variar según la edición y versión.
{: .lab-note .info .compact}

### Tarea 5.2. Generar uso sobre un índice

- {% include step_label.html %} Regresa a **Query**, ejecuta la consulta entre tres y cinco veces y espera cada resultado para generar actividad medible sobre el índice asociado.

  ```sql
  SELECT name, country, callsign
  FROM `travel-sample`.inventory.airline
  WHERE country = "United States"
  ORDER BY callsign
  LIMIT 20;
  ```

- {% include step_label.html %} Regresa a **Indexes** y comprueba si las métricas de los índices reflejan solicitudes nuevas después de ejecutar repetidamente la consulta.

### Tarea 5.3. Consultar la configuración del Index Service

- {% include step_label.html %} Consulta desde Git Bash la configuración de Index Service y revisa en el JSON el modo de almacenamiento activo del nodo.

  ```bash
  curl -s -u Administrator:Password123! \
    http://localhost:8091/settings/indexes \
    | python -m json.tool
  ```

**Salida esperada:** un objeto JSON con la configuración del Index Service. En Enterprise Edition, el modo de almacenamiento estándar debe indicar `plasma`.

### Tarea 5.4. Validar capacidades de Enterprise Edition

- {% include step_label.html %} Relaciona índices particionados, réplicas y distribución con Enterprise Edition, diferenciando capacidades disponibles de las probadas en un nodo.

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

- {% include step_label.html %} Consulta en `system:indexes` todas las definiciones `idx_lab4_%` y consolida keyspace, claves, condición, partición y estado.

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

Para validar `Consolidar los índices creados`, 6 índices, todos en estado `online`. y confirma que la respuesta permita consultar `system:indexes` y revisar los índices cuyo nombre coincide con `idx_lab4_%`; detente si aparece un error.

### Tarea 5.6. Ejecutar validación final desde Git Bash

- {% include step_label.html %} Envía por REST una sentencia SQL++ a Query Service y confirma `status: success` junto con los índices `idx_lab4_%` esperados.

  ```bash
  curl -s -u Administrator:Password123! \
    http://localhost:8093/query/service \
    --data-urlencode 'statement=SELECT name, keyspace_id, state FROM system:indexes WHERE bucket_id = "travel-sample" AND scope_id = "inventory" AND name LIKE "idx_lab4_%" ORDER BY name' \
    | python -m json.tool | grep -E '"name"|"keyspace_id"|"state"|"status"'
  ```

**Salida esperada:** nombres de los índices, `state: online` y `status: success`.

Para validar `Ejecutar validación final desde Git Bash`, nombres de los índices, `state: online` y `status: success`. y confirma que la respuesta permita enviar una sentencia SQL++ al servicio Query del puerto 8093 y comprobar el estado incluido en la respuesta JSON; detente si aparece un error.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

{% include support-prompt.html task="tarea5" %}

---

## 🛠️ Resolución de problemas

### Problema 1. El índice ya existe

Verifica su definición:

- {% include step_label.html %} Consulta los índices `idx_lab4_%` para distinguir una definición ya existente y decidir si puede reutilizarse o requiere otro nombre.

```sql
SELECT name, keyspace_id, `index_key`, condition, state
FROM system:indexes
WHERE name LIKE "idx_lab4_%"
ORDER BY name;
```

### Problema 2. El índice permanece en deferred

- {% include step_label.html %} Repite `BUILD INDEX` sobre `idx_lab4_airport_city_deferred` cuando continúe diferido y verifica que comience la construcción.

```sql
BUILD INDEX ON `travel-sample`.inventory.airport(
  idx_lab4_airport_city_deferred
);
```

### Problema 3. El índice permanece en building

Espera unos segundos y revisa nuevamente:

- {% include step_label.html %} Consulta nuevamente los estados `idx_lab4_%` después de esperar para confirmar si Index Service completó la construcción pendiente.

```sql
SELECT name, state
FROM system:indexes
WHERE name LIKE "idx_lab4_%"
ORDER BY name;
```

### Problema 4. EXPLAIN no usa el índice esperado

Verifica que esté `online`:

- {% include step_label.html %} Verifica que `idx_lab4_route_source_partitioned` esté `online` y que su clave corresponda al predicado evaluado por `EXPLAIN`.

```sql
SELECT name, state, `index_key`
FROM system:indexes
WHERE name = "idx_lab4_route_source_partitioned";
```

Después prueba:

- {% include step_label.html %} Genera nuevamente el plan para `sourceairport = "SFO"` y revisa qué índice, spans y operadores selecciona el optimizador.

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

- {% include step_label.html %} Inspecciona la imagen del contenedor si falla el índice particionado y confirma que corresponda a Couchbase Enterprise 7.6.2.

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

- {% include step_label.html %} Consulta las versiones de `python` y `python3` disponibles en Git Bash para elegir el intérprete capaz de ejecutar el módulo `json.tool`.

```bash
python --version
python3 --version
```

Si `python3` funciona, reemplaza `python -m json.tool` por `python3 -m json.tool`.