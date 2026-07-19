---
layout: lab
title: "Práctica 13: Creación y validación de analyzers personalizados para búsqueda"
permalink: /lab13/lab13/
images_base: /labs/lab13/img
duration: "60 minutos"
objective:
  - Preparar un conjunto controlado de documentos en español para validar analyzers de forma reproducible.
  - Crear un índice Search de referencia con analyzers predeterminados y un segundo índice con analyzers personalizados.
  - Comprender el pipeline formado por character filters, tokenizer y token filters.
  - Validar la normalización de HTML, acentos, stop words, variantes morfológicas y códigos SKU.
  - Comparar búsquedas lingüísticas y exactas mediante la REST API scoped de Couchbase Search.
prerequisites:
  - Haber completado la Práctica 12 o contar con experiencia equivalente en mappings Search estáticos.
  - Tener Couchbase Server Enterprise 7.6.2 en ejecución mediante la imagen couchbase/server:enterprise-7.6.2.
  - Tener habilitados los servicios Data, Query, Index y Search.
  - Tener cargado el bucket travel-sample y espacio disponible para crear un scope y una collection de laboratorio.
  - Tener acceso a la Web Console en http://localhost:8091.
  - Utilizar Visual Studio Code con Git Bash como terminal integrada.
  - Comprender consultas SQL++ básicas y la estructura de una definición JSON de índice Search.
introduction:
  - En esta práctica crearás una collection controlada con hoteles escritos en español para demostrar de forma verificable cómo un analyzer modifica el texto antes de indexarlo y consultarlo. Compararás un índice de referencia con un índice que utiliza character filters, tokenizer Unicode, filtros de stop words y stemming en español, además de un analyzer especializado para códigos SKU.
slug: lab13
lab_number: 13
final_result: >
  Al finalizar habrás creado una collection con documentos en español, dos índices Search scoped, un analyzer lingüístico y un analyzer exacto para SKU, y habrás documentado diferencias verificables entre el procesamiento estándar y el personalizado.
notes:
  - Todos los comandos deben ejecutarse desde Git Bash dentro de Visual Studio Code.
  - La imagen obligatoria para esta práctica es couchbase/server:enterprise-7.6.2.
  - Se utilizarán las credenciales Administrator y Password123! configuradas en las prácticas anteriores.
  - Los índices se crearán mediante rutas REST scoped compatibles con Couchbase Server 7.6.2.
  - Los documentos, archivos e índices se conservarán al finalizar para revisión y evidencia.
  - Los tokens exactos producidos por stemming pueden variar; la validación se basa en propiedades observables y no en una raíz lingüística predeterminada.
references: []
prev: /lab12/lab12/
next: /lab14/lab14/
---

---
<!-- Aquí comienzan las instrucciones paso a paso de la práctica -->

## 📁 Preparación del directorio de trabajo

### Preparar el subdirectorio de la práctica

- {% include step_label.html %} Abre Docker Desktop y espera hasta que el motor indique que los contenedores pueden ejecutarse, porque Couchbase Server no responderá mientras Docker continúe iniciándose.

- {% include step_label.html %} Abre Visual Studio Code y selecciona el directorio raíz del curso para mantener la misma organización utilizada en las prácticas anteriores.

  ```text
  C:\LABS\couchbase-nosql
  ```

- {% include step_label.html %} Abre una terminal integrada desde **Terminal → New Terminal** y confirma que el perfil seleccionado sea Git Bash, ya que esta práctica utiliza variables de entorno, redirecciones y bloques heredoc de Bash.

- {% include step_label.html %} Crea el subdirectorio `lab13` sin modificar los archivos de las prácticas anteriores.

  ```bash
  mkdir -p /c/LABS/couchbase-nosql/lab13
  ```

- {% include step_label.html %} Cambia al subdirectorio recién creado para que todos los archivos SQL, JSON y Markdown permanezcan como evidencia de la práctica.

  ```bash
  cd /c/LABS/couchbase-nosql/lab13
  ```

- {% include step_label.html %} Confirma la ruta actual antes de continuar con la preparación de Couchbase.

  ```bash
  pwd
  ```

**Salida esperada:**

```text
/c/LABS/couchbase-nosql/lab13
```

---

## 🧭 Tarea 1. Verificar Enterprise 7.6.2 y preparar documentos en español

En esta tarea validarás la versión del contenedor, crearás un scope y una collection específicos para el laboratorio e insertarás documentos cuyo contenido permita probar acentos, HTML, stop words, stemming y códigos SKU.

### Tarea 1.1. Definir variables y validar el contenedor

