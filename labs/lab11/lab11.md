---
layout: lab
title: "Práctica 11: Implementación de consultas Search compuestas e integración con SQL++"
permalink: /lab11/lab11/
images_base: /labs/lab11/img
duration: "70 minutos"
objective:
  - Validar que el entorno utilice Couchbase Server Enterprise 7.6.2 y que los servicios requeridos estén disponibles.
  - Reutilizar el índice scoped idx_lab10_hotel_search creado en la práctica 10 sin duplicar recursos.
  - Ejecutar Query String Queries con operadores obligatorios, exclusiones, alternativas, campos y wildcards.
  - Comparar Term Query y Match Query para comprender el impacto del analizador configurado en cada field.
  - Construir consultas compuestas mediante conjuncts y disjuncts con control de coincidencias mínimas.
  - Implementar paginación estable con from, size y sort mediante la Search REST API.
  - Integrar Couchbase Search con SQL++ usando SEARCH() y SEARCH_SCORE().
prerequisites:
  - Haber completado la Práctica 10.
  - Tener Docker Desktop en ejecución.
  - Tener activo el contenedor couchbase-lab.
  - Utilizar la imagen couchbase/server:enterprise-7.6.2.
  - Tener cargado el bucket travel-sample.
  - Tener disponible la collection travel-sample.inventory.hotel.
  - Tener creado el índice idx_lab10_hotel_search.
  - Tener acceso a la Web Console en http://localhost:8091.
  - Utilizar Visual Studio Code con Git Bash como terminal integrada.
introduction:
  - En esta práctica profundizarás en Couchbase Search reutilizando el índice scoped creado anteriormente. Ejecutarás Query String Queries, compararás Term Query frente a Match Query, combinarás condiciones con conjuncts y disjuncts, implementarás paginación estable mediante la REST API y finalmente integrarás Search con SQL++ usando SEARCH() y SEARCH_SCORE().
slug: lab11
lab_number: 11
final_result: >
  Al finalizar la práctica habrás ejecutado búsquedas avanzadas sobre idx_lab10_hotel_search, utilizado operadores Query String, diferenciado Term Query de Match Query, construido consultas conjunction y disjunction, paginado resultados de forma estable e integrado Couchbase Search con SQL++ para combinar relevancia, filtros estructurados y agregaciones.
notes:
  - Todos los comandos de terminal deben ejecutarse desde Git Bash dentro de Visual Studio Code.
  - La imagen utilizada es couchbase/server:enterprise-7.6.2.
  - Utiliza las credenciales Administrator y Password123! configuradas en las prácticas anteriores.
  - Esta práctica reutiliza idx_lab10_hotel_search y no crea un índice adicional.
  - Todos los endpoints REST utilizan la ruta scoped de Search.
  - No elimines el índice al finalizar porque forma parte de la continuidad del curso.
references: []
prev: /lab10/lab10/
next: /lab12/lab12/
---

---
<!-- Aquí comienzan las instrucciones paso a paso de la práctica -->

## 📁 Preparación del directorio de trabajo

### Preparación inicial

- {% include step_label.html %} Abre Docker Desktop y espera a que el motor quede completamente disponible antes de ejecutar cualquier validación del contenedor.

- {% include step_label.html %} Abre Visual Studio Code y selecciona el directorio raíz `C:\LABS\couchbase-nosql` para mantener la estructura uniforme del curso.

- {% include step_label.html %} Abre una terminal integrada desde **Terminal → New Terminal** y confirma que el perfil seleccionado sea **Git Bash**.

- {% include step_label.html %} Crea el subdirectorio de la práctica sin modificar los archivos generados en laboratorios anteriores:

  ```bash
  mkdir -p /c/LABS/couchbase-nosql/lab11
  ```

- {% include step_label.html %} Cambia al subdirectorio para que todos los archivos JSON y evidencias queden almacenados en la ubicación correcta:

  ```bash
  cd /c/LABS/couchbase-nosql/lab11
  ```

- {% include step_label.html %} Confirma la ruta actual antes de continuar y verifica que la salida corresponda exactamente al directorio de la práctica 11:

  ```bash
  pwd
  ```

**Salida esperada:**

```text
/c/LABS/couchbase-nosql/lab11
```

---

## 🔍 Tarea 1. Validar Couchbase Enterprise 7.6.2 y el índice Search existente

### Tarea 1.1. Verificar la imagen, el contenedor y los servicios

