---
layout: lab
title: "Práctica 12: Creación y configuración avanzada de índices Full Text Search"
permalink: /lab12/lab12/
images_base: /labs/lab12/img
duration: "80 minutos"
objective:
  - Preparar el entorno de la práctica 12 y verificar que Couchbase Server Enterprise 7.6.2 esté operativo.
  - Explorar la estructura de documentos hotel y landmark antes de diseñar mappings avanzados.
  - Crear un índice Search con mapping estático, campos simples, campos anidados y Store Field Data.
  - Validar búsquedas sobre reviews.content y reviews.ratings.Overall.
  - Crear un índice Search para landmarks y federar ambos índices mediante un alias.
  - Crear un índice con mapping dinámico mínimo y comparar su configuración con el mapping estático.
prerequisites:
  - Haber completado las prácticas 10 y 11.
  - Tener Couchbase Server Enterprise 7.6.2 en ejecución mediante la imagen couchbase/server:enterprise-7.6.2.
  - Tener habilitados los servicios Data, Query, Index y Search.
  - Tener cargado el bucket travel-sample.
  - Tener disponibles las collections travel-sample.inventory.hotel y travel-sample.inventory.landmark.
  - Tener acceso a la Web Console en http://localhost:8091.
  - Utilizar Visual Studio Code con Git Bash como terminal integrada.
  - Comprender mappings Search, analyzers, campos almacenados y consultas REST básicas.
introduction:
  - En esta práctica diseñarás índices Search avanzados sobre Couchbase Server Enterprise 7.6.2. Crearás un mapping estático para hoteles, configurarás campos de texto, keyword y number, indexarás estructuras anidadas dentro del array reviews, validarás Store Field Data y construirás un segundo índice para landmarks. Finalmente crearás un alias que combine ambos índices y compararás la configuración de un mapping estático frente a uno dinámico.
slug: lab12
lab_number: 12
final_result: >
  Al finalizar la práctica habrás creado idx_lab12_hotel_advanced, idx_lab12_landmark_search y alias_lab12_travel_search, validado campos almacenados, búsquedas en reviews y rangos numéricos, y comparado de forma controlada un mapping estático frente a uno dinámico.
notes:
  - Todos los comandos de terminal deben ejecutarse desde Git Bash dentro de Visual Studio Code.
  - La imagen utilizada debe ser couchbase/server:enterprise-7.6.2.
  - Utiliza las credenciales Administrator y Password123! configuradas en prácticas anteriores.
  - Todos los endpoints REST utilizan rutas scoped compatibles con Couchbase Server 7.6.2.
  - Los índices y el alias deben conservarse al finalizar.
  - Los conteos exactos pueden variar según la versión del dataset travel-sample.
references: []
prev: /lab11/lab11/
next: /lab13/lab13/
---

---
<!-- Aquí comienzan las instrucciones paso a paso de la práctica -->

## 📁 Preparación del directorio de trabajo

### Preparar el subdirectorio de la práctica

- {% include step_label.html %} Abre Docker Desktop y espera hasta que el motor de contenedores termine de iniciar, porque Couchbase Server y todos sus servicios dependen de que Docker esté completamente disponible.

- {% include step_label.html %} Abre Visual Studio Code y selecciona el directorio raíz del curso para conservar una estructura uniforme entre todas las prácticas.

  ```text
  C:\LABS\couchbase-nosql
  ```

- {% include step_label.html %} Abre una terminal integrada desde **Terminal → New Terminal** y confirma que el perfil activo sea Git Bash, ya que los comandos están escritos con sintaxis Bash.

- {% include step_label.html %} Crea el subdirectorio `lab12` sin eliminar ni sobrescribir los directorios de prácticas anteriores.

  ```bash
  mkdir -p /c/LABS/couchbase-nosql/lab12
  ```

- {% include step_label.html %} Cambia al subdirectorio recién creado para que todos los archivos JSON y evidencias permanezcan dentro de la práctica 12.

  ```bash
  cd /c/LABS/couchbase-nosql/lab12
  ```

- {% include step_label.html %} Confirma la ruta actual antes de continuar.

  ```bash
  pwd
  ```

**Salida esperada:**

```text
/c/LABS/couchbase-nosql/lab12
```

---

## 🧭 Tarea 1. Verificar Enterprise 7.6.2 y explorar documentos