- {% include step_label.html %} Define las variables de entorno que centralizan credenciales, endpoints, nombres de recursos y nombres completos de los índices scoped.

  ```bash
  export CB_HOST="localhost"
  export CB_ADMIN="Administrator"
  export CB_PASS="Password123!"

  export CB_URL="http://${CB_HOST}:8091"
  export CB_QUERY_URL="http://${CB_HOST}:8093/query/service"
  export CB_SEARCH_URL="http://${CB_HOST}:8094"

  export CB_BUCKET="travel-sample"
  export CB_SCOPE="search_es"
  export CB_COLLECTION="hotels"

  export STANDARD_INDEX="idx_lab13_spanish_standard"
  export CUSTOM_INDEX="idx_lab13_spanish_custom"

  export STANDARD_FULL_NAME="${CB_BUCKET}.${CB_SCOPE}.${STANDARD_INDEX}"
  export CUSTOM_FULL_NAME="${CB_BUCKET}.${CB_SCOPE}.${CUSTOM_INDEX}"
  ```

- {% include step_label.html %} Comprueba que `couchbase-lab` utilice exactamente la imagen Enterprise 7.6.2 acordada para el curso.

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

- {% include step_label.html %} Si el contenedor no aparece entre los procesos activos, inícialo y espera a que la Web Console vuelva a responder.

  ```bash
  docker start couchbase-lab
  ```

- {% include step_label.html %} Verifica que Query Service y Search Service acepten conexiones antes de crear recursos.

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

### Tarea 1.2. Crear el scope y la collection de laboratorio

- {% include step_label.html %} Crea el scope `search_es` dentro de travel-sample para separar los datos controlados de los documentos originales del sample bucket.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -d "name=${CB_SCOPE}" \
    "${CB_URL}/pools/default/buckets/${CB_BUCKET}/scopes" \
    | python -m json.tool
  ```

> Si Couchbase informa que el scope ya existe, conserva el recurso y continúa; este comportamiento permite repetir la práctica sin recrear toda la estructura.

- {% include step_label.html %} Crea la collection `hotels` dentro de `search_es` para almacenar exclusivamente los documentos en español utilizados por los analyzers.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -d "name=${CB_COLLECTION}" \
    "${CB_URL}/pools/default/buckets/${CB_BUCKET}/scopes/${CB_SCOPE}/collections" \
    | python -m json.tool
  ```

- {% include step_label.html %} Consulta la definición del scope para confirmar que la collection esté registrada antes de insertar datos.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${CB_URL}/pools/default/buckets/${CB_BUCKET}/scopes" \
    | python -m json.tool \
    | grep -E '"name": "search_es"|"name": "hotels"'
  ```

### Tarea 1.3. Insertar documentos con casos de análisis controlados

- {% include step_label.html %} Crea `prepare-spanish-data.sql` con documentos que incluyan acentos, HTML, singular, plural, stop words y códigos SKU con combinaciones de mayúsculas y minúsculas.

  ```bash
  cat > prepare-spanish-data.sql << 'EOF'
  UPSERT INTO `travel-sample`.search_es.hotels (KEY, VALUE)
  VALUES
  (
    "hotel-es-001",
    {
      "type": "spanish_hotel",
      "hotel_id": "ES-001",
      "sku": "HTL-MADRID-001",
      "name": "Hotel Mirador del Mar",
      "description": "<p>Habitaciones cómodas con vista al mar y café incluido en el precio.</p>",
      "city": "Madrid",
      "country": "España"
    }
  ),
  (
    "hotel-es-002",
    {
      "type": "spanish_hotel",
      "hotel_id": "ES-002",
      "sku": "htl-bcn-002",
      "name": "Hostal Café Central",
      "description": "Una habitación tranquila con desayuno, cafetería y acceso al centro histórico.",
      "city": "Barcelona",
      "country": "España"
    }
  ),
  (
    "hotel-es-003",
    {
      "type": "spanish_hotel",
      "hotel_id": "ES-003",
      "sku": "HTL-CUN-003",
      "name": "Hotel Paraíso del Caribe",
      "description": "<div>Habitaciones amplias con piscina y vistas al océano.</div>",
      "city": "Cancún",
      "country": "México"
    }
  ),
  (
    "hotel-es-004",
    {
      "type": "spanish_hotel",
      "hotel_id": "ES-004",
      "sku": "HTL-OAX-004",
      "name": "Posada Jardín Colonial",
      "description": "Habitación familiar con jardín, café orgánico y desayuno regional.",
      "city": "Oaxaca",
      "country": "México"
    }
  ),
  (
    "hotel-es-005",
    {
      "type": "spanish_hotel",
      "hotel_id": "ES-005",
      "sku": "htl-bog-005",
      "name": "Hotel Vista Andina",
      "description": "Cómodas habitaciones para viajeros con vistas a la montaña.",
      "city": "Bogotá",
      "country": "Colombia"
    }
  ),
  (
    "hotel-es-006",
    {
      "type": "spanish_hotel",
      "hotel_id": "ES-006",
      "sku": "HTL-LIM-006",
      "name": "Casa del Café",
      "description": "<section>Una habitación elegante con café, terraza y vista a la ciudad.</section>",
      "city": "Lima",
      "country": "Perú"
    }
  );
  EOF
  ```

- {% include step_label.html %} Envía el archivo al Query Service para insertar o reemplazar los seis documentos de forma reproducible.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    "${CB_QUERY_URL}" \
    --data-urlencode "statement@prepare-spanish-data.sql" \
    | python -m json.tool
  ```