- {% include step_label.html %} Define variables de entorno para centralizar credenciales, URLs y el nombre del índice, evitando repetir valores en cada comando:

  ```bash
  export CB_HOST="localhost"
  export CB_ADMIN="Administrator"
  export CB_PASS="Password123!"
  export CB_URL="http://${CB_HOST}:8091"
  export CB_QUERY_URL="http://${CB_HOST}:8093/query/service"
  export CB_SEARCH_URL="http://${CB_HOST}:8094"
  export FTS_INDEX="idx_lab10_hotel_search"
  export FTS_BASE_URL="${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${FTS_INDEX}"
  ```

- {% include step_label.html %} Consulta la imagen asociada al contenedor para confirmar que el laboratorio utiliza Couchbase Server Enterprise 7.6.2:

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

- {% include step_label.html %} Revisa el estado del contenedor, la imagen utilizada y el tiempo activo para confirmar que Couchbase está ejecutándose correctamente:

  {%raw%}
  ```bash
  docker ps --filter "name=couchbase-lab" \
    --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
  ```
  {%endraw%}

- {% include step_label.html %} Si el contenedor aparece detenido, inícialo y espera algunos segundos antes de consultar los servicios internos:

  ```bash
  docker start couchbase-lab
  ```

- {% include step_label.html %} Comprueba que la Web Console responda correctamente en el puerto 8091 y que el código HTTP obtenido sea 200:

  ```bash
  curl -s -o /dev/null \
    -w "Web Console: HTTP %{http_code}\n" \
    "${CB_URL}/ui/index.html"
  ```

- {% include step_label.html %} Comprueba que Query Service acepte conexiones autenticadas en el puerto 8093 antes de ejecutar sentencias SQL++:

  ```bash
  curl -sS -o /dev/null \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -w "Query Service: HTTP %{http_code}\n" \
    "${CB_QUERY_URL%/query/service}/admin/ping"
  ```

- {% include step_label.html %} Comprueba que Search Service esté disponible en el puerto 8094 y pueda listar definiciones de índices FTS:

  ```bash
  curl -s -o /dev/null \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -w "Search Service: HTTP %{http_code}\n" \
    "${CB_SEARCH_URL}/api/index"
  ```

### Tarea 1.2. Validar el índice scoped creado en la práctica 10

- {% include step_label.html %} Consulta la definición del índice scoped para confirmar que pertenece al bucket travel-sample y al scope inventory:

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${FTS_BASE_URL}" \
    | python -m json.tool
  ```

- {% include step_label.html %} Revisa la respuesta y confirma que se incluyan el nombre `idx_lab10_hotel_search`, el bucket y el scope esperados.

- {% include step_label.html %} Consulta el número de documentos indexados para verificar que el índice contiene información disponible para las búsquedas:

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${FTS_BASE_URL}/count" \
    | python -m json.tool
  ```

- {% include step_label.html %} Ejecuta una consulta mínima para comprobar que Search devuelve hits y que los fields almacenados pueden incluirse en la respuesta:

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${FTS_BASE_URL}/query" \
    -d '{
      "query": {
        "match": "hotel",
        "field": "name"
      },
      "size": 1,
      "fields": ["name", "city", "country", "description"]
    }' \
    | python -m json.tool
  ```

- {% include step_label.html %} Confirma que la respuesta incluya `total_hits`, al menos un elemento en `hits` y un objeto `fields` con información almacenada.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

{% include support-prompt.html task="tarea1" %}

---

## ✍️ Tarea 2. Ejecutar Query String Queries

### Tarea 2.1. Crear y ejecutar búsquedas con operadores

- {% include step_label.html %} Crea el archivo `query-string-pool.json` para representar una búsqueda libre sencilla mediante la sintaxis Query String:

  ```json
  {
    "query": {
      "query": "pool"
    },
    "size": 5,
    "fields": ["name", "city", "country", "description"],
    "highlight": {
      "style": "html",
      "fields": ["description"]
    }
  }
  ```

- {% include step_label.html %} Ejecuta el archivo contra el endpoint scoped y conserva la respuesta formateada para identificar hits, score y fragments:

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${FTS_BASE_URL}/query" \
    -d @query-string-pool.json \
    | python -m json.tool
  ```

- {% include step_label.html %} Crea el archivo `query-string-required.json` para exigir que los términos pool y breakfast estén presentes en la coincidencia:

  ```json
  {
    "query": {
      "query": "+pool +breakfast"
    },
    "size": 5,
    "fields": ["name", "city", "country", "description"]
  }
  ```