En esta tarea comprobarás que el contenedor use la imagen correcta, validarás los servicios requeridos y revisarás la estructura real de hotel y landmark antes de crear mappings.

### Tarea 1.1. Validar contenedor, servicios y variables

- {% include step_label.html %} Define las variables de entorno que utilizarás durante toda la práctica para evitar repetir URLs, credenciales y nombres de índices.

  ```bash
  export CB_HOST="localhost"
  export CB_ADMIN="Administrator"
  export CB_PASS="Password123!"
  export CB_URL="http://${CB_HOST}:8091"
  export CB_QUERY_URL="http://${CB_HOST}:8093/query/service"
  export CB_SEARCH_URL="http://${CB_HOST}:8094"

  export HOTEL_INDEX="idx_lab12_hotel_advanced"
  export LANDMARK_INDEX="idx_lab12_landmark_search"
  export TRAVEL_ALIAS="alias_lab12_travel_search"
  export DYNAMIC_INDEX="idx_lab12_hotel_dynamic"
  ```

- {% include step_label.html %} Verifica que el contenedor utilice exactamente la imagen Enterprise 7.6.2 definida para el curso.

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

- {% include step_label.html %} Confirma que el contenedor esté activo y revisa la imagen y el estado reportados por Docker.

  {%raw%}
  ```bash
  docker ps --filter "name=couchbase-lab" \
    --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
  ```
  {%endraw%}

- {% include step_label.html %} Si el contenedor aparece detenido, inícialo antes de ejecutar cualquier llamada REST.

  ```bash
  docker start couchbase-lab
  ```

- {% include step_label.html %} Verifica que Web Console, Query Service y Search Service respondan correctamente.

  ```bash
  curl -s -o /dev/null \
    -w "Web Console: HTTP %{http_code}\n" \
    "${CB_URL}/ui/index.html"
  ```

  ```bash
  curl -sS -o /dev/null \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -w "Query Service: HTTP %{http_code}\n" \
    "${CB_QUERY_URL%/query/service}/admin/ping"
  ```

  ```bash
  curl -s -o /dev/null \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -w "Search Service: HTTP %{http_code}\n" \
    "${CB_SEARCH_URL}/api/index"
  ```

### Tarea 1.2. Explorar la estructura de hotel y landmark

- {% include step_label.html %} Abre la Web Console en `http://localhost:8091`, inicia sesión como Administrator y selecciona **Query** para inspeccionar la estructura real de los documentos.

- {% include step_label.html %} Ejecuta la siguiente consulta para revisar un documento hotel y confirmar la existencia de campos simples y anidados.

  ```sql
  SELECT META(h).id AS document_id,
         h.name,
         h.description,
         h.city,
         h.country,
         h.reviews[0] AS first_review
  FROM `travel-sample`.inventory.hotel AS h
  LIMIT 1;
  ```

- {% include step_label.html %} Ejecuta una segunda consulta para observar la estructura interna del array reviews y verificar la ubicación de content, author y ratings.Overall.

  ```sql
  SELECT META(h).id AS document_id,
         r.content AS review_content,
         r.author AS review_author,
         r.ratings.Overall AS overall_rating
  FROM `travel-sample`.inventory.hotel AS h
  UNNEST h.reviews AS r
  WHERE r.ratings.Overall IS NOT MISSING
  LIMIT 5;
  ```

- {% include step_label.html %} Ejecuta una tercera consulta para revisar un documento landmark y confirmar los campos que compartirán nombre y estructura con el índice de hotel.

  ```sql
  SELECT META(l).id AS document_id,
         l.name,
         l.content,
         l.city,
         l.country
  FROM `travel-sample`.inventory.landmark AS l
  LIMIT 1;
  ```

