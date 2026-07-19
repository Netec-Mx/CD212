---
layout: lab
title: "Práctica 15: Uso de funciones y window functions en Analytics"
permalink: /lab15/lab15/
images_base: /labs/lab15/img
duration: "70 minutos"
objective:
  - Verificar que Couchbase Server Enterprise 7.6.2 y el entorno Analytics creado en la práctica 14 continúen disponibles.
  - Aplicar funciones integradas de cadenas, valores ausentes, fechas y arreglos sobre datos reales y valores controlados.
  - Implementar ROW_NUMBER, RANK, DENSE_RANK y NTILE sobre métricas derivadas de las rutas.
  - Utilizar LAG, LEAD, FIRST_VALUE, LAST_VALUE, SUM OVER y AVG OVER con marcos de ventana explícitos.
  - Analizar consultas mediante EXPLAIN e interpretar las etapas principales del plan de ejecución.
prerequisites:
  - Haber completado la Práctica 14.
  - Conservar el Dataverse TravelAnalytics y las Analytics collections airports, routes y hotels.
  - Tener Couchbase Server Enterprise 7.6.2 en ejecución mediante la imagen couchbase/server:enterprise-7.6.2.
  - Tener habilitado el servicio Analytics y publicado el puerto 8095.
  - Tener acceso a la Web Console en http://localhost:8091.
  - Utilizar Visual Studio Code con Git Bash como terminal integrada.
  - Comprender GROUP BY, agregaciones, subconsultas, UNNEST y JOIN.
introduction:
  - En esta práctica ampliarás el uso de SQL++ Analytics con funciones integradas y window functions. Las consultas utilizarán métricas que realmente existen o pueden derivarse de travel-sample, como rutas por aerolínea. Cada consulta se guardará en un archivo SQL++ y se ejecutará desde Analytics Workbench para conservar evidencia y facilitar su repetición.
slug: lab15
lab_number: 15
final_result: >
  Al finalizar habrás aplicado funciones integradas, generado rankings y cuartiles, comparado filas mediante LAG y LEAD, calculado extremos, acumulados y promedios móviles, y analizado planes de ejecución sobre TravelAnalytics.
notes:
  - Todos los comandos de sistema deben ejecutarse desde Git Bash dentro de Visual Studio Code.
  - Todas las sentencias SQL++ deben ejecutarse desde Analytics Workbench en la Web Console.
  - La imagen obligatoria es couchbase/server:enterprise-7.6.2.
  - Se utilizarán las credenciales Administrator y Password123!.
  - Esta práctica reutiliza airports, routes y hotels; no crea Analytics collections con nombres diferentes.
  - Los recursos y archivos generados deben conservarse al finalizar.
references: []
prev: /lab14/lab14/
next: /lab1/lab1/
---

---
<!-- Aquí comienzan las instrucciones paso a paso de la práctica -->

## 🧭 Tarea 1. Preparar y validar el entorno Analytics

En esta tarea prepararás el directorio de trabajo, validarás la imagen Docker Enterprise 7.6.2 y comprobarás que los recursos creados en la práctica 14 continúen disponibles.

### Tarea 1.1. Preparar Visual Studio Code, Git Bash y las variables

- {% include step_label.html %} Abre Docker Desktop desde el menú Inicio de Windows y espera hasta que la aplicación muestre que el motor está ejecutándose, porque el contenedor de Couchbase no podrá iniciar ni publicar sus puertos mientras Docker continúe cargando.

- {% include step_label.html %} Abre Visual Studio Code, selecciona **File → Open Folder** y abre `C:\LABS\couchbase-nosql`, de modo que el explorador lateral muestre todas las prácticas dentro del mismo proyecto.

- {% include step_label.html %} Selecciona **Terminal → New Terminal**, abre el selector de perfiles situado junto al botón `+` y elige **Git Bash**, porque los comandos posteriores utilizan variables `export`, redirecciones y scripts compatibles con Bash.