- {% include step_label.html %} Ejecuta la consulta y compara `total_hits` con la búsqueda simple para observar el efecto de las condiciones obligatorias:

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${FTS_BASE_URL}/query" \
    -d @query-string-required.json \
    | python -m json.tool
  ```

- {% include step_label.html %} Crea el archivo `query-string-exclude.json` para exigir pool y excluir documentos donde aparezca expensive:

  ```json
  {
    "query": {
      "query": "+pool -expensive"
    },
    "size": 5,
    "fields": ["name", "city", "country", "description"]
  }
  ```

- {% include step_label.html %} Ejecuta la consulta y confirma que la exclusión no incremente el número de coincidencias frente a la búsqueda sin restricción:

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${FTS_BASE_URL}/query" \
    -d @query-string-exclude.json \
    | python -m json.tool
  ```

### Tarea 2.2. Aplicar alternativas, fields y wildcards

- {% include step_label.html %} Crea el archivo `query-string-or.json` para aceptar coincidencias con pool o spa mediante el operador booleano OR:

  ```json
  {
    "query": {
      "query": "pool OR spa"
    },
    "size": 5,
    "fields": ["name", "city", "country", "description"]
  }
  ```

- {% include step_label.html %} Ejecuta la consulta y observa cómo una alternativa amplía la búsqueda al aceptar documentos de cualquiera de las condiciones:

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${FTS_BASE_URL}/query" \
    -d @query-string-or.json \
    | python -m json.tool
  ```

- {% include step_label.html %} Crea el archivo `query-string-field.json` para restringir la búsqueda del término hotel exclusivamente al field name:

  ```json
  {
    "query": {
      "query": "name:hotel"
    },
    "size": 5,
    "fields": ["name", "city", "country"]
  }
  ```

- {% include step_label.html %} Ejecuta la consulta y confirma que los resultados correspondan a coincidencias encontradas dentro del field indicado:

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${FTS_BASE_URL}/query" \
    -d @query-string-field.json \
    | python -m json.tool
  ```

- {% include step_label.html %} Crea el archivo `query-string-wildcard.json` para buscar tokens que comiencen con swim mediante el wildcard de múltiples caracteres:

  ```json
  {
    "query": {
      "query": "swim*"
    },
    "size": 5,
    "fields": ["name", "description"],
    "highlight": {
      "style": "html",
      "fields": ["description"]
    }
  }
  ```

- {% include step_label.html %} Ejecuta la consulta y revisa si fragments muestra términos indexados cuyo inicio coincide con el prefijo configurado:

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${FTS_BASE_URL}/query" \
    -d @query-string-wildcard.json \
    | python -m json.tool
  ```

- {% include step_label.html %} Crea `lab11-search-results.md` y registra total_hits, propósito y observaciones de cada Query String ejecutada:

  ```markdown
  # Resultados de Search — Práctica 11

  ## Query String

  | Consulta | Propósito | Total hits | Observación |
  |---|---|---:|---|
  | pool | Búsqueda simple | | |
  | +pool +breakfast | Términos obligatorios | | |
  | +pool -expensive | Obligatorio y exclusión | | |
  | pool OR spa | Alternativas | | |
  | name:hotel | Campo específico | | |
  | swim* | Wildcard | | |
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

{% include support-prompt.html task="tarea2" %}

---

## 🧩 Tarea 3. Comparar Term Query, Match Query y consultas compuestas

### Tarea 3.1. Comparar búsqueda exacta y búsqueda analizada

- {% include step_label.html %} Crea el archivo `term-country.json` para buscar el valor exacto United Kingdom en el field country configurado con keyword:

  ```json
  {
    "query": {
      "term": "United Kingdom",
      "field": "country"
    },
    "size": 5,
    "fields": ["name", "city", "country"]
  }
  ```

- {% include step_label.html %} Ejecuta la Term Query y registra total_hits para disponer de una referencia de búsqueda exacta sobre el valor completo:

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${FTS_BASE_URL}/query" \
    -d @term-country.json \
    | python -m json.tool
  ```

- {% include step_label.html %} Crea el archivo `match-country.json` para buscar el mismo texto mediante una Match Query que consulta el analizador del field:

  ```json
  {
    "query": {
      "match": "United Kingdom",
      "field": "country"
    },
    "size": 5,
    "fields": ["name", "city", "country"]
  }
  ```

- {% include step_label.html %} Ejecuta la Match Query y compara su total_hits con la Term Query para identificar diferencias de tratamiento del texto:

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${FTS_BASE_URL}/query" \
    -d @match-country.json \
    | python -m json.tool
  ```

