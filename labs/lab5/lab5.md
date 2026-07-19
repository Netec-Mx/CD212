---
layout: lab
title: "Práctica 5: Análisis de consultas con EXPLAIN y optimización básica"
permalink: /lab5/lab5/
images_base: /labs/lab5/img
duration: "80 minutos"
objective:
  - Preparar el directorio de trabajo de la práctica 5 y validar Couchbase Server Enterprise Edition.
  - Habilitar métricas y aplicar una metodología consistente de medición.
  - Interpretar EXPLAIN e identificar PrimaryScan3, IndexScan3, Fetch, Filter, Order y Aggregate.
  - Comparar escaneos amplios con índices compuestos y funcionales.
  - Crear índices cubrientes para reducir accesos al Data Service.
  - Optimizar ORDER BY, agregaciones, índices compuestos y JOIN ON KEYS.
  - Reconocer cuándo una búsqueda debe resolverse con Full Text Search.
prerequisites:
  - Haber completado la Práctica 4.
  - Tener Docker Desktop en ejecución.
  - Tener activo el contenedor couchbase-lab creado con la imagen couchbase/server:enterprise-7.6.2.
  - Tener cargado el bucket travel-sample.
  - Tener acceso a la Web Console en http://localhost:8091.
  - Utilizar Visual Studio Code con Git Bash como terminal integrada.
introduction:
  - En esta práctica aplicarás un ciclo básico de optimización sobre consultas SQL++ del dataset travel-sample en Couchbase Server Enterprise Edition. Para cada caso ejecutarás una consulta, revisarás su plan con EXPLAIN, identificarás operadores costosos, crearás un índice adecuado y validarás el cambio. La evidencia principal será el cambio del plan y no un porcentaje fijo de mejora.
slug: lab5
lab_number: 5
final_result: >
  Al finalizar la práctica habrás diagnosticado y optimizado consultas SQL++ sobre las collections hotel, route y airline. Podrás reconocer escaneos amplios, validar índices cubrientes, optimizar ordenamientos y agregaciones, y mejorar un JOIN ON KEYS mediante un índice adecuado.
notes:
  - Todos los comandos de terminal deben ejecutarse desde Git Bash dentro de Visual Studio Code.
  - No elimines ni detengas el contenedor couchbase-lab al finalizar la práctica.
  - Los índices creados usan el prefijo idx_lab5_.
  - Ejecuta cada consulta una vez para calentar caché y después tres veces; utiliza la mediana de executionTime.
  - Los tiempos pueden variar. Prioriza la evidencia del plan sobre el tiempo absoluto.
references: []
prev: /lab4/lab4/
next: /lab6/lab6/
---

---
<!-- Aquí comienzan las instrucciones paso a paso de la práctica -->

## 📁 Preparación del directorio de trabajo

En esta práctica conservarás el directorio raíz del curso y crearás únicamente el subdirectorio `lab5`.

### 🗂️ Crear y abrir el subdirectorio

- {% include step_label.html %} Inicia **Docker Desktop** desde el menú Inicio de Windows y espera a que el panel principal indique que el motor de Docker está en ejecución.
- {% include step_label.html %} Inicia **Visual Studio Code** y selecciona **File > Open Folder...**. En el selector de carpetas, escribe o navega hasta la siguiente ruta y pulsa **Select Folder**.

  ```text
  C:\LABS\couchbase-nosql
  ```

- {% include step_label.html %} En Visual Studio Code, abre **Terminal > New Terminal**. En la flecha situada junto al botón **+** del panel de terminal, selecciona **Git Bash** y confirma que el prompt utiliza una ruta con formato `/c/...`.
- {% include step_label.html %} En la terminal Git Bash, ejecuta los siguientes comandos. Esta acción permite crear de forma idempotente el directorio `/c/LABS/couchbase-nosql/lab5` donde se organizarán los archivos de esta práctica.

  ```bash
  mkdir -p /c/LABS/couchbase-nosql/lab5
  cd /c/LABS/couchbase-nosql/lab5
  pwd
  ```

**Salida esperada:**

```text
/c/LABS/couchbase-nosql/lab5
```

**Verificación:** La ruta mostrada por `pwd` debe terminar exactamente en `/lab5`. Si permanece en otra carpeta, vuelve a ejecutar el comando `cd` antes de continuar; de lo contrario, el archivo `comparacion-consultas.md` podría guardarse fuera de la práctica.