- {% include step_label.html %} Crea el directorio de la práctica y cambia a esa ubicación para que todos los archivos SQL++, JSON y Bash queden almacenados como evidencia.

  ```bash
  mkdir -p /c/LABS/couchbase-nosql/lab15
  cd /c/LABS/couchbase-nosql/lab15
  pwd
  ```

**Salida esperada:**

```text
/c/LABS/couchbase-nosql/lab15
```

- {% include step_label.html %} Define las credenciales, URLs y el nombre del Dataverse en la misma terminal de Git Bash; estas variables solo permanecerán disponibles mientras la terminal continúe abierta.

  ```bash
  export CB_HOST="localhost"
  export CB_ADMIN="Administrator"
  export CB_PASS="Password123!"

  export CB_URL="http://${CB_HOST}:8091"
  export CB_ANALYTICS_URL="http://${CB_HOST}:8095"
  export CB_ANALYTICS_SERVICE="${CB_ANALYTICS_URL}/analytics/service"

  export ANALYTICS_DATAVERSE="TravelAnalytics"
  ```

- {% include step_label.html %} Imprime las variables principales para confirmar que no existan espacios, errores de escritura ni puertos incorrectos antes de utilizarlas.

  ```bash
  printf "CB_URL=%s\nCB_ANALYTICS_SERVICE=%s\nDATAVERSE=%s\n" \
    "${CB_URL}" \
    "${CB_ANALYTICS_SERVICE}" \
    "${ANALYTICS_DATAVERSE}"
  ```

### Tarea 1.2. Validar la imagen, el puerto y los recursos Analytics

- {% include step_label.html %} Ejecuta `docker inspect` desde Git Bash para comprobar que el contenedor `couchbase-lab` utilice exactamente la imagen Enterprise definida para el curso.

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

- {% include step_label.html %} Comprueba que el contenedor esté activo y revisa su estado; si no aparece en la salida, ejecuta `docker start couchbase-lab` y repite la validación.

  {%raw%}
  ```bash
  docker ps --filter "name=couchbase-lab" \
    --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
  ```
  {%endraw%}

- {% include step_label.html %} Verifica que Docker publique el puerto 8095 hacia Windows, porque Analytics Workbench y las llamadas REST dependen de ese puerto.

  ```bash
  docker port couchbase-lab | grep '^8095/tcp'
  ```

- {% include step_label.html %} Ejecuta una consulta mínima desde Git Bash y guarda la respuesta en un archivo, lo que permite comprobar en una sola operación la autenticación, el puerto, el endpoint y el motor SQL++ Analytics.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    "${CB_ANALYTICS_SERVICE}" \
    --data-urlencode 'statement=SELECT VALUE 1;' \
    --data-urlencode 'client_context_id=lab15-healthcheck-001' \
    > analytics-healthcheck-response.json
  ```

- {% include step_label.html %} Formatea la respuesta y confirma que `status` sea `success` y que `results` contenga el valor `1`.

  ```bash
  python -m json.tool analytics-healthcheck-response.json
  ```

- {% include step_label.html %} Abre `http://localhost:8091` en el navegador, inicia sesión con `Administrator` y `Password123!`, selecciona **Analytics** en el menú lateral y localiza el editor central de consultas.

- {% include step_label.html %} Copia la siguiente consulta en Analytics Workbench, presiona **Execute** y confirma que aparezcan `airports`, `hotels` y `routes`.

  ```sql
  SELECT ds.DataverseName,
         ds.DatasetName
  FROM Metadata.`Dataset` AS ds
  WHERE ds.DataverseName = "TravelAnalytics"
  ORDER BY ds.DatasetName;
  ```

- {% include step_label.html %} Borra la consulta anterior del editor, pega la consulta de conteos y presiona **Execute** para verificar que las tres Analytics collections contengan datos sincronizados.

  ```sql
  USE TravelAnalytics;

  SELECT
    (SELECT VALUE COUNT(*) FROM airports)[0] AS airport_count,
    (SELECT VALUE COUNT(*) FROM routes)[0]   AS route_count,
    (SELECT VALUE COUNT(*) FROM hotels)[0]   AS hotel_count;
  ```