- {% include step_label.html %} Obtén los conteos reales de ambas collections para utilizarlos como referencia durante la validación de índices.

  ```sql
  SELECT
    (SELECT RAW COUNT(*) FROM `travel-sample`.inventory.hotel)[0]
      AS hotel_count,
    (SELECT RAW COUNT(*) FROM `travel-sample`.inventory.landmark)[0]
      AS landmark_count;
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

{% include support-prompt.html task="tarea1" %}

---

## 🏗️ Tarea 2. Crear idx_lab12_hotel_advanced con mapping estático

En esta tarea crearás un índice avanzado limitado a travel-sample.inventory.hotel y declararás únicamente los campos necesarios.

### Tarea 2.1. Crear la definición JSON del índice

- {% include step_label.html %} Crea el archivo `idx_lab12_hotel_advanced.json` dentro de `lab12` para conservar la definición como evidencia reutilizable.

  ```bash
  cat > idx_lab12_hotel_advanced.json <<'EOF'
  {
    "name": "idx_lab12_hotel_advanced",
    "type": "fulltext-index",
    "sourceType": "gocbcore",
    "sourceName": "travel-sample",
    "sourceParams": {},
    "planParams": {
      "maxPartitionsPerPIndex": 1024,
      "indexPartitions": 1,
      "numReplicas": 0
    },
    "params": {
      "doc_config": {
        "docid_prefix_delim": "",
        "docid_regexp": "",
        "mode": "scope.collection.type_field",
        "type_field": "type"
      },
      "mapping": {
        "analysis": {},
        "default_analyzer": "standard",
        "default_datetime_parser": "dateTimeOptional",
        "default_field": "_all",
        "default_mapping": {
          "enabled": false,
          "dynamic": false
        },
        "default_type": "_default",
        "docvalues_dynamic": false,
        "index_dynamic": true,
        "store_dynamic": false,
        "type_field": "_type",
        "types": {
          "inventory.hotel": {
            "enabled": true,
            "dynamic": false,
            "properties": {
              "name": {
                "enabled": true,
                "dynamic": false,
                "fields": [
                  {
                    "name": "name",
                    "type": "text",
                    "analyzer": "en",
                    "index": true,
                    "store": true,
                    "docvalues": true,
                    "include_term_vectors": true,
                    "include_in_all": true
                  }
                ]
              },
              "description": {
                "enabled": true,
                "dynamic": false,
                "fields": [
                  {
                    "name": "description",
                    "type": "text",
                    "analyzer": "en",
                    "index": true,
                    "store": false,
                    "docvalues": false,
                    "include_term_vectors": true,
                    "include_in_all": true
                  }
                ]
              },
              "city": {
                "enabled": true,
                "dynamic": false,
                "fields": [
                  {
                    "name": "city",
                    "type": "text",
                    "analyzer": "keyword",
                    "index": true,
                    "store": true,
                    "docvalues": true,
                    "include_term_vectors": false,
                    "include_in_all": false
                  }
                ]
              },
              "country": {
                "enabled": true,
                "dynamic": false,
                "fields": [
                  {
                    "name": "country",
                    "type": "text",
                    "analyzer": "keyword",
                    "index": true,
                    "store": true,
                    "docvalues": true,
                    "include_term_vectors": false,
                    "include_in_all": false
                  }
                ]
              },
              "reviews": {
                "enabled": true,
                "dynamic": false,
                "properties": {
                  "content": {
                    "enabled": true,
                    "dynamic": false,
                    "fields": [
                      {
                        "name": "content",
                        "type": "text",
                        "analyzer": "en",
                        "index": true,
                        "store": false,
                        "docvalues": false,
                        "include_term_vectors": true,
                        "include_in_all": true
                      }
                    ]
                  },
                  "author": {
                    "enabled": true,
                    "dynamic": false,
                    "fields": [
                      {
                        "name": "author",
                        "type": "text",
                        "analyzer": "keyword",
                        "index": true,
                        "store": true,
                        "docvalues": true,
                        "include_term_vectors": false,
                        "include_in_all": false
                      }
                    ]
                  },
                  "ratings": {
                    "enabled": true,
                    "dynamic": false,
                    "properties": {
                      "Overall": {
                        "enabled": true,
                        "dynamic": false,
                        "fields": [
                          {
                            "name": "Overall",
                            "type": "number",
                            "index": true,
                            "store": true,
                            "docvalues": true,
                            "include_in_all": false
                          }
                        ]
                      }
                    }
                  }
                }
              }
            }
          }
        }
      },
      "store": {
        "indexType": "scorch"
      }
    }
  }
  EOF
  ```

- {% include step_label.html %} Verifica que el archivo sea JSON válido antes de enviarlo a Couchbase.

  ```bash
  python -m json.tool idx_lab12_hotel_advanced.json > /dev/null \
    && echo "JSON válido"
  ```

### Tarea 2.2. Crear y verificar el índice

- {% include step_label.html %} Crea el índice utilizando la ruta REST scoped correspondiente al bucket travel-sample y al scope inventory.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X PUT \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${HOTEL_INDEX}" \
    -d @idx_lab12_hotel_advanced.json \
    | python -m json.tool
  ```