---

## 🧭 Tarea 1. Preparar el entorno, habilitar métricas e interpretar EXPLAIN

En esta tarea validarás Couchbase, habilitarás las métricas de consulta y reconocerás los principales operadores de un plan.

### Tarea 1.1. Verificar Couchbase Server

- {% include step_label.html %} Desde Git Bash, consulta el estado del contenedor con el siguiente comando. Esta acción permite consultar los contenedores y comprobar que `couchbase-lab` permanezca activo antes de utilizar sus servicios.

  {%raw%}
  ```bash
  docker ps --filter "name=couchbase-lab" --format "table {{.Names}}\t{{.Status}}"
  ```
  {%endraw%}

- {% include step_label.html %} Revisa la columna `STATUS`. Si no aparece ninguna fila o el contenedor está detenido, ejecuta el siguiente comando para iniciarlo sin recrearlo.

  ```bash
  docker start couchbase-lab
  ```

- {% include step_label.html %} Comprueba desde la terminal que el servicio de administración de Couchbase responde por el puerto `8091`. Esta acción permite solicitar la Web Console por el puerto 8091 e interpretar el código HTTP como prueba de.

  ```bash
  curl -s -o /dev/null -w "Web Console: HTTP %{http_code}\n" \
    http://localhost:8091/ui/index.html
  ```

**Salida esperada:**

```text
Web Console: HTTP 200
```

**Verificación:** El código `HTTP 200` confirma que la Web Console respondió correctamente. Un código distinto o la ausencia de respuesta indica que el contenedor todavía está iniciando, que el puerto no está publicado o que el servicio de administración presenta un problema.

### Tarea 1.2. Validar que el contenedor utiliza Enterprise Edition

- {% include step_label.html %} Inspecciona la configuración efectiva del contenedor para identificar la imagen con la que fue creado. Esta acción permite consultar la configuración de `couchbase-lab` y verificar que utiliza la imagen Enterprise.

  {%raw%}
  ```bash
  docker inspect couchbase-lab --format "Imagen activa: {{.Config.Image}}"
  ```
  {%endraw%}

**Salida esperada:**

```text
Imagen activa: couchbase/server:enterprise-7.6.2
```

> **IMPORTANTE:** La salida debe indicar exactamente `couchbase/server:enterprise-7.6.2`. Si aparece `couchbase/server:community-7.6.2`, no basta con haber descargado la imagen Enterprise: el contenedor anterior de Community Edition continúa siendo el que está activo. Vuelve a la Práctica 2, elimina y recrea el contenedor con Enterprise Edition antes de continuar, porque algunas capacidades y opciones de análisis utilizadas en el curso dependen de esa edición.
{: .lab-note .important .compact}

### Tarea 1.3. Validar el dataset

- {% include step_label.html %} Abre un navegador web y entra en `http://localhost:8091`. Espera a que se muestre la pantalla de autenticación o el panel principal de Couchbase.
- {% include step_label.html %} En la pantalla de acceso, escribe `Administrator` en **Username** y `Password123!` en **Password**, y después selecciona **Sign In**.
- {% include step_label.html %} En el menú lateral izquierdo de la Web Console, selecciona **Query** para abrir **Query Workbench**. Verifica que se muestre el editor SQL++ en el panel central, pega la consulta siguiente y pulsa **Execute**.

  ```sql
  SELECT RAW COUNT(*)
  FROM `travel-sample`.inventory.hotel;
  ```

**Validación:** En la pestaña **JSON** o **Results** debe mostrarse un arreglo con un único número mayor que `0`. Esto confirma que `travel-sample.inventory.hotel` contiene documentos y que Query Service puede consultar la collection. Un resultado `0`, un error de keyspace o un mensaje de índice inexistente debe resolverse antes de continuar.

### Tarea 1.4. Habilitar métricas

- {% include step_label.html %} Dentro de **Query Workbench**, localiza el botón o icono **Query Preferences** situado en la barra de herramientas del editor. Ábrelo y configura las siguientes opciones.

  ```text
  Metrics: true
  Profile: timings
  ```

- {% include step_label.html %} Cierra o aplica las preferencias y ejecuta la siguiente consulta desde el editor. Esta acción permite consultar ``travel-sample`.inventory.hotel` con el filtro `country = "United States"` y comprobar las filas y el.

  ```sql
  SELECT name, city, country
  FROM `travel-sample`.inventory.hotel
  WHERE country = "United States"
  LIMIT 5;
  ```