**Validación de la tarea:**

- La imagen es `couchbase/server:enterprise-7.6.2`.
- El puerto 8095 aparece publicado.
- La consulta REST devuelve `status: success`.
- Metadata muestra `airports`, `hotels` y `routes`.
- Los tres conteos son mayores que cero.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

{% include support-prompt.html task="tarea1" %}

---

## 🧰 Tarea 2. Aplicar funciones integradas de SQL++ Analytics

En esta tarea crearás tres archivos de consulta y ejecutarás cada uno desde Analytics Workbench. Los ejemplos utilizan datos reales cuando existen y valores literales cuando se requiere un resultado completamente reproducible.

### Tarea 2.1. Aplicar funciones de cadena y valores ausentes

- {% include step_label.html %} Regresa a Visual Studio Code y, desde la terminal Git Bash ubicada en `lab15`, crea `builtin-string-functions.sqlpp` con funciones para normalizar texto, dividir nombres, detectar patrones y reemplazar valores ausentes.

  ```bash
  cat > builtin-string-functions.sqlpp << 'EOF'
  USE TravelAnalytics;

  SELECT h.name,
         UPPER(h.country) AS country_upper,
         LENGTH(TRIM(h.name)) AS name_length,
         SPLIT(TRIM(h.name), " ")[0] AS first_word,
         REGEXP_CONTAINS(
           h.name,
           "(hotel|inn|resort|lodge)",
           "i"
         ) AS known_establishment,
         COALESCE(h.email, "sin_email") AS email_value,
         COALESCE(h.phone, "sin_telefono") AS phone_value
  FROM hotels AS h
  WHERE h.name IS NOT MISSING
    AND h.name IS NOT NULL
  ORDER BY h.name
  LIMIT 10;
  EOF
  ```

- {% include step_label.html %} Confirma que el archivo exista y muestra su contenido antes de llevarlo a la Web Console.

  ```bash
  ls -l builtin-string-functions.sqlpp
  cat builtin-string-functions.sqlpp
  ```

- {% include step_label.html %} En Visual Studio Code, abre el archivo desde el explorador lateral, selecciona todo su contenido y cópialo.

- {% include step_label.html %} Cambia al navegador, abre **Analytics**, limpia el editor, pega la consulta y presiona **Execute** para ejecutarla en el servicio Analytics.

- {% include step_label.html %} Revisa que `country_upper` esté en mayúsculas, `name_length` sea numérico, `first_word` contenga la primera palabra y que `COALESCE` devuelva textos de sustitución cuando email o phone estén ausentes.

### Tarea 2.2. Aplicar funciones de fecha y arreglos

- {% include step_label.html %} Crea `builtin-date-functions.sqlpp` desde Git Bash con fechas conocidas, de modo que puedas validar el resultado sin depender de fechas almacenadas en los documentos.

  ```bash
  cat > builtin-date-functions.sqlpp << 'EOF'
  SELECT NOW_STR() AS current_timestamp,
         DATE_DIFF_STR(
           "2026-12-31T00:00:00Z",
           "2026-01-01T00:00:00Z",
           "day"
         ) AS days_between_dates,
         DATE_ADD_STR(
           "2026-01-01T00:00:00Z",
           30,
           "day"
         ) AS date_plus_30_days;
  EOF
  ```

- {% include step_label.html %} Abre el archivo en Visual Studio Code, copia la consulta, pégala en Analytics Workbench y presiona **Execute**.

- {% include step_label.html %} Confirma que `current_timestamp` muestre la fecha actual, que `days_between_dates` sea numérico y que `date_plus_30_days` represente treinta días después del 1 de enero de 2026.

