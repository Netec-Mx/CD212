---
layout: lab
title: "Práctica 14: Ejecución de consultas analíticas con SQL++ Analytics"
permalink: /lab14/lab14/
images_base: /labs/lab14/img
duration: "50 minutos"
objective:
  - Verificar que Couchbase Server Enterprise 7.6.2 tenga disponible el servicio Analytics y el puerto 8095.
  - Crear el namespace analítico TravelAnalytics y tres Analytics collections sincronizadas desde travel-sample.
  - Ejecutar consultas analíticas con agregaciones, UNNEST y JOIN sin utilizar el servicio Query.
  - Interpretar respuestas exitosas y fallidas del Analytics REST API.
  - Automatizar una consulta analítica mediante Bash y validar los recursos creados.
prerequisites:
  - Haber completado las prácticas anteriores de SQL++ y Couchbase Search.
  - Tener Couchbase Server Enterprise 7.6.2 en ejecución mediante la imagen couchbase/server:enterprise-7.6.2.
  - Tener habilitados los servicios Data, Query, Index, Search y Analytics.
  - Tener publicado el puerto 8095 del contenedor hacia el host.
  - Tener cargado el bucket travel-sample con las collections inventory.airport, inventory.route e inventory.hotel.
  - Tener acceso a la Web Console en http://localhost:8091.
  - Utilizar Visual Studio Code con Git Bash como terminal integrada.
introduction:
  - En esta práctica configurarás un entorno Analytics básico sobre Couchbase Server Enterprise 7.6.2. Crearás un Dataverse, también entendido como Analytics scope, definirás tres Analytics collections sincronizadas mediante el Link Local, ejecutarás consultas analíticas con SQL++, interpretarás métricas del servicio y automatizarás una consulta mediante REST API.
slug: lab14
lab_number: 14
final_result: >
  Al finalizar habrás creado TravelAnalytics con las Analytics collections airports, routes y hotels, conectado el Link Local, ejecutado agregaciones, UNNEST y JOIN, interpretado respuestas REST y validado el entorno mediante un script reproducible.
notes:
  - Todos los comandos deben ejecutarse desde Git Bash dentro de Visual Studio Code.
  - La imagen obligatoria para esta práctica es couchbase/server:enterprise-7.6.2.
  - Se utilizarán las credenciales Administrator y Password123! configuradas en prácticas anteriores.
  - Analytics utiliza el puerto 8095 para HTTP y debe estar publicado por Docker.
  - En Couchbase Analytics, el término histórico Dataset también puede aparecer como Analytics collection.
  - En un nodo único existe separación lógica entre servicios, pero todos comparten los recursos físicos del mismo contenedor.
  - Los recursos creados deben conservarse al finalizar para revisión y prácticas posteriores.
references: []
prev: /lab13/lab13/
next: /lab15/lab15/
---

---
<!-- Aquí comienzan las instrucciones paso a paso de la práctica -->

## 📁 Preparación del directorio de trabajo

### Preparar el subdirectorio de la práctica

- {% include step_label.html %} Abre Docker Desktop y espera hasta que el motor termine de iniciar, porque Couchbase Server no podrá responder mientras el contenedor no tenga acceso a los recursos asignados por Docker.

- {% include step_label.html %} Abre Visual Studio Code y selecciona el directorio raíz del curso para conservar una organización uniforme entre todas las prácticas.

  ```text
  C:\LABS\couchbase-nosql
  ```

- {% include step_label.html %} Abre una terminal integrada desde **Terminal → New Terminal** y confirma que el perfil activo sea Git Bash, ya que los comandos utilizan variables de entorno, redirecciones y scripts Bash.

- {% include step_label.html %} Crea el subdirectorio `lab14` sin alterar los archivos de prácticas anteriores.

  ```bash
  mkdir -p /c/LABS/couchbase-nosql/lab14
  ```

- {% include step_label.html %} Cambia al subdirectorio recién creado para que todos los archivos SQL++, JSON, Bash y Markdown permanezcan como evidencia del laboratorio.

  ```bash
  cd /c/LABS/couchbase-nosql/lab14
  ```

- {% include step_label.html %} Confirma la ruta actual antes de continuar.

  ```bash
  pwd
  ```

**Salida esperada:**

```text
/c/LABS/couchbase-nosql/lab14
```

---

## 🧭 Tarea 1. Validar Enterprise 7.6.2 y el servicio Analytics