- {% include step_label.html %} Después de ejecutar la consulta, abre la sección **Metrics** o revisa el bloque de métricas mostrado debajo del resultado.

> **IMPORTANTE:** Para comparar consultas usa `executionTime`, no una única medición aislada. Ejecuta primero una vez para cargar páginas, índices y metadatos en memoria; después ejecuta tres veces adicionales y ordena los tres valores de menor a mayor. La mediana es el valor central y reduce el efecto de variaciones ocasionadas por caché, procesos del equipo o actividad interna del clúster.
{: .lab-note .important .compact}

### Tarea 1.5. Ejecutar EXPLAIN

- {% include step_label.html %} Ejecuta el `EXPLAIN` en Query Workbench para identificar índice, spans y operadores; utiliza el JSON como evidencia de la comparación.

  ```sql
  EXPLAIN
  SELECT name, city, country
  FROM `travel-sample`.inventory.hotel
  WHERE country = "United States"
  LIMIT 5;
  ```

| Operador | Interpretación |
|---|---|
| `PrimaryScan3` | Escaneo amplio mediante índice primario |
| `IndexScan3` | Escaneo mediante índice secundario |
| `Fetch` | Recuperación del documento desde Data Service |
| `Filter` | Aplicación de condiciones residuales |
| `Order` | Ordenamiento fuera del índice |
| `Aggregate` | Procesamiento de agregaciones |
| `Limit` | Restricción del número de resultados |

> **NOTA:** Lee el plan como evidencia del entorno real. El optimizador puede elegir índices distintos según los índices creados en prácticas anteriores, sus condiciones parciales y las estadísticas disponibles. No asumas que siempre aparecerá `PrimaryScan3`; identifica el nombre del índice utilizado y determina si el plan sigue realizando un escaneo amplio, un `Fetch` o un `Filter` residual.
{: .lab-note .info .compact}

### Tarea 1.6. Crear la tabla comparativa

- {% include step_label.html %} En el panel **Explorer** de Visual Studio Code, asegúrate de estar dentro de la carpeta `lab5`, selecciona **New File** y crea `comparacion-consultas.md`.

```markdown
| Caso | Plan inicial | Optimización | Plan final | Mediana inicial | Mediana final | Evidencia |
|---|---|---|---|---:|---:|---|
```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

{% include support-prompt.html task="tarea1" %}

---

## 🔍 Tarea 2. Comparar escaneo amplio, índice compuesto y función en WHERE

### Tarea 2.1. Analizar una consulta sin índice específico

- {% include step_label.html %} Ejecuta el `EXPLAIN` en Query Workbench para identificar índice, spans y operadores; utiliza el JSON como evidencia de la comparación. Aquí cubre la primera parte de `Analizar una consulta sin índice específico`.

  ```sql
  EXPLAIN
  SELECT name, city, country, avg_rating
  FROM `travel-sample`.inventory.hotel
  WHERE avg_rating > 4.5
    AND free_breakfast = true;
  ```

- {% include step_label.html %} En el JSON del plan, busca el nombre del índice y los operadores principales. Esta acción permite completar `Analizar una consulta sin índice específico` y verificar su efecto dentro de la tarea `Comparar escaneo.
- {% include step_label.html %} Retira la palabra `EXPLAIN` y ejecuta la consulta real una vez para calentar caché. Luego ejecútala tres veces adicionales, copia cada `executionTime`, calcula la mediana y regístrala como medición inicial.

### Tarea 2.2. Crear un índice compuesto

- {% include step_label.html %} Ejecuta la siguiente sentencia en Query Workbench para crear un índice secundario compuesto. Esta acción permite crear el índice `idx_lab5_hotel_rating_breakfast` para el patrón de consulta analizado.

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab5_hotel_rating_breakfast
  ON `travel-sample`.inventory.hotel(avg_rating, free_breakfast)
  WHERE avg_rating IS NOT MISSING;
  ```

- {% include step_label.html %} Ejecuta la consulta de metadatos siguiente para confirmar que el índice existe, conocer sus claves y revisar la condición parcial.

  ```sql
  SELECT name, state, `index_key`, condition
  FROM system:indexes
  WHERE name = "idx_lab5_hotel_rating_breakfast";
  ```