- {% include step_label.html %} Crea un índice primario únicamente para facilitar las verificaciones SQL++ de esta collection de laboratorio.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    "${CB_QUERY_URL}" \
    --data-urlencode 'statement=CREATE PRIMARY INDEX IF NOT EXISTS ON `travel-sample`.search_es.hotels;' \
    | python -m json.tool
  ```

- {% include step_label.html %} Verifica que los documentos hayan quedado disponibles y revisa los valores de name, sku y description.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    "${CB_QUERY_URL}" \
    --data-urlencode 'statement=SELECT META(h).id, h.sku, h.name, h.description FROM `travel-sample`.search_es.hotels AS h ORDER BY META(h).id;' \
    | python -m json.tool
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

{% include support-prompt.html task="tarea1" %}

---

## 🏗️ Tarea 2. Crear índices estándar y personalizados

En esta tarea crearás dos índices sobre la misma collection. El primero funcionará como referencia y el segundo aplicará un pipeline diseñado para contenido en español y códigos exactos.

### Tarea 2.1. Comprender el pipeline de análisis

El pipeline de un analyzer procesa el texto en este orden:

```text
Texto de entrada
      ↓
Character filters
      ↓
Tokenizer
      ↓
Token filters
      ↓
Tokens indexados o consultados
```

| Etapa | Componente utilizado | Función en esta práctica |
|---|---|---|
| Character filter | `html` | Elimina etiquetas HTML sin eliminar su contenido textual |
| Character filter | `asciifolding` | Convierte caracteres como `á`, `é` o `ñ` a equivalentes ASCII cuando existen |
| Tokenizer | `unicode` | Divide lenguaje natural conforme a límites de palabras Unicode |
| Token filter | `to_lower` | Convierte los tokens a minúsculas |
| Token filter | `stop_es` | Elimina stop words frecuentes del español castellano |
| Token filter | `stemmer_es_snowball` | Reduce variantes morfológicas a stems comparables |
| Tokenizer SKU | `single` | Conserva el código completo como un único token |
| Token filter SKU | `to_lower` | Normaliza mayúsculas y minúsculas sin dividir el código |

### Tarea 2.2. Crear el índice de referencia

- {% include step_label.html %} Crea `idx_lab13_spanish_standard.json` con un mapping estático que utilice `standard` para name y description, y `keyword` para sku, city y country.

  ```bash
  cat > idx_lab13_spanish_standard.json << 'EOF'
  {
    "type": "fulltext-index",
    "name": "idx_lab13_spanish_standard",
    "sourceType": "gocbcore",
    "sourceName": "travel-sample",
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
          "dynamic": false,
          "enabled": false
        },
        "default_type": "_default",
        "docvalues_dynamic": false,
        "index_dynamic": false,
        "store_dynamic": false,
        "type_field": "_type",
        "types": {
          "search_es.hotels": {
            "dynamic": false,
            "enabled": true,
            "properties": {
              "name": {
                "dynamic": false,
                "enabled": true,
                "fields": [
                  {
                    "name": "name",
                    "type": "text",
                    "analyzer": "standard",
                    "index": true,
                    "store": true,
                    "include_term_vectors": true,
                    "include_in_all": true
                  }
                ]
              },
              "description": {
                "dynamic": false,
                "enabled": true,
                "fields": [
                  {
                    "name": "description",
                    "type": "text",
                    "analyzer": "standard",
                    "index": true,
                    "store": true,
                    "include_term_vectors": true,
                    "include_in_all": true
                  }
                ]
              },
              "sku": {
                "dynamic": false,
                "enabled": true,
                "fields": [
                  {
                    "name": "sku",
                    "type": "text",
                    "analyzer": "keyword",
                    "index": true,
                    "store": true,
                    "include_in_all": false
                  }
                ]
              },
              "city": {
                "dynamic": false,
                "enabled": true,
                "fields": [
                  {
                    "name": "city",
                    "type": "text",
                    "analyzer": "keyword",
                    "index": true,
                    "store": true
                  }
                ]
              },
              "country": {
                "dynamic": false,
                "enabled": true,
                "fields": [
                  {
                    "name": "country",
                    "type": "text",
                    "analyzer": "keyword",
                    "index": true,
                    "store": true
                  }
                ]
              }
            }
          }
        }
      },
      "store": {
        "indexType": "scorch",
        "segmentVersion": 15
      }
    },
    "sourceParams": {}
  }
  EOF
  ```

- {% include step_label.html %} Valida que la definición sea JSON válido antes de enviarla al Search Service.

  ```bash
  python -m json.tool idx_lab13_spanish_standard.json > /dev/null \
    && echo "Definición estándar válida"
  ```

- {% include step_label.html %} Crea el índice mediante la ruta REST scoped del bucket travel-sample y el scope search_es.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X PUT \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/bucket/${CB_BUCKET}/scope/${CB_SCOPE}/index/${STANDARD_INDEX}" \
    -d @idx_lab13_spanish_standard.json \
    | python -m json.tool
  ```