- {% include step_label.html %} Crea `builtin-array-functions.sqlpp` con arreglos literales para demostrar eliminación de duplicados, aplanamiento y conteo sin depender de un field opcional del sample bucket.

  ```bash
  cat > builtin-array-functions.sqlpp << 'EOF'
  SELECT ARRAY_DISTINCT(
           ["wifi", "pool", "wifi", "breakfast", "pool"]
         ) AS unique_values,
         ARRAY_FLATTEN(
           [
             ["wifi", "pool"],
             ["parking"],
             ["breakfast", "wifi"]
           ],
           1
         ) AS flattened_values,
         ARRAY_LENGTH(
           ARRAY_DISTINCT(
             ["wifi", "pool", "wifi", "breakfast", "pool"]
           )
         ) AS unique_count;
  EOF
  ```

- {% include step_label.html %} Abre el archivo, copia su contenido, pégalo en Analytics Workbench y ejecútalo.

- {% include step_label.html %} Comprueba que `unique_values` no contenga duplicados, `flattened_values` sea un solo arreglo y `unique_count` represente la cantidad de valores distintos.

**Validación de la tarea:**

- Los tres archivos `.sqlpp` existen en `lab15`.
- Cada consulta se ejecutó desde Analytics Workbench.
- Las funciones devolvieron resultados sin errores.
- Comprendes por qué `COALESCE` cubre NULL y MISSING.
- Comprendes por qué los ejemplos de fecha y arrays usan valores controlados.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

{% include support-prompt.html task="tarea2" %}

---

## 🏆 Tarea 3. Crear rankings y cuartiles con window functions

En esta tarea primero generarás una métrica agregada de rutas por aerolínea y después aplicarás funciones de ventana sobre ese resultado.

### Tarea 3.1. Construir y ejecutar los rankings

- {% include step_label.html %} Crea `airline-route-totals.sqlpp` desde Git Bash para comprobar la métrica base antes de utilizar window functions.

  ```bash
  cat > airline-route-totals.sqlpp << 'EOF'
  USE TravelAnalytics;

  SELECT r.airline,
         COUNT(*) AS total_routes,
         COUNT(DISTINCT r.sourceairport) AS origin_airports
  FROM routes AS r
  WHERE r.airline IS NOT MISSING
    AND r.airline IS NOT NULL
  GROUP BY r.airline
  ORDER BY total_routes DESC, r.airline
  LIMIT 20;
  EOF
  ```

- {% include step_label.html %} Abre el archivo en Visual Studio Code, copia su contenido, pégalo en Analytics Workbench y ejecútalo para confirmar que cada aerolínea tenga un conteo numérico.

- {% include step_label.html %} Crea `airline-rankings.sqlpp` con `ROW_NUMBER`, `RANK` y `DENSE_RANK` aplicados sobre la subconsulta agregada.

  ```bash
  cat > airline-rankings.sqlpp << 'EOF'
  USE TravelAnalytics;

  SELECT totals.airline,
         totals.total_routes,
         totals.origin_airports,
         ROW_NUMBER() OVER (
           ORDER BY totals.total_routes DESC,
                    totals.airline ASC
         ) AS row_number_value,
         RANK() OVER (
           ORDER BY totals.total_routes DESC
         ) AS rank_value,
         DENSE_RANK() OVER (
           ORDER BY totals.total_routes DESC
         ) AS dense_rank_value
  FROM (
    SELECT r.airline,
           COUNT(*) AS total_routes,
           COUNT(DISTINCT r.sourceairport) AS origin_airports
    FROM routes AS r
    WHERE r.airline IS NOT MISSING
      AND r.airline IS NOT NULL
    GROUP BY r.airline
  ) AS totals
  ORDER BY rank_value, totals.airline
  LIMIT 20;
  EOF
  ```

- {% include step_label.html %} Copia y ejecuta `airline-rankings.sqlpp` desde Analytics Workbench, y verifica que `row_number_value` sea único y ascendente desde 1.