### Tarea 2.3. Comparar el nuevo plan

- {% include step_label.html %} Vuelve a ejecutar exactamente la consulta de la Tarea 2.1 precedida de `EXPLAIN`, sin cambiar filtros ni proyección.

**Validación:**

- Debe utilizar `idx_lab5_hotel_rating_breakfast` o un índice compatible.
- El escaneo debe ser más selectivo.
- Puede continuar apareciendo `Fetch` porque `name`, `city` y `country` no están en el índice.

### Tarea 2.4. Analizar una función en WHERE

- {% include step_label.html %} Ejecuta el `EXPLAIN` en Query Workbench para identificar índice, spans y operadores; utiliza el JSON como evidencia de la comparación. Aquí cubre la primera parte de `Analizar una función en WHERE`.

  ```sql
  EXPLAIN
  SELECT name, city, country
  FROM `travel-sample`.inventory.hotel
  WHERE LOWER(name) = "the westin new york grand central";
  ```

- {% include step_label.html %} Examina el plan y localiza la expresión `LOWER(name)`. Esta acción permite completar `Analizar una función en WHERE` y verificar su efecto dentro de la tarea `Comparar escaneo amplio.

### Tarea 2.5. Crear un índice funcional

- {% include step_label.html %} Crea el siguiente índice funcional. Esta acción permite crear el índice `idx_lab5_hotel_name_lower` para el patrón de consulta analizado.

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab5_hotel_name_lower
  ON `travel-sample`.inventory.hotel(LOWER(name));
  ```

- {% include step_label.html %} Consulta `system:indexes` para verificar que `idx_lab5_hotel_name_lower` esté `online`; después vuelve a ejecutar el mismo `EXPLAIN`.

**Validación:** El plan debe mencionar `idx_lab5_hotel_name_lower` o un índice funcional compatible y mostrar que la igualdad sobre `LOWER(name)` se utiliza para delimitar el escaneo. Si el índice aparece pero la expresión continúa únicamente dentro de `Filter`, revisa que la función y el valor usados en la consulta coincidan exactamente con la expresión definida en el índice.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

{% include support-prompt.html task="tarea2" %}

---

## 🧱 Tarea 3. Comparar SELECT *, proyección e índice cubriente

### Tarea 3.1. Analizar SELECT *

- {% include step_label.html %} Ejecuta el siguiente `EXPLAIN` para establecer el plan base de una consulta que solicita el documento completo. Esta versión sirve como contraste frente a una proyección limitada y un índice cubriente

  ```sql
  EXPLAIN
  SELECT *
  FROM `travel-sample`.inventory.hotel
  WHERE country = "United States"
    AND avg_rating > 4.0
  LIMIT 20;
  ```

- {% include step_label.html %} Revisa el plan y confirma la presencia de `Fetch`. Esta acción permite completar `Analizar SELECT *` y verificar su efecto dentro de la tarea `Comparar SELECT *, proyección e índice cubriente`.

### Tarea 3.2. Crear un índice cubriente