### Tarea 3.2. Construir consultas conjunction y disjunction

- {% include step_label.html %} Crea `conjunction-query.json` para exigir que pool y breakfast coincidan dentro del field description:

  ```json
  {
    "query": {
      "conjuncts": [
        {
          "match": "pool",
          "field": "description"
        },
        {
          "match": "breakfast",
          "field": "description"
        }
      ]
    },
    "size": 5,
    "fields": ["name", "city", "country", "description"],
    "highlight": {
      "style": "html",
      "fields": ["description"]
    }
  }
  ```

- {% include step_label.html %} Ejecuta la consulta y verifica que el número de resultados sea coherente con una condición que exige ambas coincidencias:

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${FTS_BASE_URL}/query" \
    -d @conjunction-query.json \
    | python -m json.tool
  ```

- {% include step_label.html %} Crea `disjunction-min1.json` para aceptar al menos una coincidencia entre beach, airport y spa:

  ```json
  {
    "query": {
      "disjuncts": [
        {
          "match": "beach",
          "field": "description"
        },
        {
          "match": "airport",
          "field": "description"
        },
        {
          "match": "spa",
          "field": "description"
        }
      ],
      "min": 1
    },
    "size": 5,
    "fields": ["name", "city", "country", "description"]
  }
  ```

- {% include step_label.html %} Ejecuta la consulta y registra total_hits como referencia para comparar con una condición mínima más restrictiva:

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${FTS_BASE_URL}/query" \
    -d @disjunction-min1.json \
    | python -m json.tool
  ```

- {% include step_label.html %} Duplica el archivo anterior como `disjunction-min2.json` y cambia únicamente el valor de min a 2:

  ```json
  {
    "query": {
      "disjuncts": [
        {
          "match": "beach",
          "field": "description"
        },
        {
          "match": "airport",
          "field": "description"
        },
        {
          "match": "spa",
          "field": "description"
        }
      ],
      "min": 2
    },
    "size": 5,
    "fields": ["name", "city", "country", "description"]
  }
  ```

- {% include step_label.html %} Ejecuta la consulta y confirma que total_hits con min 2 sea menor o igual que el resultado obtenido con min 1:

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${FTS_BASE_URL}/query" \
    -d @disjunction-min2.json \
    | python -m json.tool
  ```

- {% include step_label.html %} Agrega al archivo de resultados una tabla que documente la diferencia entre Term, Match, conjunction y disjunction:

  ```markdown
  ## Term, Match y consultas compuestas

  | Consulta | Total hits | Observación |
  |---|---:|---|
  | Term country = United Kingdom | | |
  | Match country = United Kingdom | | |
  | Conjunction pool + breakfast | | |
  | Disjunction min 1 | | |
  | Disjunction min 2 | | |
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

{% include support-prompt.html task="tarea3" %}

---

## 📄 Tarea 4. Implementar paginación estable mediante REST API

### Tarea 4.1. Crear y ejecutar las páginas de resultados

- {% include step_label.html %} Crea `page-1.json` con from igual a 0, size igual a 5 y un orden estable por score e identificador:

  ```json
  {
    "query": {
      "match": "hotel",
      "field": "name"
    },
    "from": 0,
    "size": 5,
    "sort": [
      "-_score",
      "_id"
    ],
    "fields": ["name", "city", "country"]
  }
  ```

- {% include step_label.html %} Crea `page-2.json` con la misma consulta y orden, modificando únicamente from para recuperar los siguientes cinco resultados:

  ```json
  {
    "query": {
      "match": "hotel",
      "field": "name"
    },
    "from": 5,
    "size": 5,
    "sort": [
      "-_score",
      "_id"
    ],
    "fields": ["name", "city", "country"]
  }
  ```

- {% include step_label.html %} Ejecuta la primera página y guarda la respuesta completa en un archivo para poder compararla posteriormente:

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${FTS_BASE_URL}/query" \
    -d @page-1.json \
    > page-1-response.json
  ```

- {% include step_label.html %} Ejecuta la segunda página y conserva su respuesta en un archivo diferente para evitar sobrescribir la evidencia anterior:

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${FTS_BASE_URL}/query" \
    -d @page-2.json \
    > page-2-response.json
  ```

- {% include step_label.html %} Formatea ambos archivos con Python para confirmar que contienen JSON válido y que cada página incluye resultados:

  ```bash
  python -m json.tool page-1-response.json
  python -m json.tool page-2-response.json
  ```