### Tarea 2.3. Crear el índice con analyzers personalizados

- {% include step_label.html %} Crea `idx_lab13_spanish_custom.json` con `spanish_hotel_analyzer` y `sku_analyzer`, declarados dentro de mapping.analysis.analyzers.

  ```bash
  cat > idx_lab13_spanish_custom.json << 'EOF'
  {
    "type": "fulltext-index",
    "name": "idx_lab13_spanish_custom",
    "sourceType": "gocbcore",
    "sourceName": "travel-sample",
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
        "analysis": {
          "analyzers": {
            "spanish_hotel_analyzer": {
              "type": "custom",
              "char_filters": [
                "html",
                "asciifolding"
              ],
              "tokenizer": "unicode",
              "token_filters": [
                "to_lower",
                "stop_es",
                "stemmer_es_snowball"
              ]
            },
            "sku_analyzer": {
              "type": "custom",
              "char_filters": [],
              "tokenizer": "single",
              "token_filters": [
                "to_lower"
              ]
            }
          }
        },
        "default_analyzer": "standard",
        "default_datetime_parser": "dateTimeOptional",
        "default_field": "_all",
        "default_mapping": {
          "dynamic": false,
          "enabled": false
        },
        "default_type": "_default",
        "docvalues_dynamic": false,
        "index_dynamic": false,
        "store_dynamic": false,
        "type_field": "_type",
        "types": {
          "search_es.hotels": {
            "dynamic": false,
            "enabled": true,
            "properties": {
              "name": {
                "dynamic": false,
                "enabled": true,
                "fields": [
                  {
                    "name": "name",
                    "type": "text",
                    "analyzer": "spanish_hotel_analyzer",
                    "index": true,
                    "store": true,
                    "include_term_vectors": true,
                    "include_in_all": true
                  }
                ]
              },
              "description": {
                "dynamic": false,
                "enabled": true,
                "fields": [
                  {
                    "name": "description",
                    "type": "text",
                    "analyzer": "spanish_hotel_analyzer",
                    "index": true,
                    "store": true,
                    "include_term_vectors": true,
                    "include_in_all": true
                  }
                ]
              },
              "sku": {
                "dynamic": false,
                "enabled": true,
                "fields": [
                  {
                    "name": "sku",
                    "type": "text",
                    "analyzer": "sku_analyzer",
                    "index": true,
                    "store": true,
                    "include_in_all": false
                  }
                ]
              },
              "city": {
                "dynamic": false,
                "enabled": true,
                "fields": [
                  {
                    "name": "city",
                    "type": "text",
                    "analyzer": "keyword",
                    "index": true,
                    "store": true
                  }
                ]
              },
              "country": {
                "dynamic": false,
                "enabled": true,
                "fields": [
                  {
                    "name": "country",
                    "type": "text",
                    "analyzer": "keyword",
                    "index": true,
                    "store": true
                  }
                ]
              }
            }
          }
        }
      },
      "store": {
        "indexType": "scorch",
        "segmentVersion": 15
      }
    },
    "sourceParams": {}
  }
  EOF
  ```