- {% include step_label.html %} Crea el siguiente índice con las claves de filtrado al inicio y los campos proyectados después. Esta acción permite crear el índice `idx_lab5_hotel_country_rating_cover` para el patrón de consulta analizado.

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab5_hotel_country_rating_cover
  ON `travel-sample`.inventory.hotel(country, avg_rating, name, city)
  WHERE country IS NOT MISSING;
  ```

- {% include step_label.html %} Ejecuta una consulta sobre `system:indexes` para `idx_lab5_hotel_country_rating_cover` y confirma que `state` sea `online` antes de probar el nuevo plan.

### Tarea 3.3. Usar proyección específica

- {% include step_label.html %} Ejecuta la versión con proyección específica. Esta acción permite obtener el plan de ejecución e identificar el índice, los spans y los operadores seleccionados por Query.

  ```sql
  EXPLAIN
  SELECT name, city, avg_rating
  FROM `travel-sample`.inventory.hotel
  WHERE country = "United States"
    AND avg_rating > 4.0
  LIMIT 20;
  ```

### Tarea 3.4. Validar cobertura

- {% include step_label.html %} En el resultado de `EXPLAIN`, usa la búsqueda del navegador o del panel de resultados para localizar los siguientes elementos.

Busca:

- `IndexScan3`.
- `idx_lab5_hotel_country_rating_cover`.
- Propiedades `covers`.
- Ausencia de `Fetch`.

> **IMPORTANTE:** La ausencia de `Fetch`, junto con la presencia de `covers`, demuestra que todos los campos requeridos por `SELECT` y `WHERE` están disponibles en el índice. Si `Fetch` continúa apareciendo, revisa que la consulta proyecte únicamente `name`, `city` y `avg_rating`, y que el plan realmente utilice `idx_lab5_hotel_country_rating_cover`.
{: .lab-note .important .compact}

### Tarea 3.5. Comparar ejecuciones

- {% include step_label.html %} Ejecuta primero la consulta con `SELECT *` una vez para calentar caché y tres veces para medir; después repite el mismo procedimiento con la proyección específica.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

{% include support-prompt.html task="tarea3" %}

---

## 📊 Tarea 4. Optimizar ORDER BY, agregaciones y orden de campos

### Tarea 4.1. Analizar ORDER BY

- {% include step_label.html %} Ejecuta el siguiente `EXPLAIN` para determinar si el ordenamiento por `avg_rating DESC` puede resolverse durante el recorrido del índice o si Query Service debe ordenar posteriormente los resultados

  ```sql
  EXPLAIN
  SELECT name, avg_rating, city
  FROM `travel-sample`.inventory.hotel
  WHERE country = "France"
  ORDER BY avg_rating DESC
  LIMIT 10;
  ```

- {% include step_label.html %} En el plan, busca el operador `Order`. Su presencia indica que Query Service debe ordenar filas después del escaneo porque el índice elegido no entrega los registros en el orden requerido por `ORDER BY avg_rating DESC`

### Tarea 4.2. Crear un índice con orden compatible

- {% include step_label.html %} Crea un índice cuya primera clave soporte la igualdad por `country` y cuya segunda clave conserve `avg_rating` en dirección descendente.

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab5_hotel_country_rating_desc
  ON `travel-sample`.inventory.hotel(country, avg_rating DESC, name);
  ```

- {% include step_label.html %} Repite el `EXPLAIN` de la consulta anterior sin cambiar el `WHERE`, el `ORDER BY` ni el `LIMIT`. Compara si el nuevo índice se selecciona y si el operador `Order` desaparece o el orden se incorpora al escaneo del índice

**Validación:** El plan debe utilizar `idx_lab5_hotel_country_rating_desc` o un índice compatible. Puede eliminar el operador `Order` o reflejar que el orden se obtiene durante el escaneo del índice. `Fetch` puede continuar apareciendo porque `city` no está incluido; esto no invalida la optimización del ordenamiento, pero demuestra que la consulta todavía necesita recuperar ese campo desde el documento.

### Tarea 4.3. Analizar una agregación

- {% include step_label.html %} Ejecuta el siguiente `EXPLAIN` para observar cómo Couchbase obtiene los campos, agrupa por país, calcula las funciones agregadas, aplica `HAVING` y ordena el resultado.

  ```sql
  EXPLAIN
  SELECT country,
         COUNT(*) AS total_hoteles,
         AVG(avg_rating) AS promedio,
         MAX(avg_rating) AS maxima,
         MIN(avg_rating) AS minima
  FROM `travel-sample`.inventory.hotel
  WHERE country IS NOT MISSING
    AND avg_rating IS NOT MISSING
  GROUP BY country
  HAVING COUNT(*) > 5
  ORDER BY promedio DESC;
  ```

- {% include step_label.html %} Recorre la secuencia de operadores del plan y verifica si aparece `Fetch` antes de `Aggregate`. Esta acción permite completar `Analizar una agregación` y verificar su efecto dentro de la tarea `Optimizar ORDER BY.

### Tarea 4.4. Crear un índice cubriente para agregación

- {% include step_label.html %} Crea un índice parcial que contenga los dos campos necesarios para filtrar, agrupar y calcular las métricas. Esta acción permite crear el índice `idx_lab5_hotel_aggregate_cover` para el patrón de consulta analizado.

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab5_hotel_aggregate_cover
  ON `travel-sample`.inventory.hotel(country, avg_rating)
  WHERE country IS NOT MISSING
    AND avg_rating IS NOT MISSING;
  ```

- {% include step_label.html %} Ejecuta nuevamente el `EXPLAIN` de la agregación de la Tarea 4.3, conservando exactamente el mismo `WHERE`, `GROUP BY`, `HAVING` y `ORDER BY`.