### Tarea 4.2. Comparar IDs y métricas de las páginas

- {% include step_label.html %} Extrae los identificadores de cada página para visualizar los documentos recuperados en cada bloque de resultados:

  ```bash
  python -c "
  import json

  for filename in ['page-1-response.json', 'page-2-response.json']:
      with open(filename, encoding='utf-8') as file:
          data = json.load(file)

      print(filename)
      for hit in data.get('hits', []):
          print('  ', hit.get('id'))
  "
  ```

- {% include step_label.html %} Compara los conjuntos de IDs y confirma que no existan documentos duplicados entre la primera y la segunda página:

  ```bash
  python -c "
  import json

  with open('page-1-response.json', encoding='utf-8') as file:
      page1 = json.load(file)

  with open('page-2-response.json', encoding='utf-8') as file:
      page2 = json.load(file)

  ids1 = {hit.get('id') for hit in page1.get('hits', [])}
  ids2 = {hit.get('id') for hit in page2.get('hits', [])}
  duplicates = ids1.intersection(ids2)

  print('Página 1:', len(ids1), 'documentos')
  print('Página 2:', len(ids2), 'documentos')
  print('Duplicados:', duplicates if duplicates else 'ninguno')
  "
  ```

- {% include step_label.html %} Consulta total_hits, max_score y took para distinguir la cantidad total de coincidencias del tiempo interno del servicio:

  ```bash
  python -c "
  import json

  with open('page-1-response.json', encoding='utf-8') as file:
      data = json.load(file)

  print('Total hits:', data.get('total_hits'))
  print('Max score:', data.get('max_score'))
  print('Took interno (ns):', data.get('took'))
  print('Took aproximado (ms):', data.get('took', 0) / 1_000_000)
  "
  ```

- {% include step_label.html %} Documenta que took representa tiempo interno de Search y no incluye latencia de red ni procesamiento de la Web Console.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

{% include support-prompt.html task="tarea4" %}

---

## 🔗 Tarea 5. Integrar Couchbase Search con SQL++

### Tarea 5.1. Ejecutar SEARCH() y SEARCH_SCORE()

- {% include step_label.html %} Abre la Web Console, inicia sesión como Administrator y selecciona Query para trabajar con SQL++ sobre la collection hotel.

- {% include step_label.html %} Ejecuta una consulta SEARCH() simple para delegar la búsqueda textual al índice y ordenar los documentos mediante SEARCH_SCORE():

  ```sql
  SELECT
      h.name,
      h.city,
      h.country,
      SEARCH_SCORE() AS relevance_score
  FROM `travel-sample`.inventory.hotel AS h
  WHERE SEARCH(
      h,
      {
          "match": "pool breakfast",
          "field": "description",
          "operator": "or"
      },
      {
          "index": "travel-sample.inventory.idx_lab10_hotel_search"
      }
  )
  ORDER BY relevance_score DESC;
  ```

- {% include step_label.html %} Verifica que la consulta devuelva documentos, que relevance_score exista y que los resultados aparezcan ordenados de mayor a menor.

- {% include step_label.html %} Ejecuta una segunda consulta que combine Search con un filtro estructurado para limitar los resultados al Reino Unido:

  ```sql
  SELECT h.name,
         h.city,
         h.country,
         h.description,
         SEARCH_SCORE(h) AS relevance_score
  FROM `travel-sample`.inventory.hotel AS h
  WHERE h.country = "United Kingdom"
    AND SEARCH(
      h,
      {
        "match": "breakfast",
        "field": "description"
      },
      {
        "index": "travel-sample.inventory.idx_lab10_hotel_search"
      }
    )
  ORDER BY relevance_score DESC
  LIMIT 10;
  ```

- {% include step_label.html %} Confirma que todos los documentos devueltos tengan country igual a United Kingdom y que el score siga ordenando la salida.

### Tarea 5.2. Combinar Search con consultas compuestas y agregaciones

- {% include step_label.html %} Ejecuta una consulta compuesta dentro de SEARCH() para aceptar hoteles que mencionen pool, spa o gym en description:

  ```sql
  SELECT h.name,
         h.city,
         h.country,
         SEARCH_SCORE(h) AS relevance_score
  FROM `travel-sample`.inventory.hotel AS h
  WHERE SEARCH(
    h,
    {
      "disjuncts": [
        {
          "match": "pool",
          "field": "description"
        },
        {
          "match": "spa",
          "field": "description"
        },
        {
          "match": "gym",
          "field": "description"
        }
      ],
      "min": 1
    },
    {
      "index": "travel-sample.inventory.idx_lab10_hotel_search"
    }
  )
  ORDER BY relevance_score DESC
  LIMIT 10;
  ```