- {% include step_label.html %} Consulta la definición creada para confirmar el nombre, la collection y la configuración de mapping estático.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${HOTEL_INDEX}" \
    | python -m json.tool
  ```

- {% include step_label.html %} Consulta el conteo de documentos indexados y repite únicamente si el proceso todavía no termina.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${HOTEL_INDEX}/count" \
    | python -m json.tool
  ```

- {% include step_label.html %} Abre **Search** en la Web Console y verifica visualmente que `idx_lab12_hotel_advanced` aparezca dentro de travel-sample.inventory.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

{% include support-prompt.html task="tarea2" %}

---

## 🧪 Tarea 3. Validar Store Field Data, reviews y rango numérico

En esta tarea comprobarás que los campos almacenados aparezcan en la respuesta y ejecutarás búsquedas sobre contenido y ratings anidados.

### Tarea 3.1. Validar Store Field Data

- {% include step_label.html %} Crea `stored-fields-query.json` con una consulta sobre name y solicita únicamente campos configurados con Store habilitado.

  ```bash
  cat > stored-fields-query.json << 'EOF'
  {
    "query": {
      "match": "hostel",
      "field": "name"
    },
    "size": 5,
    "fields": [
      "name",
      "city",
      "country"
    ],
    "highlight": {
      "style": "html",
      "fields": [
        "name"
      ]
    }
  }
  EOF
  ```

- {% include step_label.html %} Ejecuta la consulta y verifica que name, city y country aparezcan dentro de fields.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${HOTEL_INDEX}/query" \
    -d @stored-fields-query.json \
    | python -m json.tool
  ```

- {% include step_label.html %} Crea una segunda consulta que solicite description, aunque ese campo fue configurado con Store deshabilitado.

  ```bash
  cat > non-stored-field-query.json << 'EOF'
  {
    "query": {
      "match": "breakfast",
      "field": "description"
    },
    "size": 3,
    "fields": [
      "name",
      "description",
      "city"
    ]
  }
  EOF
  ```

- {% include step_label.html %} Ejecuta la consulta y confirma que description puede utilizarse para buscar, aunque no necesariamente aparezca dentro de fields.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${HOTEL_INDEX}/query" \
    -d @non-stored-field-query.json \
    | python -m json.tool
  ```

### Tarea 3.2. Validar campos anidados y rango numérico

- {% include step_label.html %} Crea `reviews-content-query.json` para buscar texto dentro de reviews.content y solicitar el autor almacenado.

  ```bash
  cat > reviews-content-query.json << 'EOF'
  {
    "query": {
      "match": "excellent service",
      "field": "reviews.content",
      "operator": "or"
    },
    "size": 5,
    "fields": [
      "name",
      "city",
      "reviews.author"
    ],
    "highlight": {
      "style": "html",
      "fields": [
        "reviews.content"
      ]
    }
  }
  EOF
  ```

- {% include step_label.html %} Ejecuta la búsqueda y verifica que total_hits sea mayor que cero cuando existan reseñas coincidentes.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${HOTEL_INDEX}/query" \
    -d @reviews-content-query.json \
    | python -m json.tool
  ```

- {% include step_label.html %} Crea `reviews-rating-query.json` para consultar reviews.ratings.Overall entre 4 y 5.

  ```bash
  cat > reviews-rating-query.json << 'EOF'
  {
    "query": {
      "min": 4,
      "max": 5,
      "inclusive_min": true,
      "inclusive_max": true,
      "field": "reviews.ratings.Overall"
    },
    "size": 5,
    "fields": [
      "name",
      "city",
      "country",
      "reviews.ratings.Overall"
    ]
  }
  EOF
  ```

- {% include step_label.html %} Ejecuta la consulta numérica y revisa que los valores devueltos se encuentren dentro del rango esperado.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${HOTEL_INDEX}/query" \
    -d @reviews-rating-query.json \
    | python -m json.tool
  ```

- {% include step_label.html %} Crea una consulta compuesta para demostrar que un documento puede coincidir con texto en reviews.content y con un rating dentro del rango.

  ```bash
  cat > reviews-combined-query.json << 'EOF'
  {
    "query": {
      "conjuncts": [
        {
          "match": "clean comfortable",
          "field": "reviews.content",
          "operator": "or"
        },
        {
          "min": 4,
          "max": 5,
          "inclusive_min": true,
          "inclusive_max": true,
          "field": "reviews.ratings.Overall"
        }
      ]
    },
    "size": 5,
    "fields": [
      "name",
      "city",
      "country"
    ]
  }
  EOF
  ```