En esta tarea verificarás la imagen del contenedor, el puerto 8095, la disponibilidad del servicio Analytics y la capacidad real de ejecutar una consulta mínima.

### Tarea 1.1. Definir variables del laboratorio

- {% include step_label.html %} Define variables de entorno para centralizar credenciales y endpoints, evitando repetir valores y reduciendo errores de escritura durante la práctica.

  ```bash
  export CB_HOST="localhost"
  export CB_ADMIN="Administrator"
  export CB_PASS="Password123!"

  export CB_URL="http://${CB_HOST}:8091"
  export CB_QUERY_URL="http://${CB_HOST}:8093/query/service"
  export CB_ANALYTICS_URL="http://${CB_HOST}:8095"
  export CB_ANALYTICS_SERVICE="${CB_ANALYTICS_URL}/analytics/service"

  export ANALYTICS_DATAVERSE="TravelAnalytics"
  ```

- {% include step_label.html %} Verifica que las variables principales hayan quedado definidas correctamente antes de usarlas en comandos REST.

  ```bash
  printf "CB_URL=%s\nCB_ANALYTICS_URL=%s\nDATAVERSE=%s\n" \
    "${CB_URL}" \
    "${CB_ANALYTICS_URL}" \
    "${ANALYTICS_DATAVERSE}"
  ```

### Tarea 1.2. Verificar imagen, contenedor y puerto 8095

- {% include step_label.html %} Comprueba que `couchbase-lab` utilice exactamente la imagen Enterprise 7.6.2 definida para el curso.

  {%raw%}
  ```bash
  docker inspect couchbase-lab \
    --format '{{.Config.Image}}'
  ```
  {%endraw%}

**Salida esperada:**

```text
couchbase/server:enterprise-7.6.2
```

- {% include step_label.html %} Confirma que el contenedor esté activo y revisa simultáneamente su nombre, imagen y estado.

  {%raw%}
  ```bash
  docker ps --filter "name=couchbase-lab" \
    --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
  ```
  {%endraw%}

- {% include step_label.html %} Si el contenedor está detenido, inícialo antes de validar servicios.

  ```bash
  docker start couchbase-lab
  ```

- {% include step_label.html %} Revisa los puertos publicados por Docker y confirma que el puerto 8095 del contenedor esté accesible desde el host.

  ```bash
  docker port couchbase-lab
  ```

**Debes identificar una asociación equivalente a:**

```text
8095/tcp -> 0.0.0.0:8095
```

> Si 8095 no aparece, Analytics puede estar habilitado internamente, pero Git Bash no podrá acceder al servicio desde `localhost:8095`.

### Tarea 1.3. Validar el servicio Analytics con una consulta real

- {% include step_label.html %} Consulta la asignación de servicios del nodo para confirmar que Couchbase reporta los componentes relacionados con Analytics.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${CB_URL}/pools/default/nodeServices" \
    | python -m json.tool \
    | grep -Ei '"cbas"|"cbasAdmin"|"cbasCc"|8095'
  ```

- {% include step_label.html %} Ejecuta una consulta mínima contra el Analytics REST API para comprobar autenticación, endpoint y motor SQL++ en una sola operación.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    "${CB_ANALYTICS_SERVICE}" \
    --data-urlencode 'statement=SELECT VALUE 1;' \
    --data-urlencode 'client_context_id=lab14-healthcheck-001' \
    > analytics-healthcheck-response.json
  ```

- {% include step_label.html %} Formatea la respuesta y confirma que `status` sea `success` y que `results` contenga el valor 1.

  ```bash
  python -m json.tool analytics-healthcheck-response.json
  ```

- {% include step_label.html %} Abre la Web Console en `http://localhost:8091`, inicia sesión y confirma que la opción **Analytics** aparezca en el menú lateral.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

{% include support-prompt.html task="tarea1" %}

---

## 🏗️ Tarea 2. Crear TravelAnalytics y conectar el Link Local

En esta tarea crearás el namespace analítico, tres Analytics collections y el enlace que inicia la sincronización desde las collections operacionales.

### Tarea 2.1. Crear el Dataverse y las Analytics collections