**Validación:** El plan debe usar `idx_lab5_hotel_aggregate_cover` o un índice compatible, mostrar `country` y `avg_rating` como claves cubiertas y evitar la recuperación de documentos completos. La agregación y el ordenamiento pueden seguir apareciendo porque forman parte del trabajo lógico solicitado; la optimización buscada es eliminar lecturas innecesarias del Data Service.

### Tarea 4.5. Comparar el orden de campos

- {% include step_label.html %} Crea dos índices con las mismas propiedades pero en orden inverso. Esta acción permite crear el índice `idx_lab5_hotel_wrong_order` para el patrón de consulta analizado.

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab5_hotel_wrong_order
  ON `travel-sample`.inventory.hotel(avg_rating, country);

  CREATE INDEX IF NOT EXISTS idx_lab5_hotel_correct_order
  ON `travel-sample`.inventory.hotel(country, avg_rating);
  ```

- {% include step_label.html %} Ejecuta los dos planes con `USE INDEX` para forzar temporalmente cada alternativa. Esta acción permite obtener el plan de ejecución e identificar el índice, los spans y los operadores seleccionados por Query.

  ```sql
  EXPLAIN
  SELECT name, avg_rating
  FROM `travel-sample`.inventory.hotel
  USE INDEX (idx_lab5_hotel_wrong_order USING GSI)
  WHERE country = "Spain"
    AND avg_rating BETWEEN 3.5 AND 5.0
  ORDER BY avg_rating;
  ```

Durante `Comparar el orden de campos`, ejecuta este bloque en Query Workbench para obtener el plan de ejecución e identificar el índice, los spans y los operadores seleccionados por Query; conserva las líneas y revisa la respuesta antes de continuar.

  ```sql
  EXPLAIN
  SELECT name, avg_rating
  FROM `travel-sample`.inventory.hotel
  USE INDEX (idx_lab5_hotel_correct_order USING GSI)
  WHERE country = "Spain"
    AND avg_rating BETWEEN 3.5 AND 5.0
  ORDER BY avg_rating;
  ```

**Validación:** En el índice correcto, el plan debe representar `country = "Spain"` como una igualdad sobre la primera clave y `avg_rating BETWEEN 3.5 AND 5.0` como un rango sobre la segunda. El índice con orden invertido no puede aprovechar de la misma manera una primera clave sin igualdad. Después elimina `USE INDEX` y ejecuta `EXPLAIN` nuevamente para comprobar qué alternativa selecciona el optimizador por sí mismo.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

{% include support-prompt.html task="tarea4" %}

---

## 🔗 Tarea 5. Optimizar JOIN ON KEYS y reconocer el límite de LIKE

### Tarea 5.1. Analizar el JOIN

- {% include step_label.html %} Ejecuta el siguiente `EXPLAIN` para revisar cómo Couchbase filtra documentos de `route` y después utiliza `r.airlineid` como document key para recuperar el documento relacionado de `airline`.

  ```sql
  EXPLAIN
  SELECT r.sourceairport,
         r.destinationairport,
         r.distance,
         a.name AS airline_name
  FROM `travel-sample`.inventory.route AS r
  JOIN `travel-sample`.inventory.airline AS a
    ON KEYS r.airlineid
  WHERE r.sourceairport = "SFO"
    AND r.distance > 1000
  LIMIT 20;
  ```

- {% include step_label.html %} Examina por separado las dos ramas del plan. En `route`, identifica el índice utilizado y si `sourceairport` y `distance` se resuelven como spans o filtros.

### Tarea 5.2. Crear un índice para route

- {% include step_label.html %} Crea el siguiente índice compuesto sobre `route`. Esta acción permite crear el índice `idx_lab5_route_source_distance` para el patrón de consulta analizado.

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab5_route_source_distance
  ON `travel-sample`.inventory.route(
    sourceairport,
    distance,
    airlineid,
    destinationairport
  )
  WHERE sourceairport IS NOT MISSING;
  ```

- {% include step_label.html %} Consulta `system:indexes` usando el nombre `idx_lab5_route_source_distance` y confirma que el estado sea `online`.

### Tarea 5.3. Comparar el nuevo plan

- {% include step_label.html %} Ejecuta nuevamente el `EXPLAIN` de la Tarea 5.1 sin cambiar la proyección, el `JOIN`, los filtros ni el `LIMIT`.