- {% include step_label.html %} Valida la sintaxis JSON para detectar errores de comas, llaves o nombres antes de solicitar la construcción del índice.

  ```bash
  python -m json.tool idx_lab13_spanish_custom.json > /dev/null \
    && echo "Definición personalizada válida"
  ```

- {% include step_label.html %} Crea el índice personalizado mediante la ruta REST scoped.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X PUT \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/bucket/${CB_BUCKET}/scope/${CB_SCOPE}/index/${CUSTOM_INDEX}" \
    -d @idx_lab13_spanish_custom.json \
    | python -m json.tool
  ```

- {% include step_label.html %} Consulta los conteos de ambos índices y repite solo si alguno todavía no refleja los seis documentos insertados.

  ```bash
  for index_name in "${STANDARD_INDEX}" "${CUSTOM_INDEX}"; do
    echo "=== ${index_name} ==="
    curl -s \
      -u "${CB_ADMIN}:${CB_PASS}" \
      "${CB_SEARCH_URL}/api/bucket/${CB_BUCKET}/scope/${CB_SCOPE}/index/${index_name}/count" \
      | python -m json.tool
  done
  ```

- {% include step_label.html %} Abre **Search** en la Web Console y verifica que ambos índices aparezcan dentro de travel-sample.search_es sin errores de mapping.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

{% include support-prompt.html task="tarea2" %}

---

## 🔬 Tarea 3. Validar el pipeline mediante Analyze Document

En esta tarea enviarás el mismo documento a ambos índices para observar cómo cada mapping transforma sus fields. El endpoint utiliza el nombre completo scoped del índice porque la ruta general no incluye bucket y scope por separado.

### Tarea 3.1. Crear el documento de análisis

- {% include step_label.html %} Crea `analyze-spanish-document.json` con HTML, acentos, stop words, variantes morfológicas y un SKU en mayúsculas.

  ```bash
  cat > analyze-spanish-document.json << 'EOF'
  {
    "type": "spanish_hotel",
    "hotel_id": "ANALYZE-001",
    "sku": "HTL-MADRID-001",
    "name": "Habitaciones del Café Central",
    "description": "<p>Las habitaciones tienen café y una vista al mar.</p>",
    "city": "Madrid",
    "country": "España"
  }
  EOF
  ```

- {% include step_label.html %} Valida el documento para evitar que un error JSON se confunda con un problema del analyzer.

  ```bash
  python -m json.tool analyze-spanish-document.json > /dev/null \
    && echo "Documento de análisis válido"
  ```

### Tarea 3.2. Analizar el documento con ambos índices

- {% include step_label.html %} Envía el documento al índice estándar y guarda la respuesta para revisar los tokens sin perder la evidencia.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/index/${STANDARD_FULL_NAME}/analyzeDoc" \
    -d @analyze-spanish-document.json \
    > standard-analysis-response.json
  ```

- {% include step_label.html %} Formatea la respuesta estándar y revisa cómo se trataron las etiquetas HTML, los acentos y las palabras frecuentes.

  ```bash
  python -m json.tool standard-analysis-response.json
  ```

- {% include step_label.html %} Envía el mismo documento al índice personalizado para que la única variable de comparación sea el analyzer asignado a los fields.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/index/${CUSTOM_FULL_NAME}/analyzeDoc" \
    -d @analyze-spanish-document.json \
    > custom-analysis-response.json
  ```

- {% include step_label.html %} Formatea la respuesta personalizada y comprueba las siguientes propiedades sin exigir una raíz exacta del stemmer:

  ```bash
  python -m json.tool custom-analysis-response.json
  ```

**Propiedades que deben revisarse:**

- Las etiquetas `<p>` y `</p>` no deben formar parte de los tokens útiles.
- Las variantes acentuadas deben normalizarse cuando exista un equivalente ASCII.
- Las stop words del español deben reducirse en el resultado.
- Las variantes `habitación` y `habitaciones` deben producir stems comparables.
- El valor completo de `sku` debe conservarse como un único token normalizado a minúsculas.

> Analyze Document muestra cómo el mapping procesaría un documento; no inserta el documento enviado ni modifica la collection.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

{% include support-prompt.html task="tarea3" %}

---

## 🔎 Tarea 4. Comparar búsquedas lingüísticas y exactas

En esta tarea ejecutarás las mismas Match Queries contra ambos índices para observar diferencias causadas por la normalización, las stop words y el stemming.

### Tarea 4.1. Comparar búsqueda sin acento

- {% include step_label.html %} Crea `query-cafe.json` para buscar `cafe` sin acento dentro de description.

  ```bash
  cat > query-cafe.json << 'EOF'
  {
    "query": {
      "match": "cafe",
      "field": "description"
    },
    "size": 10,
    "fields": [
      "sku",
      "name",
      "description",
      "city"
    ]
  }
  EOF
  ```

- {% include step_label.html %} Ejecuta la consulta contra el índice estándar y guarda la respuesta para compararla posteriormente.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/bucket/${CB_BUCKET}/scope/${CB_SCOPE}/index/${STANDARD_INDEX}/query" \
    -d @query-cafe.json \
    > cafe-standard-response.json
  ```