- {% include step_label.html %} Observa `rank_value` y `dense_rank_value`; cuando dos aerolíneas tengan el mismo total, RANK puede dejar un salto y DENSE_RANK debe continuar sin huecos.

- {% include step_label.html %} Ejecuta el siguiente ejemplo literal directamente en Analytics Workbench para garantizar un empate y confirmar el comportamiento de las tres funciones.

  ```sql
  SELECT item.name,
         item.score,
         ROW_NUMBER() OVER (
           ORDER BY item.score DESC, item.name
         ) AS row_number_value,
         RANK() OVER (
           ORDER BY item.score DESC
         ) AS rank_value,
         DENSE_RANK() OVER (
           ORDER BY item.score DESC
         ) AS dense_rank_value
  FROM [
    {"name": "A", "score": 100},
    {"name": "B", "score": 100},
    {"name": "C", "score": 90}
  ] AS item
  ORDER BY item.score DESC, item.name;
  ```

**Comportamiento esperado:**

```text
ROW_NUMBER: 1, 2, 3
RANK:       1, 1, 3
DENSE_RANK: 1, 1, 2
```

### Tarea 3.2. Segmentar aerolíneas con NTILE

- {% include step_label.html %} Crea `airline-route-quartiles.sqlpp` desde Git Bash para dividir las aerolíneas en cuatro grupos de tamaño aproximadamente similar.

  ```bash
  cat > airline-route-quartiles.sqlpp << 'EOF'
  USE TravelAnalytics;

  SELECT totals.airline,
         totals.total_routes,
         NTILE(4) OVER (
           ORDER BY totals.total_routes ASC,
                    totals.airline ASC
         ) AS route_quartile
  FROM (
    SELECT r.airline,
           COUNT(*) AS total_routes
    FROM routes AS r
    WHERE r.airline IS NOT MISSING
      AND r.airline IS NOT NULL
    GROUP BY r.airline
  ) AS totals
  ORDER BY route_quartile,
           totals.total_routes,
           totals.airline;
  EOF
  ```

- {% include step_label.html %} Abre el archivo, copia la consulta, pégala en Analytics Workbench y presiona **Execute**.

- {% include step_label.html %} Revisa la columna `route_quartile` y confirma que solo contenga valores 1, 2, 3 o 4.

- {% include step_label.html %} Interpreta los cuartiles como grupos por posición ordenada, no como rangos fijos de total_routes.

**Validación de la tarea:**

- La métrica base devuelve aerolíneas y conteos.
- ROW_NUMBER produce números únicos.
- El ejemplo literal demuestra el tratamiento de empates.
- NTILE produce valores del 1 al 4.
- Todos los archivos permanecen guardados en `lab15`.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

{% include support-prompt.html task="tarea3" %}

---

## 📈 Tarea 4. Comparar filas y calcular agregados con ventana

En esta tarea aplicarás LAG, LEAD, FIRST_VALUE, LAST_VALUE, SUM, AVG y RATIO_TO_REPORT sobre la misma métrica de rutas por aerolínea.

### Tarea 4.1. Comparar filas anteriores y siguientes

- {% include step_label.html %} Crea `airline-lag-lead.sqlpp` desde Git Bash con un orden determinista que combine total_routes y airline.

  ```bash
  cat > airline-lag-lead.sqlpp << 'EOF'
  USE TravelAnalytics;

  SELECT totals.airline,
         totals.total_routes,
         LAG(totals.total_routes, 1, NULL) OVER (
           ORDER BY totals.total_routes DESC,
                    totals.airline ASC
         ) AS previous_total,
         LEAD(totals.total_routes, 1, NULL) OVER (
           ORDER BY totals.total_routes DESC,
                    totals.airline ASC
         ) AS next_total,
         totals.total_routes -
         LAG(totals.total_routes, 1, NULL) OVER (
           ORDER BY totals.total_routes DESC,
                    totals.airline ASC
         ) AS difference_from_previous
  FROM (
    SELECT r.airline,
           COUNT(*) AS total_routes
    FROM routes AS r
    WHERE r.airline IS NOT MISSING
      AND r.airline IS NOT NULL
    GROUP BY r.airline
  ) AS totals
  ORDER BY totals.total_routes DESC,
           totals.airline
  LIMIT 20;
  EOF
  ```