- {% include step_label.html %} Ejecuta la consulta y recuerda que ambas condiciones se evalúan sobre campos multivaluados del documento, pero no necesariamente sobre el mismo elemento del array reviews.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${HOTEL_INDEX}/query" \
    -d @reviews-combined-query.json \
    | python -m json.tool
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

{% include support-prompt.html task="tarea3" %}

---

## 🗺️ Tarea 4. Crear índice landmark y alias federado

En esta tarea crearás un índice compatible para landmark y después un alias que consulte hotel y landmark desde un único nombre lógico.

### Tarea 4.1. Crear idx_lab12_landmark_search

- {% include step_label.html %} Crea el archivo `idx_lab12_landmark_search.json` con fields compatibles con el índice de hotel.

  ```bash
  cat > idx_lab12_landmark_search.json <<'EOF'
  {
    "name": "idx_lab12_landmark_search",
    "type": "fulltext-index",
    "sourceType": "gocbcore",
    "sourceName": "travel-sample",
    "sourceParams": {},
    "planParams": {
      "maxPartitionsPerPIndex": 1024,
      "indexPartitions": 1,
      "numReplicas": 0
    },
    "params": {
      "doc_config": {
        "docid_prefix_delim": "",
        "docid_regexp": "",
        "mode": "scope.collection.type_field",
        "type_field": "type"
      },
      "mapping": {
        "analysis": {},
        "default_analyzer": "standard",
        "default_datetime_parser": "dateTimeOptional",
        "default_field": "_all",
        "default_mapping": {
          "enabled": false,
          "dynamic": false
        },
        "default_type": "_default",
        "docvalues_dynamic": false,
        "index_dynamic": true,
        "store_dynamic": false,
        "type_field": "_type",
        "types": {
          "inventory.landmark": {
            "enabled": true,
            "dynamic": false,
            "properties": {
              "name": {
                "enabled": true,
                "dynamic": false,
                "fields": [
                  {
                    "name": "name",
                    "type": "text",
                    "analyzer": "en",
                    "index": true,
                    "store": true,
                    "docvalues": true,
                    "include_term_vectors": true,
                    "include_in_all": true
                  }
                ]
              },
              "content": {
                "enabled": true,
                "dynamic": false,
                "fields": [
                  {
                    "name": "content",
                    "type": "text",
                    "analyzer": "en",
                    "index": true,
                    "store": false,
                    "docvalues": false,
                    "include_term_vectors": true,
                    "include_in_all": true
                  }
                ]
              },
              "city": {
                "enabled": true,
                "dynamic": false,
                "fields": [
                  {
                    "name": "city",
                    "type": "text",
                    "analyzer": "keyword",
                    "index": true,
                    "store": true,
                    "docvalues": true,
                    "include_term_vectors": false,
                    "include_in_all": false
                  }
                ]
              },
              "country": {
                "enabled": true,
                "dynamic": false,
                "fields": [
                  {
                    "name": "country",
                    "type": "text",
                    "analyzer": "keyword",
                    "index": true,
                    "store": true,
                    "docvalues": true,
                    "include_term_vectors": false,
                    "include_in_all": false
                  }
                ]
              }
            }
          }
        }
      },
      "store": {
        "indexType": "scorch"
      }
    }
  }
  EOF
  ```

- {% include step_label.html %} Valida el JSON antes de crear el índice.

  ```bash
  python -m json.tool idx_lab12_landmark_search.json > /dev/null \
    && echo "JSON válido"
  ```

- {% include step_label.html %} Crea el índice scoped para la collection landmark.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X PUT \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${LANDMARK_INDEX}" \
    -d @idx_lab12_landmark_search.json \
    | python -m json.tool
  ```

- {% include step_label.html %} Verifica que el índice contenga documentos.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${LANDMARK_INDEX}/count" \
    | python -m json.tool
  ```

### Tarea 4.2. Crear y consultar alias_lab12_travel_search