- {% include step_label.html %} Ejecuta la misma consulta contra el índice personalizado, donde asciifolding permite comparar `cafe` con contenido que originalmente contiene `café`.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/bucket/${CB_BUCKET}/scope/${CB_SCOPE}/index/${CUSTOM_INDEX}/query" \
    -d @query-cafe.json \
    > cafe-custom-response.json
  ```

### Tarea 4.2. Comparar singular, plural y stop words

- {% include step_label.html %} Crea `query-habitacion.json` para buscar la forma singular sin acento.

  ```bash
  cat > query-habitacion.json << 'EOF'
  {
    "query": {
      "match": "habitacion",
      "field": "description"
    },
    "size": 10,
    "fields": [
      "sku",
      "name",
      "description"
    ]
  }
  EOF
  ```

- {% include step_label.html %} Ejecuta la consulta sobre ambos índices y conserva las respuestas.

  ```bash
  for index_name in "${STANDARD_INDEX}" "${CUSTOM_INDEX}"; do
    curl -s \
      -u "${CB_ADMIN}:${CB_PASS}" \
      -X POST \
      -H "Content-Type: application/json" \
      "${CB_SEARCH_URL}/api/bucket/${CB_BUCKET}/scope/${CB_SCOPE}/index/${index_name}/query" \
      -d @query-habitacion.json \
      > "habitacion-${index_name}.json"
  done
  ```

- {% include step_label.html %} Crea `query-vista-mar.json` con varias palabras y operator and para comprobar cómo las stop words influyen en los términos obligatorios.

  ```bash
  cat > query-vista-mar.json << 'EOF'
  {
    "query": {
      "match": "con vista al mar",
      "field": "description",
      "operator": "and"
    },
    "size": 10,
    "fields": [
      "sku",
      "name",
      "description"
    ]
  }
  EOF
  ```

- {% include step_label.html %} Ejecuta la búsqueda multi-término sobre el índice personalizado y revisa qué documentos permanecen después de eliminar stop words y aplicar stemming.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/bucket/${CB_BUCKET}/scope/${CB_SCOPE}/index/${CUSTOM_INDEX}/query" \
    -d @query-vista-mar.json \
    | python -m json.tool
  ```

### Tarea 4.3. Validar el analyzer exacto para SKU

- {% include step_label.html %} Crea `query-sku.json` con una Term Query en minúsculas para buscar un SKU almacenado originalmente en mayúsculas.

  ```bash
  cat > query-sku.json << 'EOF'
  {
    "query": {
      "term": "htl-madrid-001",
      "field": "sku"
    },
    "size": 5,
    "fields": [
      "sku",
      "name",
      "city",
      "country"
    ]
  }
  EOF
  ```

- {% include step_label.html %} Ejecuta la consulta contra el índice personalizado y confirma que sku_analyzer preservó el código completo como un token único en minúsculas.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/bucket/${CB_BUCKET}/scope/${CB_SCOPE}/index/${CUSTOM_INDEX}/query" \
    -d @query-sku.json \
    | python -m json.tool
  ```

- {% include step_label.html %} Ejecuta el siguiente script para resumir los conteos de las respuestas guardadas sin depender de grep sobre JSON formateado.

  ```bash
  python - << 'PY'
  import json
  from pathlib import Path

  files = [
      "cafe-standard-response.json",
      "cafe-custom-response.json",
      "habitacion-idx_lab13_spanish_standard.json",
      "habitacion-idx_lab13_spanish_custom.json",
  ]

  for filename in files:
      path = Path(filename)
      if not path.exists():
          print(f"{filename}: archivo no encontrado")
          continue

      with path.open(encoding="utf-8") as file:
          data = json.load(file)

      print(f"{filename}: total_hits={data.get('total_hits', 0)}")
  PY
  ```

> El índice personalizado no tiene que devolver siempre más resultados. La evidencia válida es que normalice variantes esperadas de forma coherente y que las respuestas correspondan al comportamiento observado en Analyze Document.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

{% include support-prompt.html task="tarea4" %}

---

## ✅ Tarea 5. Inspeccionar definiciones y validar la práctica

En esta tarea verificarás que los analyzers estén registrados, confirmarás su asignación a fields y generarás un resumen reproducible de los recursos creados.

### Tarea 5.1. Descargar e inspeccionar las definiciones

- {% include step_label.html %} Descarga la definición del índice personalizado para conservar la versión registrada por Couchbase.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${CB_SEARCH_URL}/api/bucket/${CB_BUCKET}/scope/${CB_SCOPE}/index/${CUSTOM_INDEX}" \
    > custom-index-definition-retrieved.json
  ```