- {% include step_label.html %} Ejecuta una agregación SQL++ sobre los documentos seleccionados por Search para contar coincidencias agrupadas por país:

  ```sql
  SELECT h.country,
         COUNT(*) AS matching_hotels
  FROM `travel-sample`.inventory.hotel AS h
  WHERE SEARCH(
    h,
    {
      "disjuncts": [
        {
          "match": "pool",
          "field": "description"
        },
        {
          "match": "spa",
          "field": "description"
        },
        {
          "match": "gym",
          "field": "description"
        }
      ],
      "min": 1
    },
    {
      "index": "travel-sample.inventory.idx_lab10_hotel_search"
    }
  )
  GROUP BY h.country
  ORDER BY matching_hotels DESC
  LIMIT 10;
  ```

- {% include step_label.html %} Verifica que la consulta devuelva países y conteos, demostrando que Search selecciona documentos y SQL++ realiza la agregación.

### Tarea 5.3. Validar SEARCH() mediante Query REST API

- {% include step_label.html %} Crea el archivo `search-sqlpp.json` para enviar una sentencia SQL++ completa mediante Query REST API:

  ```json
  {
    "statement": "SELECT h.name, h.city, h.country, SEARCH_SCORE(h) AS relevance_score FROM `travel-sample`.inventory.hotel AS h WHERE SEARCH(h, {\"match\":\"pool\",\"field\":\"description\"}, {\"index\":\"travel-sample.inventory.idx_lab10_hotel_search\"}) ORDER BY relevance_score DESC LIMIT 5;"
  }
  ```

- {% include step_label.html %} Ejecuta el archivo contra Query Service y formatea la respuesta para confirmar que la integración funciona fuera de Web Console:

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${CB_QUERY_URL}" \
    -d @search-sqlpp.json \
    | python -m json.tool
  ```

- {% include step_label.html %} Confirma que la respuesta tenga status success, un arreglo results y valores relevance_score para los documentos recuperados.

- {% include step_label.html %} Agrega una conclusión al archivo de resultados explicando la responsabilidad de Search y la responsabilidad de SQL++:

  ```markdown
  ## Integración Search con SQL++

  SEARCH() utiliza el índice Search para resolver coincidencias de texto libre.

  SEARCH_SCORE() expone el score de relevancia asociado a cada documento.

  SQL++ proyecta campos, aplica filtros estructurados, ordena, limita y agrega
  los documentos seleccionados por Search.
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

{% include support-prompt.html task="tarea5" %}

---

## 🛠️ Resolución de problemas

### Problema 1. La imagen no corresponde a Enterprise 7.6.2

{%raw%}
```bash
docker inspect couchbase-lab \
  --format '{{.Config.Image}}'
```
{%endraw%}

La salida debe mostrar:

```text
couchbase/server:enterprise-7.6.2
```

### Problema 2. El índice no existe

```bash
curl -s \
  -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_SEARCH_URL}/api/index" \
  | python -m json.tool
```

Confirma que exista:

```text
idx_lab10_hotel_search
```

### Problema 3. La Term Query devuelve cero resultados

Verifica en el índice que `country` tenga:

```text
Analyzer: keyword
Index: enabled
Store: enabled
```

### Problema 4. fields no aparece

Edita el índice y confirma que Store esté habilitado para:

```text
name
description
city
country
```

### Problema 5. Las páginas repiten documentos

Confirma que ambas consultas utilicen:

```json
"sort": [
  "-_score",
  "_id"
]
```

y que únicamente cambie el valor de `from`.

### Problema 6. SEARCH() no encuentra el índice

Confirma que el tercer argumento utilice:

```json
{
  "index": "idx_lab10_hotel_search"
}
```

y que la consulta apunte a:

```text
travel-sample.inventory.hotel
```

### Problema 7. SEARCH_SCORE() genera error

Verifica que:

- La consulta contenga `SEARCH(h, ...)`.
- El alias en `SEARCH_SCORE(h)` sea el mismo alias `h`.
- El índice sea compatible con la collection consultada.

### Problema 8. python -m json.tool no funciona

```bash
python --version
python3 --version
```

Si `python3` funciona, reemplaza:

```bash
python -m json.tool
```

por:

```bash
python3 -m json.tool
```