- {% include step_label.html %} Crea `create-analytics-environment.sqlpp` para conservar todas las sentencias de preparación en un único archivo reproducible.

  ```bash
  cat > create-analytics-environment.sqlpp << 'EOF'
  CREATE DATAVERSE TravelAnalytics IF NOT EXISTS;

  USE TravelAnalytics;

  CREATE DATASET airports
  ON `travel-sample`.inventory.airport;

  CREATE DATASET routes
  ON `travel-sample`.inventory.route;

  CREATE DATASET hotels
  ON `travel-sample`.inventory.hotel;

  CONNECT LINK Local;
  EOF
  ```

- {% include step_label.html %} Abre la sección **Analytics** de la Web Console y ejecuta primero la sentencia de creación del Dataverse.

  ```sql
  CREATE DATAVERSE TravelAnalytics IF NOT EXISTS;
  ```

- {% include step_label.html %} Establece `TravelAnalytics` como contexto activo para que las siguientes Analytics collections se creen dentro del mismo namespace.

  ```sql
  USE TravelAnalytics;
  ```

- {% include step_label.html %} Crea la Analytics collection `airports` a partir de `travel-sample.inventory.airport`.

  ```sql
  CREATE DATASET airports
  ON `travel-sample`.inventory.airport;
  ```

- {% include step_label.html %} Crea la Analytics collection `routes` a partir de `travel-sample.inventory.route`.

  ```sql
  CREATE DATASET routes
  ON `travel-sample`.inventory.route;
  ```

- {% include step_label.html %} Crea la Analytics collection `hotels` a partir de `travel-sample.inventory.hotel`.

  ```sql
  CREATE DATASET hotels
  ON `travel-sample`.inventory.hotel;
  ```

> En Couchbase Analytics, la palabra Dataset corresponde al término histórico utilizado por la sintaxis y los metadatos. Conceptualmente, cada objeto representa una Analytics collection.

### Tarea 2.2. Conectar el Link Local y verificar la sincronización

- {% include step_label.html %} Conecta el Link Local para iniciar la ingestión y mantener sincronizados los datos operacionales mediante DCP.

  ```sql
  USE TravelAnalytics;

  CONNECT LINK Local;
  ```

- {% include step_label.html %} Consulta los metadatos para confirmar que las tres Analytics collections hayan quedado registradas.

  ```sql
  SELECT ds.DataverseName,
         ds.DatasetName
  FROM Metadata.`Dataset` AS ds
  WHERE ds.DataverseName = "TravelAnalytics"
  ORDER BY ds.DatasetName;
  ```

- {% include step_label.html %} Ejecuta una consulta de conteo sobre las tres collections para comprobar que la sincronización haya comenzado.

  ```sql
  USE TravelAnalytics;

  SELECT
    (SELECT VALUE COUNT(*) FROM airports)[0] AS airport_count,
    (SELECT VALUE COUNT(*) FROM routes)[0]   AS route_count,
    (SELECT VALUE COUNT(*) FROM hotels)[0]   AS hotel_count;
  ```

- {% include step_label.html %} Si algún conteo todavía es cero, espera unos segundos y repite únicamente la consulta de conteo, porque la ingestión inicial mediante DCP es asíncrona.

> No utilices valores rígidos como criterio de aprobación. La validación correcta consiste en confirmar que cada conteo sea mayor que cero y sea coherente con la collection operacional correspondiente.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

{% include support-prompt.html task="tarea2" %}

---

## 📊 Tarea 3. Ejecutar agregaciones, UNNEST y JOIN analítico

En esta tarea ejecutarás tres consultas representativas del servicio Analytics: una agregación, una consulta con UNNEST y un JOIN entre Analytics collections.

### Tarea 3.1. Ejecutar una agregación por país

- {% include step_label.html %} Crea `analytics-airports-by-country.sqlpp` para conservar la consulta utilizada en la primera prueba analítica.

  ```bash
  cat > analytics-airports-by-country.sqlpp << 'EOF'
  USE TravelAnalytics;

  SELECT a.country,
         COUNT(*) AS total_airports
  FROM airports AS a
  WHERE a.country IS NOT MISSING
    AND a.country IS NOT NULL
  GROUP BY a.country
  ORDER BY total_airports DESC
  LIMIT 10;
  EOF
  ```

- {% include step_label.html %} Ejecuta la consulta desde el editor Analytics de la Web Console y revisa cómo `GROUP BY` agrupa documentos antes de calcular `COUNT(*)`.

  ```sql
  USE TravelAnalytics;

  SELECT a.country,
         COUNT(*) AS total_airports
  FROM airports AS a
  WHERE a.country IS NOT MISSING
    AND a.country IS NOT NULL
  GROUP BY a.country
  ORDER BY total_airports DESC
  LIMIT 10;
  ```