- {% include step_label.html %} Crea la definición del alias con ambos índices como targets.

  ```bash
  cat > alias_lab12_travel_search.json <<'EOF'
  {
    "name": "alias_lab12_travel_search",
    "type": "fulltext-alias",
    "params": {
      "targets": {
        "travel-sample.inventory.idx_lab12_hotel_advanced": {},
        "travel-sample.inventory.idx_lab12_landmark_search": {}
      }
    },
    "sourceType": "nil",
    "sourceName": "",
    "sourceParams": null,
    "planParams": {}
  }
  EOF
  ```

- {% include step_label.html %} Crea el alias mediante la API general de índices Search, ya que un alias no pertenece a una sola collection.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X PUT \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/index/${TRAVEL_ALIAS}" \
    -d @alias_lab12_travel_search.json \
    | python -m json.tool
  ```

- {% include step_label.html %} Verifica que el alias apunte a los dos índices esperados.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${CB_SEARCH_URL}/api/index/${TRAVEL_ALIAS}" \
    | python -m json.tool
  ```

- {% include step_label.html %} Crea una consulta federada sobre name, un field presente y almacenado en ambos índices.

  ```bash
  cat > alias-search-query.json << 'EOF'
  {
    "query": {
      "match": "museum",
      "field": "name"
    },
    "size": 10,
    "fields": [
      "name",
      "city",
      "country"
    ]
  }
  EOF
  ```

- {% include step_label.html %} Ejecuta la búsqueda sobre el alias y revisa los IDs para identificar si los resultados provienen de hotel, landmark o de ambas collections.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/index/${TRAVEL_ALIAS}/query" \
    -d @alias-search-query.json \
    | python -m json.tool
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

{% include support-prompt.html task="tarea4" %}

---

## ⚖️ Tarea 5. Crear mapping dinámico mínimo y comparar configuraciones

En esta tarea crearás un índice dinámico únicamente para comparar su definición con el mapping estático, sin realizar afirmaciones de rendimiento no medidas.

### Tarea 5.1. Crear idx_lab12_hotel_dynamic

- {% include step_label.html %} Crea `idx_lab12_hotel_dynamic.json` con dynamic habilitado para que Couchbase detecte e indexe campos automáticamente.

  ```bash
  cat > idx_lab12_hotel_dynamic.json <<'EOF'
  {
    "name": "idx_lab12_hotel_dynamic",
    "type": "fulltext-index",
    "sourceType": "gocbcore",
    "sourceName": "travel-sample",
    "sourceParams": {},
    "planParams": {
      "maxPartitionsPerPIndex": 1024,
      "indexPartitions": 1,
      "numReplicas": 0
    },
    "params": {
      "doc_config": {
        "docid_prefix_delim": "",
        "docid_regexp": "",
        "mode": "scope.collection.type_field",
        "type_field": "type"
      },
      "mapping": {
        "analysis": {},
        "default_analyzer": "standard",
        "default_datetime_parser": "dateTimeOptional",
        "default_field": "_all",
        "default_mapping": {
          "enabled": false,
          "dynamic": false
        },
        "default_type": "_default",
        "docvalues_dynamic": true,
        "index_dynamic": true,
        "store_dynamic": false,
        "type_field": "_type",
        "types": {
          "inventory.hotel": {
            "enabled": true,
            "dynamic": true
          }
        }
      },
      "store": {
        "indexType": "scorch"
      }
    }
  }
  EOF
  ```

- {% include step_label.html %} Valida el JSON y crea el índice scoped.

  ```bash
  python -m json.tool idx_lab12_hotel_dynamic.json > /dev/null \
    && echo "JSON válido"
  ```

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X PUT \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${DYNAMIC_INDEX}" \
    -d @idx_lab12_hotel_dynamic.json \
    | python -m json.tool
  ```

### Tarea 5.2. Comparar mapping estático y dinámico

- {% include step_label.html %} Descarga la definición del índice estático y guárdala como evidencia local.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${HOTEL_INDEX}" \
    > static-index-definition.json
  ```