- {% include step_label.html %} Ejecuta un script de Python para localizar los analyzers y mostrar sus pipelines en un formato resumido.

  ```bash
  python - << 'PY'
  import json

  with open("custom-index-definition-retrieved.json", encoding="utf-8") as file:
      data = json.load(file)

  index_def = data.get("indexDef", data)
  mapping = index_def.get("params", {}).get("mapping", {})
  analyzers = mapping.get("analysis", {}).get("analyzers", {})

  print("Analyzers personalizados:")
  for name, analyzer in analyzers.items():
      print(f"\n{name}")
      print("  char_filters:", analyzer.get("char_filters", []))
      print("  tokenizer:", analyzer.get("tokenizer"))
      print("  token_filters:", analyzer.get("token_filters", []))
  PY
  ```

- {% include step_label.html %} Inspecciona qué analyzer fue asignado a name, description y sku.

  ```bash
  python - << 'PY'
  import json

  with open("custom-index-definition-retrieved.json", encoding="utf-8") as file:
      data = json.load(file)

  index_def = data.get("indexDef", data)
  types = index_def.get("params", {}).get("mapping", {}).get("types", {})

  for type_name, type_definition in types.items():
      print(f"Mapping: {type_name}")
      properties = type_definition.get("properties", {})

      for property_name, property_definition in properties.items():
          for field in property_definition.get("fields", []):
              print(
                  f"  {property_name}: "
                  f"type={field.get('type')} "
                  f"analyzer={field.get('analyzer')} "
                  f"store={field.get('store')}"
              )
  PY
  ```

### Tarea 5.2. Ejecutar la validación final

- {% include step_label.html %} Crea `validate-lab13.sh` para verificar imagen, documentos, índices, analyzers y búsquedas básicas.

  {%raw%}
  ```bash
  cat > validate-lab13.sh << 'EOF'
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

  DOC_COUNT=$(curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    "${CB_QUERY_URL}" \
    --data-urlencode 'statement=SELECT RAW COUNT(*) FROM `travel-sample`.search_es.hotels;' \
    | python -c "import json,sys; print(json.load(sys.stdin).get('results',[0])[0])" 2>/dev/null || echo 0)

  if [ "${DOC_COUNT}" -ge 6 ] 2>/dev/null; then
    pass "La collection contiene al menos seis documentos"
  else
    fail "Conteo de documentos: ${DOC_COUNT}"
  fi

  for INDEX_NAME in "${STANDARD_INDEX}" "${CUSTOM_INDEX}"; do
    INDEX_COUNT=$(curl -s \
      -u "${CB_ADMIN}:${CB_PASS}" \
      "${CB_SEARCH_URL}/api/bucket/${CB_BUCKET}/scope/${CB_SCOPE}/index/${INDEX_NAME}/count" \
      | python -c "import json,sys; print(json.load(sys.stdin).get('count',0))" 2>/dev/null || echo 0)

    if [ "${INDEX_COUNT}" -gt 0 ] 2>/dev/null; then
      pass "${INDEX_NAME} contiene documentos"
    else
      fail "${INDEX_NAME} no contiene documentos"
    fi
  done

  SKU_HITS=$(curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    -H "Content-Type: application/json" \
    "${CB_SEARCH_URL}/api/bucket/${CB_BUCKET}/scope/${CB_SCOPE}/index/${CUSTOM_INDEX}/query" \
    -d @query-sku.json \
    | python -c "import json,sys; print(json.load(sys.stdin).get('total_hits',0))" 2>/dev/null || echo 0)

  if [ "${SKU_HITS}" -gt 0 ] 2>/dev/null; then
    pass "El analyzer de SKU devuelve coincidencias"
  else
    fail "La búsqueda de SKU no devolvió coincidencias"
  fi

  if grep -q '"spanish_hotel_analyzer"' custom-index-definition-retrieved.json \
    && grep -q '"sku_analyzer"' custom-index-definition-retrieved.json; then
    pass "La definición contiene ambos analyzers personalizados"
  else
    fail "No se localizaron los dos analyzers en la definición"
  fi

  echo
  echo "Resultado final: ${PASS} verificaciones correctas | ${FAIL} verificaciones fallidas"
  EOF
  ```
  {%endraw%}