- {% include step_label.html %} Revisa la sección de métricas y registra `elapsedTime`, `executionTime`, `resultCount` y `processedObjects` en `analytics-results.md`.

### Tarea 3.2. Calcular promedios mediante UNNEST

- {% include step_label.html %} Crea `analytics-hotel-ratings.sqlpp` con una consulta que expanda el array reviews antes de promediar ratings.Overall.

  ```bash
  cat > analytics-hotel-ratings.sqlpp << 'EOF'
  USE TravelAnalytics;

  SELECT h.city,
         COUNT(DISTINCT META(h).id) AS total_hotels,
         AVG(r.ratings.Overall) AS average_rating
  FROM hotels AS h
  UNNEST h.reviews AS r
  WHERE h.city IS NOT MISSING
    AND h.city IS NOT NULL
    AND r.ratings.Overall IS NOT MISSING
    AND r.ratings.Overall IS NOT NULL
  GROUP BY h.city
  ORDER BY total_hotels DESC
  LIMIT 10;
  EOF
  ```

- {% include step_label.html %} Ejecuta la consulta en Analytics y confirma que `UNNEST` genere una fila lógica por elemento de reviews antes de aplicar `AVG`.

  ```sql
  USE TravelAnalytics;

  SELECT
      hotel.city,
      COUNT(DISTINCT hotel.hotel_id) AS total_hotels,
      AVG(hotel.overall_rating) AS average_rating
  FROM (
      SELECT
          META(h).id AS hotel_id,
          h.city AS city,
          r.ratings.Overall AS overall_rating
      FROM hotels AS h
      UNNEST h.reviews AS r
      WHERE h.city IS NOT MISSING
        AND h.city IS NOT NULL
        AND r.ratings.Overall IS NOT MISSING
        AND r.ratings.Overall IS NOT NULL
  ) AS hotel
  GROUP BY hotel.city
  ORDER BY total_hotels DESC
  LIMIT 10;
  ```

### Tarea 3.3. Ejecutar un JOIN entre routes y airports

- {% include step_label.html %} Crea `analytics-routes-airports-join.sqlpp` para relacionar rutas con su aeropuerto de origen.

  ```bash
  cat > analytics-routes-airports-join.sqlpp << 'EOF'
  USE TravelAnalytics;

  SELECT a.airportname,
         a.city,
         a.country,
         COUNT(*) AS outbound_routes
  FROM airports AS a
  JOIN routes AS r
    ON a.faa = r.sourceairport
  WHERE a.country = "United States"
  GROUP BY a.airportname, a.city, a.country
  ORDER BY outbound_routes DESC
  LIMIT 10;
  EOF
  ```

- {% include step_label.html %} Ejecuta el JOIN y revisa cómo Analytics combina ambas collections antes de agrupar las rutas salientes.

  ```sql
  USE TravelAnalytics;

  SELECT a.airportname,
         a.city,
         a.country,
         COUNT(*) AS outbound_routes
  FROM airports AS a
  JOIN routes AS r
    ON a.faa = r.sourceairport
  WHERE a.country = "United States"
  GROUP BY a.airportname, a.city, a.country
  ORDER BY outbound_routes DESC
  LIMIT 10;
  ```

- {% include step_label.html %} Documenta en `analytics-results.md` qué operación fue más compleja y evita interpretar `processedObjects` como prueba automática de un escaneo completo.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

{% include support-prompt.html task="tarea3" %}

---

## 🌐 Tarea 4. Ejecutar Analytics REST API e interpretar respuestas

En esta tarea ejecutarás una consulta exitosa, analizarás métricas y provocarás un error controlado para revisar la estructura del campo errors.

### Tarea 4.1. Ejecutar y guardar una respuesta exitosa

- {% include step_label.html %} Crea `rest-success-query.sqlpp` con una consulta breve y reproducible.

  ```bash
  cat > rest-success-query.sqlpp << 'EOF'
  USE TravelAnalytics;

  SELECT a.country,
         COUNT(*) AS total
  FROM airports AS a
  WHERE a.country IS NOT MISSING
  GROUP BY a.country
  ORDER BY total DESC
  LIMIT 5;
  EOF
  ```