**Validación:** La rama `route` debe usar `idx_lab5_route_source_distance` o un índice compatible y representar `sourceairport = "SFO"` como igualdad, seguido del rango `distance > 1000`. La rama `airline` debe continuar recuperándose mediante `ON KEYS r.airlineid`; esa recuperación por clave es el comportamiento esperado y no un escaneo que deba eliminarse.

> **NOTA:** No necesitas crear un índice secundario sobre `META(a).id`. `ON KEYS r.airlineid` utiliza directamente la document key administrada por el Data Service, por lo que agregar un índice GSI sobre esa misma identidad no mejoraría este acceso y solo consumiría almacenamiento y recursos de mantenimiento.
{: .lab-note .info .compact}

### Tarea 5.4. Analizar LIKE con wildcard inicial

- {% include step_label.html %} Ejecuta el siguiente `EXPLAIN` para analizar una búsqueda de subcadena. Esta acción permite obtener el plan de ejecución e identificar el índice, los spans y los operadores seleccionados por Query.

  ```sql
  EXPLAIN
  SELECT name, city, country
  FROM `travel-sample`.inventory.hotel
  WHERE name LIKE "%Marriott%";
  ```

- {% include step_label.html %} Crea el siguiente índice sobre `name`. Esta acción permite crear el índice `idx_lab5_hotel_name_prefix` para el patrón de consulta analizado.

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab5_hotel_name_prefix
  ON `travel-sample`.inventory.hotel(name)
  WHERE name IS NOT MISSING;
  ```

- {% include step_label.html %} Ejecuta el siguiente `EXPLAIN` y compáralo con el anterior. En `"Marriott%"`, el prefijo conocido permite establecer un límite inferior y superior aproximado dentro del índice, a diferencia de `"%Marriott%"`

  ```sql
  EXPLAIN
  SELECT name, city, country
  FROM `travel-sample`.inventory.hotel
  WHERE name LIKE "Marriott%";
  ```

**Interpretación:**

- `LIKE "Marriott%"` puede aprovechar un rango por prefijo.
- `LIKE "%Marriott%"` no puede formar un rango selectivo desde el inicio.
- Las búsquedas de palabras, relevancia o subcadenas deben resolverse con Full Text Search.

### Tarea 5.5. Consolidar resultados

- {% include step_label.html %} Regresa a `comparacion-consultas.md` y completa una fila por cada caso. Esta acción permite aplicar el bloque que comienza con `| Caso | Plan inicial | Optimización | Plan final | Mediana inicial | Mediana final |.

```markdown
| Caso | Plan inicial | Optimización | Plan final | Mediana inicial | Mediana final | Evidencia |
|---|---|---|---|---:|---:|---|
| Filtro compuesto | Escaneo amplio | Índice compuesto | IndexScan3 | | | Menor escaneo |
| LOWER() | Filter residual | Índice funcional | Span funcional | | | Función indexada |
| SELECT * | Fetch | Índice cubriente | Sin Fetch | | | Covers |
| ORDER BY | Order | Índice ordenado | Orden del índice | | | Sin Order o pushdown |
| Agregación | Fetch + Aggregate | Índice cubriente | Aggregate sobre índice | | | Sin Fetch |
| JOIN ON KEYS | Rama route amplia | Índice en route | Escaneo selectivo | | | ON KEYS |
| LIKE inicial | Filtro amplio | Prefijo o FTS | Rango por prefijo | | | Puente a FTS |
```

### Tarea 5.6. Validar índices y Query Service

- {% include step_label.html %} En Query Workbench, ejecuta la siguiente consulta final sobre `system:indexes`. Esta acción permite consultar `system:indexes` y revisar los índices cuyo nombre coincide con `idx_lab5_%`.

  ```sql
  SELECT name,
         keyspace_id AS collection_name,
         state,
         `index_key`,
         condition
  FROM system:indexes
  WHERE bucket_id = "travel-sample"
    AND scope_id = "inventory"
    AND name LIKE "idx_lab5_%"
  ORDER BY collection_name, name;
  ```