- {% include step_label.html %} Descarga la definición del índice dinámico para realizar una comparación equivalente.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${DYNAMIC_INDEX}" \
    > dynamic-index-definition.json
  ```

- {% include step_label.html %} Ejecuta un script corto para mostrar el valor de dynamic y la cantidad de properties declaradas en cada definición.

  ```bash
  python -c "
  import json

  files = {
      'Estático': 'static-index-definition.json',
      'Dinámico': 'dynamic-index-definition.json'
  }

  type_mapping_name = 'inventory.hotel'

  for label, filename in files.items():
      with open(filename, encoding='utf-8') as file:
          data = json.load(file)

      mapping = (
          data.get('indexDef', {})
              .get('params', {})
              .get('mapping', {})
      )

      default_mapping = mapping.get('default_mapping', {})
      type_mapping = (
          mapping.get('types', {})
                .get(type_mapping_name, {})
      )

      properties = type_mapping.get('properties', {})

      print(label)
      print('  default_mapping enabled:',
            default_mapping.get('enabled'))
      print('  default_mapping dynamic:',
            default_mapping.get('dynamic'))
      print('  type mapping:',
            type_mapping_name)
      print('  type mapping enabled:',
            type_mapping.get('enabled'))
      print('  type mapping dynamic:',
            type_mapping.get('dynamic'))
      print('  properties declaradas:',
            len(properties))
      print('  fields raíz:',
            sorted(properties.keys()))
      print()
  "
  ```

- {% include step_label.html %} Crea `mapping-comparison.md` y documenta las diferencias observadas sin asignar porcentajes de ahorro no medidos.

  ```markdown
  # Comparación de mappings Search

  | Criterio | Mapping estático | Mapping dinámico |
  |---|---|---|
  | Configuración inicial | Mayor | Menor |
  | Campos indexados | Solo declarados | Detectados automáticamente |
  | Control de tipos | Alto | Menor |
  | Riesgo de indexar datos innecesarios | Menor | Mayor |
  | Mantenimiento | Explícito | Dependiente del esquema |
  | Uso recomendado | Casos planificados | Prototipos y exploración |
  ```

- {% include step_label.html %} Verifica que los cuatro recursos creados permanezcan disponibles al terminar la práctica.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${CB_SEARCH_URL}/api/index" \
    | python -m json.tool \
    | grep -E "${HOTEL_INDEX}|${LANDMARK_INDEX}|${TRAVEL_ALIAS}|${DYNAMIC_INDEX}"
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

{% include support-prompt.html task="tarea5" %}

---

## 🛠️ Resolución de problemas

### Problema 1. La imagen no corresponde a Enterprise 7.6.2

- {% include step_label.html %} Ejecuta nuevamente la inspección del contenedor para confirmar la imagen real.

  {%raw%}
  ```bash
  docker inspect couchbase-lab \
    --format '{{.Config.Image}}'
  ```
  {%endraw%}

- {% include step_label.html %} Si la salida no coincide con `couchbase/server:enterprise-7.6.2`, revisa cómo fue creado el contenedor antes de continuar.

### Problema 2. El índice reporta count igual a 0

- {% include step_label.html %} Confirma que la source collection sea hotel o landmark según el índice.

- {% include step_label.html %} Revisa la definición completa para comprobar sourceParams, scope_name y collection_names.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${HOTEL_INDEX}" \
    | python -m json.tool
  ```

### Problema 3. fields no aparece en los hits

- {% include step_label.html %} Confirma que el field solicitado tenga Store habilitado dentro de la definición.

- {% include step_label.html %} Recuerda que Index permite buscar, mientras Store permite devolver el valor dentro de fields.

### Problema 4. reviews.content no devuelve resultados

- {% include step_label.html %} Revisa que reviews esté configurado como objeto padre y content como child field.

- {% include step_label.html %} Prueba un término más general para confirmar que el mapping funciona.

  ```json
  {
    "query": {
      "match": "great",
      "field": "reviews.content"
    },
    "size": 5
  }
  ```

### Problema 5. El rango numérico produce error

- {% include step_label.html %} Confirma que Overall esté declarado como type number.

- {% include step_label.html %} Verifica que la ruta del field sea exactamente:

  ```text
  reviews.ratings.Overall
  ```

### Problema 6. El alias devuelve index not found

- {% include step_label.html %} Verifica que ambos índices target existan y tengan documentos.

- {% include step_label.html %} Consulta la definición del alias y revisa targets.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${CB_SEARCH_URL}/api/index/${TRAVEL_ALIAS}" \
    | python -m json.tool
  ```

### Problema 7. python -m json.tool no funciona

- {% include step_label.html %} Comprueba las versiones disponibles.

  ```bash
  python --version
  python3 --version
  ```

- {% include step_label.html %} Si solo funciona python3, reemplaza `python` por `python3` en los comandos correspondientes.