- {% include step_label.html %} Envía la consulta mediante `--data-urlencode` para preservar saltos de línea, comillas y caracteres especiales sin construir JSON manualmente.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    "${CB_ANALYTICS_SERVICE}" \
    --data-urlencode "statement@rest-success-query.sqlpp" \
    --data-urlencode "client_context_id=lab14-rest-success-001" \
    --data-urlencode "pretty=true" \
    > analytics-success-response.json
  ```

- {% include step_label.html %} Formatea la respuesta y localiza requestID, clientContextID, results, status y metrics.

  ```bash
  python -m json.tool analytics-success-response.json
  ```

- {% include step_label.html %} Ejecuta un script corto para resumir los campos más importantes sin asumir que `errors` debe aparecer cuando no existen errores.

  ```bash
  python - << 'PY'
  import json

  with open("analytics-success-response.json", encoding="utf-8") as file:
      response = json.load(file)

  print("status:", response.get("status"))
  print("clientContextID:", response.get("clientContextID"))
  print("resultCount:", response.get("metrics", {}).get("resultCount"))
  print("elapsedTime:", response.get("metrics", {}).get("elapsedTime"))
  print("executionTime:", response.get("metrics", {}).get("executionTime"))
  print("processedObjects:", response.get("metrics", {}).get("processedObjects"))
  print("errors:", response.get("errors", []))
  PY
  ```

### Tarea 4.2. Analizar una respuesta con error

- {% include step_label.html %} Crea `rest-error-query.sqlpp` con una referencia intencional a un dataset inexistente.

  ```bash
  cat > rest-error-query.sqlpp << 'EOF'
  USE TravelAnalytics;

  SELECT *
  FROM dataset_inexistente;
  EOF
  ```

- {% include step_label.html %} Ejecuta la consulta inválida y guarda la respuesta para analizarla sin perder evidencia.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    "${CB_ANALYTICS_SERVICE}" \
    --data-urlencode "statement@rest-error-query.sqlpp" \
    --data-urlencode "client_context_id=lab14-rest-error-001" \
    --data-urlencode "pretty=true" \
    > analytics-error-response.json
  ```

- {% include step_label.html %} Formatea la respuesta y confirma que status sea distinto de success y que errors contenga al menos un objeto con code y msg.

  ```bash
  python -m json.tool analytics-error-response.json
  ```