- {% include step_label.html %} Revisa cada fila devuelta por la consulta. Esta acción permite completar `Validar índices y Query Service` y verificar su efecto dentro de la tarea `Optimizar JOIN ON KEYS y reconocer el límite de LIKE`.
- {% include step_label.html %} Regresa a la terminal **Git Bash** de Visual Studio Code y ejecuta la siguiente solicitud HTTP contra Query Service en el puerto `8093`.

  ```bash
  curl -s -u Administrator:Password123! \
    http://localhost:8093/query/service \
    --data-urlencode 'statement=SELECT name, keyspace_id, state FROM system:indexes WHERE bucket_id = "travel-sample" AND scope_id = "inventory" AND name LIKE "idx_lab5_%" ORDER BY name' \
    | python -m json.tool | grep -E '"name"|"keyspace_id"|"state"|"status"'
  ```

**Salida esperada:** deben aparecer los nombres de los índices creados, su collection correspondiente, `state` con valor `online` y `status` con valor `success`. `status: success` confirma que Query Service procesó correctamente la solicitud REST; no sustituye la revisión individual del estado de cada índice.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

{% include support-prompt.html task="tarea5" %}

---

## 🛠️ Resolución de problemas

### Problema 1. EXPLAIN usa otro índice

El optimizador evalúa todos los índices elegibles y puede seleccionar uno creado en prácticas anteriores. Esto no es necesariamente un error. Primero inventaría los índices disponibles y compara sus claves y condiciones con la consulta:

```sql
SELECT name, keyspace_id, state, `index_key`, condition
FROM system:indexes
WHERE bucket_id = "travel-sample"
  AND scope_id = "inventory"
ORDER BY keyspace_id, name;
```

Usa `USE INDEX` únicamente para aislar una comparación didáctica. Después retira el hint y ejecuta nuevamente `EXPLAIN`, porque una optimización válida debe comprobar también la decisión normal del optimizador.

### Problema 2. El índice no está online

Un índice que no está `online` no puede ser seleccionado por Query Service. Consulta su estado exacto antes de repetir la consulta:

```sql
SELECT name, state
FROM system:indexes
WHERE name LIKE "idx_lab5_%"
ORDER BY name;
```

Si está `deferred`, significa que fue creado con construcción aplazada y debes ejecutar `BUILD INDEX` sobre la collection correspondiente. Si está `building`, espera y vuelve a consultar el estado; no continúes la comparación hasta que cambie a `online`.

### Problema 3. Los tiempos varían demasiado

Los tiempos de una instalación local pueden cambiar por caché, uso de CPU, memoria, Docker Desktop y procesos de Windows. Para obtener una comparación reproducible:

1. Ejecuta una vez para calentar caché.
2. Ejecuta tres veces adicionales.
3. Usa `executionTime`.
4. Calcula la mediana.
5. Prioriza cambios del plan.

### Problema 4. Sigue apareciendo Fetch

`Fetch` indica que el índice no contiene todo lo necesario para completar la consulta. Algún campo del `SELECT`, `WHERE`, `ORDER BY` o `JOIN` no está en el índice. Revisa:

```sql
SELECT name, `index_key`, condition
FROM system:indexes
WHERE name = "nombre_del_indice";
```

### Problema 5. El plan no elimina Order

Que exista un índice con los mismos campos no garantiza que pueda satisfacer el ordenamiento. Verifica:

- Campos de igualdad primero.
- Campo de orden después de las igualdades.
- Misma dirección `ASC` o `DESC`.
- Ausencia de un rango anterior que impida aprovechar el orden.

### Problema 6. El contenedor activo utiliza Community Edition

**Síntoma:** la validación de imagen muestra `couchbase/server:community-7.6.2`.

**Solución:**

Para resolver `El contenedor activo utiliza Community Edition`, aplica el diagnóstico en el componente indicado, interpreta la respuesta y confirma la recuperación antes de retomar la práctica.

{%raw%}
```bash
docker inspect couchbase-lab   --format "Imagen activa: {{.Config.Image}}"
```
{%endraw%}

La salida correcta debe ser:

```text
Imagen activa: couchbase/server:enterprise-7.6.2
```

Si continúa apareciendo Community Edition, vuelve a la Práctica 2 y recrea el contenedor con la imagen Enterprise antes de ejecutar esta práctica.

### Problema 7. `python -m json.tool` falla

El comando REST utiliza el módulo estándar `json.tool` únicamente para dar formato legible a la respuesta. Comprueba qué ejecutable de Python está disponible en Git Bash:

```bash
python --version
python3 --version
```

Si `python3` funciona, reemplaza `python -m json.tool` por `python3 -m json.tool`.