- {% include step_label.html %} Abre el archivo, copia la consulta, pégala en Analytics Workbench y ejecútala.

- {% include step_label.html %} Confirma que la primera fila tenga `previous_total` igual a NULL y que la última fila del conjunto completo sea la que no tenga `next_total`.

- {% include step_label.html %} Revisa `difference_from_previous` y recuerda que la diferencia se calcula según el orden establecido, no según una relación temporal.

### Tarea 4.2. Obtener extremos, acumulados y promedios móviles

- {% include step_label.html %} Crea `airline-first-last.sqlpp` para obtener el valor mínimo y máximo de toda la ventana mediante un marco explícito.

  ```bash
  cat > airline-first-last.sqlpp << 'EOF'
  USE TravelAnalytics;

  SELECT totals.airline,
         totals.total_routes,
         FIRST_VALUE(totals.total_routes) OVER (
           ORDER BY totals.total_routes ASC,
                    totals.airline ASC
           ROWS BETWEEN UNBOUNDED PRECEDING
                    AND UNBOUNDED FOLLOWING
         ) AS minimum_route_total,
         LAST_VALUE(totals.total_routes) OVER (
           ORDER BY totals.total_routes ASC,
                    totals.airline ASC
           ROWS BETWEEN UNBOUNDED PRECEDING
                    AND UNBOUNDED FOLLOWING
         ) AS maximum_route_total
  FROM (
    SELECT r.airline,
           COUNT(*) AS total_routes
    FROM routes AS r
    WHERE r.airline IS NOT MISSING
      AND r.airline IS NOT NULL
    GROUP BY r.airline
  ) AS totals
  ORDER BY totals.total_routes,
           totals.airline
  LIMIT 20;
  EOF
  ```

- {% include step_label.html %} Copia y ejecuta la consulta desde Analytics Workbench, y confirma que `minimum_route_total` y `maximum_route_total` permanezcan constantes en todas las filas mostradas.

- {% include step_label.html %} Crea `airline-window-aggregates.sqlpp` para calcular acumulado, porcentaje individual y promedio móvil.

  ```bash
  cat > airline-window-aggregates.sqlpp << 'EOF'
  USE TravelAnalytics;

  SELECT totals.airline,
         totals.total_routes,
         SUM(totals.total_routes) OVER (
           ORDER BY totals.total_routes DESC,
                    totals.airline ASC
           ROWS BETWEEN UNBOUNDED PRECEDING
                    AND CURRENT ROW
         ) AS cumulative_routes,
         ROUND(
           RATIO_TO_REPORT(totals.total_routes) OVER () * 100,
           2
         ) AS route_percentage,
         ROUND(
           AVG(totals.total_routes) OVER (
             ORDER BY totals.total_routes DESC,
                      totals.airline ASC
             ROWS BETWEEN 2 PRECEDING
                      AND 2 FOLLOWING
           ),
           2
         ) AS moving_average_5
  FROM (
    SELECT r.airline,
           COUNT(*) AS total_routes
    FROM routes AS r
    WHERE r.airline IS NOT MISSING
      AND r.airline IS NOT NULL
    GROUP BY r.airline
  ) AS totals
  ORDER BY totals.total_routes DESC,
           totals.airline
  LIMIT 20;
  EOF
  ```

- {% include step_label.html %} Abre el archivo, copia la consulta, pégala en Analytics Workbench y ejecútala.

- {% include step_label.html %} Confirma que `cumulative_routes` nunca disminuya, que `route_percentage` sea un porcentaje y que `moving_average_5` cambie conforme avanza la ventana.

**Validación de la tarea:**