- {% include step_label.html %} Asigna permiso de ejecución al script sin moverlo fuera del directorio de la práctica.

  ```bash
  chmod +x validate-lab13.sh
  ```

- {% include step_label.html %} Ejecuta la validación final y corrige cualquier elemento marcado como FAIL antes de considerar terminada la práctica.

  ```bash
  ./validate-lab13.sh
  ```

- {% include step_label.html %} Conserva la collection, los dos índices y todos los archivos generados para revisar posteriormente el pipeline y repetir las consultas sin reconstruir el entorno.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

{% include support-prompt.html task="tarea5" %}

---

## 🛠️ Resolución de problemas

### Problema 1. Couchbase rechaza la creación del scope o de la collection

- {% include step_label.html %} Consulta los scopes existentes para comprobar si el recurso ya fue creado en una ejecución anterior.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${CB_URL}/pools/default/buckets/${CB_BUCKET}/scopes" \
    | python -m json.tool
  ```

- {% include step_label.html %} Si `search_es` y `hotels` ya aparecen, no intentes duplicarlos y continúa con la inserción de documentos.

### Problema 2. La creación del índice responde con un error de analyzer desconocido

- {% include step_label.html %} Revisa que los nombres estén escritos exactamente como aparecen en la definición:

  ```text
  html
  asciifolding
  unicode
  to_lower
  stop_es
  stemmer_es_snowball
  single
  ```

- {% include step_label.html %} Valida nuevamente el JSON y revisa que los analyzers estén dentro de `params.mapping.analysis.analyzers`.

### Problema 3. El índice existe pero reporta count igual a cero

- {% include step_label.html %} Confirma que la collection contenga documentos antes de revisar el Search Service.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    "${CB_QUERY_URL}" \
    --data-urlencode 'statement=SELECT COUNT(*) AS total FROM `travel-sample`.search_es.hotels;' \
    | python -m json.tool
  ```

- {% include step_label.html %} Descarga la definición del índice y verifica que el type mapping sea `search_es.hotels`.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${CB_SEARCH_URL}/api/bucket/${CB_BUCKET}/scope/${CB_SCOPE}/index/${CUSTOM_INDEX}" \
    | python -m json.tool
  ```

### Problema 4. Analyze Document devuelve index not found

- {% include step_label.html %} Comprueba que la URL utilice el nombre completo scoped:

  ```text
  travel-sample.search_es.idx_lab13_spanish_custom
  ```

- {% include step_label.html %} Lista los índices generales y localiza el nombre completo registrado por Couchbase.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    "${CB_SEARCH_URL}/api/index" \
    | python -m json.tool \
    | grep -E "idx_lab13_spanish_standard|idx_lab13_spanish_custom"
  ```

### Problema 5. Los tokens no coinciden exactamente con los ejemplos conceptuales

- {% include step_label.html %} No interpretes el stem como una palabra completa del diccionario, porque Snowball puede producir raíces técnicas.

- {% include step_label.html %} Valida propiedades estables: eliminación de HTML, reducción de stop words, normalización de acentos y convergencia de variantes morfológicas.

### Problema 6. La Term Query de SKU no devuelve resultados

- {% include step_label.html %} Confirma que el documento contenga el SKU esperado y revisa su combinación de mayúsculas y minúsculas.

  ```bash
  curl -s \
    -u "${CB_ADMIN}:${CB_PASS}" \
    -X POST \
    "${CB_QUERY_URL}" \
    --data-urlencode 'statement=SELECT META(h).id, h.sku FROM `travel-sample`.search_es.hotels AS h ORDER BY META(h).id;' \
    | python -m json.tool
  ```

- {% include step_label.html %} Comprueba que sku utilice `sku_analyzer`, cuyo tokenizer single conserva el código completo y cuyo filtro to_lower normaliza el texto.

### Problema 7. `python -m json.tool` no está disponible

- {% include step_label.html %} Revisa qué comando de Python reconoce la terminal.

  ```bash
  python --version
  python3 --version
  ```

- {% include step_label.html %} Sustituye `python` por `python3` cuando esa sea la instalación disponible en el equipo.