- {% include step_label.html %} Valida la estructura del error sin depender de un código numérico fijo. El dataset_inexistente no existe dentro del scope Analytics TravelAnalytics

  ```bash
  python - << 'PY'
  import json

  with open("analytics-error-response.json", encoding="utf-8") as file:
      response = json.load(file)

  errors = response.get("errors", [])

  print("status:", response.get("status"))
  print("error_count:", len(errors))

  if errors:
      print("first_error_code:", errors[0].get("code"))
      print("first_error_message:", errors[0].get("msg"))
  PY
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

{% include support-prompt.html task="tarea4" %}

---

## ⚙️ Tarea 5. Automatizar y validar el entorno Analytics

En esta tarea crearás un script Bash que ejecute una consulta analítica, interprete la respuesta y valide los recursos principales del laboratorio.

### Tarea 5.1. Crear un script de automatización

- {% include step_label.html %} Crea `run-analytics-report.sh` con una consulta que agrupe aeropuertos por país y procese el resultado mediante Python.

  ```bash
  cat > run-analytics-report.sh << 'EOF'
  #!/usr/bin/env bash

  set -u

  QUERY_FILE="rest-success-query.sqlpp"
  RESPONSE_FILE="analytics-report-response.json"

  echo "Ejecutando consulta analítica..."

  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    "${CB_ANALYTICS_SERVICE}" \
    --data-urlencode "statement@${QUERY_FILE}" \
    --data-urlencode "client_context_id=lab14-report-001" \
    --data-urlencode "pretty=true" \
    > "${RESPONSE_FILE}"

  python - << 'PY'
  import json
  import sys

  with open("analytics-report-response.json", encoding="utf-8") as file:
      response = json.load(file)

  status = response.get("status")
  print("Estado:", status)

  if status != "success":
      print("La consulta no terminó correctamente.")
      print(json.dumps(response.get("errors", []), indent=2, ensure_ascii=False))
      sys.exit(1)

  metrics = response.get("metrics", {})
  print("Tiempo transcurrido:", metrics.get("elapsedTime"))
  print("Tiempo de ejecución:", metrics.get("executionTime"))
  print("Resultados:")

  for row in response.get("results", []):
      print(f"  {row.get('country')}: {row.get('total')}")
  PY
  EOF
  ```

- {% include step_label.html %} Asigna permiso de ejecución al script sin moverlo fuera del directorio de la práctica.

  ```bash
  chmod +x run-analytics-report.sh
  ```

- {% include step_label.html %} Ejecuta el script y confirma que muestre `Estado: success` y una lista de países con sus conteos.

  ```bash
  ./run-analytics-report.sh
  ```

### Tarea 5.2. Crear y ejecutar la validación final

- {% include step_label.html %} Crea `validate-lab14.sh` para comprobar imagen, puerto, servicio, metadatos, conteos y respuesta REST.

  {%raw%}
  ```bash
  cat > validate-lab14.sh << 'EOF'
  #!/usr/bin/env bash

  set -u

  PASS=0
  FAIL=0

  pass() {
    echo "[PASS] $1"
    PASS=$((PASS + 1))
  }

  fail() {
    echo "[FAIL] $1"
    FAIL=$((FAIL + 1))
  }

  IMAGE=$(docker inspect couchbase-lab --format '{{.Config.Image}}' 2>/dev/null || true)

  if [ "${IMAGE}" = "couchbase/server:enterprise-7.6.2" ]; then
    pass "La imagen corresponde a Enterprise 7.6.2"
  else
    fail "Imagen detectada: ${IMAGE:-no disponible}"
  fi

  if docker port couchbase-lab | grep -q '^8095/tcp'; then
    pass "El puerto 8095 está publicado"
  else
    fail "El puerto 8095 no está publicado"
  fi

  HEALTH_STATUS=$(python - << 'PY'
  import json

  with open("analytics-healthcheck-response.json", encoding="utf-8") as file:
      response = json.load(file)

  print(response.get("status", "unknown"))
  PY
  )

  if [ "${HEALTH_STATUS}" = "success" ]; then
    pass "Analytics ejecuta consultas correctamente"
  else
    fail "Estado del health check: ${HEALTH_STATUS}"
  fi

  METADATA_RESPONSE=$(curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    "${CB_ANALYTICS_SERVICE}" \
    --data-urlencode 'statement=
      SELECT VALUE ds.DatasetName
      FROM Metadata.`Dataset` AS ds
      WHERE ds.DataverseName = "TravelAnalytics"
      ORDER BY ds.DatasetName;' \
    --data-urlencode 'client_context_id=lab14-validation-metadata-001')

  DATASET_COUNT=$(printf '%s' "${METADATA_RESPONSE}" \
    | python -c "import json,sys; print(len(json.load(sys.stdin).get('results',[])))" 2>/dev/null || echo 0)

  if [ "${DATASET_COUNT}" -ge 3 ] 2>/dev/null; then
    pass "TravelAnalytics contiene al menos tres Analytics collections"
  else
    fail "Collections detectadas: ${DATASET_COUNT}"
  fi

  COUNT_RESPONSE=$(curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    "${CB_ANALYTICS_SERVICE}" \
    --data-urlencode 'statement=
      USE TravelAnalytics;
      SELECT
        (SELECT VALUE COUNT(*) FROM airports)[0] AS airports,
        (SELECT VALUE COUNT(*) FROM routes)[0] AS routes,
        (SELECT VALUE COUNT(*) FROM hotels)[0] AS hotels;' \
    --data-urlencode 'client_context_id=lab14-validation-counts-001')

  COUNTS_OK=$(printf '%s' "${COUNT_RESPONSE}" \
    | python -c "
  import json,sys
  response=json.load(sys.stdin)
  row=(response.get('results') or [{}])[0]
  print('yes' if all(row.get(k,0) > 0 for k in ('airports','routes','hotels')) else 'no')
  " 2>/dev/null || echo no)

  if [ "${COUNTS_OK}" = "yes" ]; then
    pass "Las tres Analytics collections contienen datos"
  else
    fail "Uno o más conteos son cero"
  fi

  REST_STATUS=$(python - << 'PY'
  import json

  with open("analytics-success-response.json", encoding="utf-8") as file:
      response = json.load(file)

  print(response.get("status", "unknown"))
  PY
  )

  if [ "${REST_STATUS}" = "success" ]; then
    pass "La respuesta REST exitosa tiene status success"
  else
    fail "Estado REST detectado: ${REST_STATUS}"
  fi

  ERROR_COUNT=$(python - << 'PY'
  import json

  with open("analytics-error-response.json", encoding="utf-8") as file:
      response = json.load(file)

  print(len(response.get("errors", [])))
  PY
  )

  if [ "${ERROR_COUNT}" -gt 0 ] 2>/dev/null; then
    pass "La consulta inválida devuelve información en errors"
  else
    fail "No se encontraron errores en la respuesta inválida"
  fi

  echo
  echo "Resultado final: ${PASS} verificaciones correctas | ${FAIL} verificaciones fallidas"
  EOF
  ```
  {%endraw%}

- {% include step_label.html %} Asigna permiso de ejecución al script de validación.

  ```bash
  chmod +x validate-lab14.sh
  ```

- {% include step_label.html %} Ejecuta la validación y corrige cualquier resultado marcado como FAIL antes de finalizar.

  ```bash
  ./validate-lab14.sh
  ```

- {% include step_label.html %} Conserva TravelAnalytics, las tres Analytics collections y los archivos generados, porque continúan consumiendo recursos para sincronización y pueden reutilizarse posteriormente.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

{% include support-prompt.html task="tarea5" %}

---

## 🛠️ Resolución de problemas

### Problema 1. El puerto 8095 no está publicado

- {% include step_label.html %} Ejecuta `docker port couchbase-lab` y confirma si existe una línea para 8095/tcp.

- {% include step_label.html %} Si no existe, el contenedor debe recrearse con el puerto publicado; no intentes resolverlo únicamente desde la Web Console.

### Problema 2. Analytics no aparece entre los servicios del nodo

- {% include step_label.html %} Consulta `nodeServices` y busca cbas, cbasAdmin, cbasCc o el puerto 8095.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${CB_URL}/pools/default/nodeServices" \
    | python -m json.tool
  ```

- {% include step_label.html %} Si Analytics no aparece, revisa la configuración original del clúster y la memoria disponible en Docker antes de modificar servicios.

### Problema 3. CREATE DATASET devuelve que el objeto ya existe

- {% include step_label.html %} Consulta Metadata.Dataset para confirmar que el dataset ya fue creado.

  ```sql
  SELECT ds.DataverseName,
         ds.DatasetName
  FROM Metadata.`Dataset` AS ds
  WHERE ds.DataverseName = "TravelAnalytics";
  ```

- {% include step_label.html %} Si el objeto ya existe, no lo elimines; continúa con CONNECT LINK Local y los conteos.

### Problema 4. Los conteos permanecen en cero

- {% include step_label.html %} Confirma que Link Local esté conectado y repite la consulta de conteo después de unos segundos.

- {% include step_label.html %} Verifica que las collections operacionales tengan documentos mediante Query Service.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    "${CB_QUERY_URL}" \
    --data-urlencode 'statement=
      SELECT
        (SELECT RAW COUNT(*) FROM `travel-sample`.inventory.airport)[0] AS airports,
        (SELECT RAW COUNT(*) FROM `travel-sample`.inventory.route)[0] AS routes,
        (SELECT RAW COUNT(*) FROM `travel-sample`.inventory.hotel)[0] AS hotels;' \
    | python -m json.tool
  ```

### Problema 5. CONNECT LINK Local devuelve que el enlace ya está conectado

- {% include step_label.html %} Interpreta el mensaje como una condición reutilizable y continúa con la validación de conteos.

- {% include step_label.html %} No desconectes el enlace si los datasets ya están sincronizando correctamente.

### Problema 6. La consulta con UNNEST no devuelve resultados

- {% include step_label.html %} Confirma que hotels contenga documentos y que reviews exista como array.

- {% include step_label.html %} Prueba primero una consulta exploratoria.

  ```sql
  USE TravelAnalytics;

  SELECT h.name,
         ARRAY_LENGTH(h.reviews) AS review_count
  FROM hotels AS h
  WHERE h.reviews IS NOT MISSING
  LIMIT 5;
  ```

### Problema 7. La respuesta REST no es JSON válido

- {% include step_label.html %} Revisa que `CB_ANALYTICS_SERVICE` apunte exactamente a `/analytics/service`.

- {% include step_label.html %} Ejecuta la consulta sin redirección para observar el mensaje completo devuelto por curl.

### Problema 8. python -m json.tool no está disponible

- {% include step_label.html %} Verifica qué comando de Python está instalado.

  ```bash
  python --version
  python3 --version
  ```

- {% include step_label.html %} Sustituye `python` por `python3` cuando esa sea la instalación disponible.