- LAG y LEAD muestran valores anteriores y siguientes.
- FIRST_VALUE y LAST_VALUE usan toda la ventana.
- El acumulado no disminuye.
- El porcentaje se calcula respecto al total.
- El promedio móvil utiliza hasta cinco filas.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

{% include support-prompt.html task="tarea4" %}

---

## 🔍 Tarea 5. Analizar el plan y validar la práctica

En esta tarea usarás EXPLAIN para revisar la estructura lógica de una consulta y ejecutarás un script final que confirme los elementos principales del laboratorio.

### Tarea 5.1. Ejecutar EXPLAIN y comparar proyecciones

- {% include step_label.html %} Crea `explain-airline-ranking.sqlpp` desde Git Bash para solicitar el plan de la consulta de ranking sin ejecutar el resultado final completo.

  ```bash
  cat > explain-airline-ranking.sqlpp << 'EOF'
  USE TravelAnalytics;

  EXPLAIN
  SELECT totals.airline,
         totals.total_routes,
         RANK() OVER (
           ORDER BY totals.total_routes DESC
         ) AS route_rank
  FROM (
    SELECT r.airline,
           COUNT(*) AS total_routes
    FROM routes AS r
    WHERE r.airline IS NOT MISSING
      AND r.airline IS NOT NULL
    GROUP BY r.airline
  ) AS totals
  ORDER BY route_rank,
           totals.airline
  LIMIT 20;
  EOF
  ```

- {% include step_label.html %} Abre el archivo, copia su contenido, pégalo en Analytics Workbench y presiona **Execute**.

- {% include step_label.html %} Revisa el JSON del plan y localiza conceptualmente las etapas de lectura de routes, filtro de airline, agrupación, ordenamiento, cálculo de ventana y entrega de resultados.

- {% include step_label.html %} Borra el plan del editor y ejecuta la siguiente consulta de proyección amplia con un límite seguro.

  ```sql
  USE TravelAnalytics;

  SELECT *
  FROM routes AS r
  WHERE r.airline IS NOT MISSING
  LIMIT 100;
  ```

- {% include step_label.html %} Ejecuta después una proyección reducida que devuelva solamente los fields necesarios.

  ```sql
  USE TravelAnalytics;

  SELECT r.airline,
         r.sourceairport,
         r.destinationairport
  FROM routes AS r
  WHERE r.airline IS NOT MISSING
    AND r.sourceairport IS NOT MISSING
    AND r.destinationairport IS NOT MISSING
  LIMIT 100;
  ```

- {% include step_label.html %} Compara en el panel de resultados `resultSize`, `resultCount`, `elapsedTime` y `executionTime`, sin asumir que una ejecución aislada siempre será más rápida por efecto de caché y carga del sistema.

### Tarea 5.2. Crear y ejecutar la validación final

- {% include step_label.html %} Regresa a Git Bash y crea `validate-lab15.sh` para comprobar imagen, puerto, Analytics, datasets y archivos generados.

  {%raw%}
  ```bash
  cat > validate-lab15.sh << 'EOF'
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
    pass "Analytics ejecuta consultas"
  else
    fail "Estado Analytics: ${HEALTH_STATUS}"
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
    --data-urlencode 'client_context_id=lab15-validation-metadata-001')

  COLLECTIONS_OK=$(printf '%s' "${METADATA_RESPONSE}" \
    | python -c "
  import json,sys
  response=json.load(sys.stdin)
  names=set(response.get('results',[]))
  print('yes' if {'airports','routes','hotels'}.issubset(names) else 'no')
  " 2>/dev/null || echo no)

  if [ "${COLLECTIONS_OK}" = "yes" ]; then
    pass "TravelAnalytics contiene airports, routes y hotels"
  else
    fail "Faltan Analytics collections requeridas"
  fi

  for FILE in \
    builtin-string-functions.sqlpp \
    builtin-date-functions.sqlpp \
    builtin-array-functions.sqlpp \
    airline-rankings.sqlpp \
    airline-route-quartiles.sqlpp \
    airline-lag-lead.sqlpp \
    airline-first-last.sqlpp \
    airline-window-aggregates.sqlpp \
    explain-airline-ranking.sqlpp
  do
    if [ -s "${FILE}" ]; then
      pass "Existe ${FILE}"
    else
      fail "Falta ${FILE}"
    fi
  done

  echo
  echo "Resultado final: ${PASS} verificaciones correctas | ${FAIL} verificaciones fallidas"
  EOF
  ```
  {%endraw%}

- {% include step_label.html %} Asigna permiso de ejecución al script, porque los archivos creados con `cat` no siempre quedan marcados como ejecutables.

  ```bash
  chmod +x validate-lab15.sh
  ```

- {% include step_label.html %} Ejecuta el script desde `/c/LABS/couchbase-nosql/lab15` y revisa cada línea PASS o FAIL.

  ```bash
  ./validate-lab15.sh
  ```

- {% include step_label.html %} Si aparece un FAIL, corrige primero el archivo, variable, puerto o recurso indicado y vuelve a ejecutar el script hasta obtener todas las verificaciones correctas.

- {% include step_label.html %} Conserva todos los archivos `.sqlpp`, el JSON de health check y el script de validación para disponer de evidencia y poder repetir las consultas sin reconstruirlas.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

{% include support-prompt.html task="tarea5" %}

---

## 🛠️ Resolución de problemas

### Problema 1. Analytics Workbench no aparece en la Web Console

- {% include step_label.html %} Ejecuta `docker port couchbase-lab` desde Git Bash y confirma que el puerto 8095 esté publicado.

- {% include step_label.html %} Consulta `nodeServices` para verificar que el nodo tenga Analytics habilitado.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${CB_URL}/pools/default/nodeServices" \
    | python -m json.tool
  ```

### Problema 2. No existen airports, routes o hotels

- {% include step_label.html %} Ejecuta la consulta de Metadata.Dataset desde Analytics Workbench y confirma los nombres existentes.

- {% include step_label.html %} Si faltan recursos, completa nuevamente la práctica 14; no crees datasets con nombres alternativos.

### Problema 3. Una función integrada devuelve error

- {% include step_label.html %} Copia únicamente la expresión problemática en una consulta SELECT mínima para aislar si el error pertenece al nombre de la función, a los argumentos o al tipo de dato.

- {% include step_label.html %} Revisa que se utilicen UPPER, LENGTH, TRIM, SPLIT, REGEXP_CONTAINS, COALESCE, NOW_STR, DATE_DIFF_STR, DATE_ADD_STR, ARRAY_DISTINCT y ARRAY_FLATTEN.

### Problema 4. RANK y DENSE_RANK devuelven los mismos valores

- {% include step_label.html %} Comprueba si realmente existen empates en total_routes.

- {% include step_label.html %} Ejecuta el ejemplo literal de la Tarea 3.1 para demostrar el comportamiento con un empate garantizado.

### Problema 5. LAG o LEAD devuelve NULL

- {% include step_label.html %} Recuerda que la primera fila no tiene fila anterior y la última no tiene fila siguiente; NULL representa correctamente esa ausencia.

### Problema 6. LAST_VALUE devuelve el valor actual

- {% include step_label.html %} Comprueba que la función incluya el marco completo:

  ```text
  ROWS BETWEEN UNBOUNDED PRECEDING
           AND UNBOUNDED FOLLOWING
  ```

### Problema 7. EXPLAIN no muestra exactamente los mismos operadores

- {% include step_label.html %} No busques nombres rígidos; identifica las etapas conceptuales de lectura, filtro, agregación, ordenamiento, ventana y entrega.

### Problema 8. La terminal no reconoce python

- {% include step_label.html %} Ejecuta ambos comandos para identificar la instalación disponible.

  ```bash
  python --version
  python3 --version
  ```

- {% include step_label.html %} Sustituye `python` por `python3` en los comandos cuando esa sea la instalación disponible.