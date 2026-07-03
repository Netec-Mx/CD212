# Creación y Configuración de Índices Full Text Search

## 1. Metadatos

| Atributo | Valor |
|---|---|
| **Duración estimada** | 80 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear (Create) |
| **Dataset requerido** | `travel-sample` (bucket completo) |
| **Versión Couchbase** | 7.6.x (Community Edition o Enterprise Trial) |

---

## 2. Descripción General

En este laboratorio crearás y configurarás índices Full Text Search (FTS) avanzados sobre el dataset `travel-sample` de Couchbase. Partirás de la comprensión conceptual de mappings dinámicos versus estáticos, avanzarás hacia la configuración explícita de campos con tipos específicos, analyzers y Store Field Data, e indexarás campos anidados dentro de arrays de documentos JSON. Finalmente, crearás un Index Alias que federe múltiples índices, clonarás un índice existente para crear variantes y explorarás cómo los Flex Indexes permiten al optimizador SQL++ utilizar automáticamente índices FTS.

---

## 3. Objetivos de Aprendizaje

Al finalizar este laboratorio serás capaz de:

- [ ] Configurar mappings explícitos (`dynamic: false`) en un índice FTS especificando tipos de campo (`text`, `number`, `geopoint`) y analyzers apropiados para documentos del tipo `hotel` en `travel-sample`.
- [ ] Habilitar **Store Field Data** en campos seleccionados para recuperar valores directamente desde el índice FTS sin acceder al bucket principal.
- [ ] Indexar campos anidados dentro de arrays (`reviews[].content`, `reviews[].ratings.Overall`) usando child field mappings con dot notation.
- [ ] Crear un **Index Alias** `hotels-and-landmarks` que combine dos índices FTS para búsquedas federadas.
- [ ] Clonar un índice FTS existente y modificar su configuración de analyzer para generar una variante sin partir desde cero.
- [ ] Explicar y demostrar el funcionamiento de **Flex Indexes** para integrar búsqueda FTS con predicados SQL++.

---

## 4. Prerrequisitos

### Conocimiento previo

| Área | Nivel requerido |
|---|---|
| Queries FTS básicas (Práctica 11) | Completado o experiencia equivalente |
| Estructura JSON anidada en Couchbase | Comprensión sólida |
| Analyzers de texto (tokenización conceptual) | Conocimiento básico |
| Web Console de Couchbase — gestión de índices | Familiaridad operativa |
| SQL++ SELECT con JOIN y filtros | Comprensión básica |

### Acceso y configuración requerida

- Couchbase Server 7.6.x en ejecución (nodo único o Docker).
- Bucket `travel-sample` cargado con todos sus scopes y collections (`inventory.hotel`, `inventory.landmark`, `inventory.airport`).
- Servicios habilitados: **Data**, **Query**, **Index**, **Search**.
- Acceso a la Web Console en `http://localhost:8091`.
- `curl` disponible en terminal (versión 7.x o superior).
- Usuario administrador con credenciales conocidas (por defecto: `Administrator` / `password`).

> **Verificación rápida:** Antes de iniciar, abre la Web Console, navega a **Buckets** y confirma que `travel-sample` aparece con estado verde y muestra documentos en el scope `inventory`.

---

## 5. Entorno de Laboratorio

### Hardware mínimo recomendado

| Componente | Mínimo | Recomendado |
|---|---|---|
| RAM | 8 GB | 16 GB |
| CPU | 4 núcleos x86_64 | 8 núcleos |
| Almacenamiento | 20 GB libres (SSD) | 50 GB SSD |
| Red | localhost, puertos 8091-8097 libres | — |
| Pantalla | 1280×768 | 1280×800 o superior |

### Software requerido

| Software | Versión | Uso en este lab |
|---|---|---|
| Couchbase Server | 7.6.x | Servidor principal |
| Navegador web | Chrome/Firefox/Edge 110+ | Web Console |
| `curl` | 7.x+ | Llamadas REST API |
| `cbq` (Couchbase Query Shell) | Incluido con Server | Queries SQL++ |
| Editor de texto | VS Code 1.80+ (o equivalente) | Editar JSON de índices |

### Verificación del entorno

Ejecuta los siguientes comandos en tu terminal para confirmar que el entorno está listo:

```bash
# 1. Verificar que Couchbase responde
curl -s -u Administrator:password http://localhost:8091/pools/default \
  | python3 -m json.tool | grep '"name"'

# 2. Verificar que el servicio Search está activo
curl -s -u Administrator:password http://localhost:8091/pools/default \
  | python3 -c "import sys,json; d=json.load(sys.stdin); \
    [print(n['hostname'], n.get('services','')) for n in d.get('nodes',[])]"

# 3. Verificar que travel-sample tiene documentos de tipo hotel
curl -s -u Administrator:password \
  "http://localhost:8091/pools/default/buckets/travel-sample" \
  | python3 -m json.tool | grep '"itemCount"'
```

**Salida esperada:** El bucket `travel-sample` debe reportar aproximadamente 63,000 documentos. El nodo debe listar `fts` entre sus servicios activos.

---

## 6. Pasos del Laboratorio

---

### Paso 1 — Explorar la Estructura de Documentos `hotel` en travel-sample

**Objetivo:** Comprender la estructura JSON de los documentos que se van a indexar antes de definir los mappings.

#### Instrucciones

1. Abre la Web Console en `http://localhost:8091` e inicia sesión.

2. Navega a **Query** en el menú lateral izquierdo.

3. Ejecuta la siguiente query para examinar un documento `hotel` representativo:

```sql
SELECT META().id AS doc_id,
       name,
       description,
       city,
       country,
       `type`,
       ratings,
       geo,
       reviews[0] AS primer_review
FROM `travel-sample`.inventory.hotel
LIMIT 1;
```

4. Ejecuta una segunda query para entender la estructura del array `reviews`:

```sql
SELECT META().id AS doc_id,
       r.content       AS review_content,
       r.ratings.Overall AS rating_overall,
       r.author        AS review_author
FROM `travel-sample`.inventory.hotel AS h
UNNEST h.reviews AS r
WHERE r.ratings.Overall IS NOT MISSING
LIMIT 5;
```

5. Anota los campos que aparecen en los resultados. Necesitarás esta información para definir el mapping en el Paso 2.

#### Salida esperada

El primer resultado mostrará un documento con campos como:

```json
{
  "doc_id": "hotel_10025",
  "name": "Medway Youth Hostel",
  "description": "40 bed youth hostel...",
  "city": "Medway",
  "country": "United Kingdom",
  "type": "hotel",
  "geo": { "lat": 51.35785, "lon": 0.55818, "accuracy": "ROOFTOP" },
  "primer_review": {
    "content": "Great hostel...",
    "author": "travel_reviewer_1",
    "ratings": { "Overall": 4, "Rooms": 3, "Service": 5 }
  }
}
```

#### Verificación

```sql
-- Contar documentos hotel disponibles
SELECT COUNT(*) AS total_hoteles
FROM `travel-sample`.inventory.hotel;
```

Debe retornar aproximadamente **917** documentos.

---

### Paso 2 — Crear el Índice FTS con Mapping Explícito para Hoteles

**Objetivo:** Crear un índice FTS con `dynamic: false` y campos explícitamente configurados con tipos y analyzers apropiados.

#### Instrucciones

1. En la Web Console, navega a **Search** en el menú lateral.

2. Haz clic en **Add Index** (botón azul en la esquina superior derecha).

3. Completa los campos del formulario principal:
   - **Index Name:** `idx-hotel-explicit`
   - **Bucket:** `travel-sample`
   - **Scope:** `inventory`
   - **Collection:** `hotel`

4. En la sección **Type Mappings**, elimina el mapping `_default` que aparece por defecto (haz clic en la X junto a él) y luego haz clic en **Add Type Mapping**.

5. En el diálogo de nuevo type mapping:
   - **Type:** `hotel`
   - Desmarca la opción **dynamic** (o selecciona **only index specified fields**)
   - Haz clic en **OK**

6. Con el type mapping `hotel` creado, haz clic en **+** junto a él para agregar campos. Agrega cada uno de los siguientes campos:

   **Campo `name`:**
   - Field Name: `name`
   - Field Type: `text`
   - Analyzer: `en` (English)
   - Marca **Store** como activado (`true`)
   - Marca **Index** como activado
   - Haz clic en **OK**

   **Campo `description`:**
   - Field Name: `description`
   - Field Type: `text`
   - Analyzer: `en` (English)
   - **Store:** desactivado (`false`)
   - **Index:** activado
   - Haz clic en **OK**

   **Campo `city`:**
   - Field Name: `city`
   - Field Type: `text`
   - Analyzer: `keyword`
   - **Store:** activado (`true`)
   - **Index:** activado
   - Haz clic en **OK**

   **Campo `country`:**
   - Field Name: `country`
   - Field Type: `text`
   - Analyzer: `keyword`
   - **Store:** activado (`true`)
   - **Index:** activado
   - Haz clic en **OK**

   **Campo `ratings` (número):**
   - Field Name: `ratings`
   - Field Type: `number`
   - **Store:** desactivado
   - **Index:** activado
   - Haz clic en **OK**

   **Campo `geo` (geopoint):**
   - Field Name: `geo`
   - Field Type: `geopoint`
   - **Store:** desactivado
   - **Index:** activado
   - Haz clic en **OK**

7. En la sección **Advanced** del formulario, asegúrate de que:
   - **Default Analyzer:** `standard`
   - **Default Type Field:** `type` (Couchbase usará el campo `type` del documento para identificar el type mapping)
   - **Default Mapping:** desactivado (para que solo se indexen documentos que coincidan con el type mapping `hotel`)

8. Haz clic en **Create Index**.

9. Alternativamente, puedes crear el mismo índice vía **REST API** con el siguiente comando (guarda el JSON en un archivo `idx-hotel-explicit.json` primero):

```bash
# Crear el archivo de definición del índice
cat > /tmp/idx-hotel-explicit.json << 'EOF'
{
  "name": "idx-hotel-explicit",
  "type": "fulltext-index",
  "params": {
    "mapping": {
      "default_mapping": {
        "enabled": false,
        "dynamic": false
      },
      "types": {
        "hotel": {
          "enabled": true,
          "dynamic": false,
          "properties": {
            "name": {
              "fields": [
                {
                  "name": "name",
                  "type": "text",
                  "analyzer": "en",
                  "store": true,
                  "index": true,
                  "include_term_vectors": true
                }
              ]
            },
            "description": {
              "fields": [
                {
                  "name": "description",
                  "type": "text",
                  "analyzer": "en",
                  "store": false,
                  "index": true
                }
              ]
            },
            "city": {
              "fields": [
                {
                  "name": "city",
                  "type": "text",
                  "analyzer": "keyword",
                  "store": true,
                  "index": true
                }
              ]
            },
            "country": {
              "fields": [
                {
                  "name": "country",
                  "type": "text",
                  "analyzer": "keyword",
                  "store": true,
                  "index": true
                }
              ]
            },
            "ratings": {
              "fields": [
                {
                  "name": "ratings",
                  "type": "number",
                  "store": false,
                  "index": true
                }
              ]
            },
            "geo": {
              "fields": [
                {
                  "name": "geo",
                  "type": "geopoint",
                  "store": false,
                  "index": true
                }
              ]
            }
          }
        }
      },
      "default_type": "_default",
      "default_analyzer": "standard",
      "default_datetime_parser": "dateTimeOptional",
      "type_field": "type"
    }
  },
  "sourceType": "gocbcore",
  "sourceName": "travel-sample",
  "sourceParams": {
    "scope_name": "inventory",
    "collection_names": ["hotel"]
  },
  "planParams": {
    "maxPartitionsPerPIndex": 1024,
    "indexPartitions": 1
  }
}
EOF

# Crear el índice via REST API
curl -s -X PUT \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d @/tmp/idx-hotel-explicit.json \
  "http://localhost:8094/api/index/idx-hotel-explicit" \
  | python3 -m json.tool
```

#### Salida esperada

La API REST debe responder:

```json
{
  "status": "ok"
}
```

En la Web Console, el índice `idx-hotel-explicit` aparecerá en la lista con estado **indexing** y luego **Ready**.

#### Verificación

```bash
# Verificar el estado del índice
curl -s -u Administrator:password \
  "http://localhost:8094/api/index/idx-hotel-explicit" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
idx = d.get('indexDef', {})
print('Nombre:', idx.get('name'))
print('Tipo:', idx.get('type'))
print('Estado: OK si no hay error')
"

# Verificar conteo de documentos indexados
curl -s -u Administrator:password \
  "http://localhost:8094/api/index/idx-hotel-explicit/count" \
  | python3 -m json.tool
```

El conteo debe aproximarse a **917** documentos (total de hoteles en `inventory.hotel`).

---

### Paso 3 — Configurar Store Field Data y Verificar Recuperación desde el Índice

**Objetivo:** Demostrar que los campos con `store: true` pueden recuperarse directamente desde el índice FTS sin acceder al bucket, y comparar el comportamiento con campos `store: false`.

#### Instrucciones

1. Ejecuta una búsqueda FTS que recupere campos almacenados directamente desde el índice. Usa `curl` para llamar al endpoint de búsqueda:

```bash
curl -s -X POST \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "match": "youth hostel",
      "field": "name"
    },
    "fields": ["name", "city", "country"],
    "size": 5,
    "highlight": {
      "style": "html",
      "fields": ["name"]
    }
  }' \
  "http://localhost:8094/api/index/idx-hotel-explicit/query" \
  | python3 -m json.tool
```

2. Observa en la respuesta la sección `fields` de cada hit. Los campos `name`, `city` y `country` (configurados con `store: true`) aparecerán con sus valores. El campo `description` (configurado con `store: false`) **no** aparecerá aunque esté indexado.

3. Ahora intenta recuperar el campo `description` (que tiene `store: false`):

```bash
curl -s -X POST \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "match": "youth hostel",
      "field": "description"
    },
    "fields": ["name", "description", "city"],
    "size": 3
  }' \
  "http://localhost:8094/api/index/idx-hotel-explicit/query" \
  | python3 -m json.tool
```

4. Observa que `description` **no** aparece en la sección `fields` de los resultados, aunque la búsqueda sí encuentra documentos que contienen "youth hostel" en ese campo.

5. Desde la Web Console, navega a **Search → idx-hotel-explicit** y haz clic en **Search** para usar la interfaz gráfica. Ingresa `youth hostel` en el campo de búsqueda y observa los resultados.

#### Salida esperada

La respuesta de la primera query debe incluir en cada hit una sección similar a:

```json
{
  "id": "hotel_10025",
  "score": 1.234,
  "fields": {
    "name": "Medway Youth Hostel",
    "city": "Medway",
    "country": "United Kingdom"
  },
  "fragments": {
    "name": ["Medway <b>Youth</b> <b>Hostel</b>"]
  }
}
```

La segunda query mostrará hits pero **sin** el campo `description` en `fields`.

#### Verificación

```bash
# Verificar que la búsqueda retorna resultados y que 'name' está en fields
curl -s -X POST \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{"query":{"match":"hostel","field":"name"},"fields":["name","city"],"size":1}' \
  "http://localhost:8094/api/index/idx-hotel-explicit/query" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
hits = d.get('hits', [])
if hits and 'fields' in hits[0]:
    print('Store Field Data OK. Campos recuperados:', list(hits[0]['fields'].keys()))
else:
    print('ERROR: No se recuperaron campos almacenados')
"
```

---

### Paso 4 — Indexar Campos Anidados con Child Field Mappings

**Objetivo:** Configurar el índice para indexar campos dentro del array `reviews`, específicamente `reviews[].content` y `reviews[].ratings.Overall`.

#### Instrucciones

1. Primero, crea un nuevo índice `idx-hotel-reviews` que incluya child mappings para el array `reviews`. Guarda la siguiente definición JSON:

```bash
cat > /tmp/idx-hotel-reviews.json << 'EOF'
{
  "name": "idx-hotel-reviews",
  "type": "fulltext-index",
  "params": {
    "mapping": {
      "default_mapping": {
        "enabled": false,
        "dynamic": false
      },
      "types": {
        "hotel": {
          "enabled": true,
          "dynamic": false,
          "properties": {
            "name": {
              "fields": [
                {
                  "name": "name",
                  "type": "text",
                  "analyzer": "en",
                  "store": true,
                  "index": true
                }
              ]
            },
            "city": {
              "fields": [
                {
                  "name": "city",
                  "type": "text",
                  "analyzer": "keyword",
                  "store": true,
                  "index": true
                }
              ]
            },
            "reviews": {
              "enabled": true,
              "dynamic": false,
              "properties": {
                "content": {
                  "fields": [
                    {
                      "name": "content",
                      "type": "text",
                      "analyzer": "en",
                      "store": false,
                      "index": true
                    }
                  ]
                },
                "author": {
                  "fields": [
                    {
                      "name": "author",
                      "type": "text",
                      "analyzer": "keyword",
                      "store": true,
                      "index": true
                    }
                  ]
                },
                "ratings": {
                  "enabled": true,
                  "dynamic": false,
                  "properties": {
                    "Overall": {
                      "fields": [
                        {
                          "name": "Overall",
                          "type": "number",
                          "store": true,
                          "index": true
                        }
                      ]
                    }
                  }
                }
              }
            }
          }
        }
      },
      "default_type": "_default",
      "default_analyzer": "standard",
      "type_field": "type"
    }
  },
  "sourceType": "gocbcore",
  "sourceName": "travel-sample",
  "sourceParams": {
    "scope_name": "inventory",
    "collection_names": ["hotel"]
  },
  "planParams": {
    "maxPartitionsPerPIndex": 1024,
    "indexPartitions": 1
  }
}
EOF

# Crear el índice
curl -s -X PUT \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d @/tmp/idx-hotel-reviews.json \
  "http://localhost:8094/api/index/idx-hotel-reviews" \
  | python3 -m json.tool
```

2. Espera a que el índice termine de indexar (aproximadamente 30-60 segundos). Verifica el conteo:

```bash
curl -s -u Administrator:password \
  "http://localhost:8094/api/index/idx-hotel-reviews/count" \
  | python3 -m json.tool
```

3. Ejecuta una búsqueda en el contenido de las reseñas:

```bash
curl -s -X POST \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "match": "excellent service",
      "field": "reviews.content"
    },
    "fields": ["name", "city", "reviews.author"],
    "size": 5
  }' \
  "http://localhost:8094/api/index/idx-hotel-reviews/query" \
  | python3 -m json.tool
```

4. Ejecuta una búsqueda combinada que filtre por contenido de reseña Y por rating numérico:

```bash
curl -s -X POST \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "conjuncts": [
        {
          "match": "clean comfortable",
          "field": "reviews.content"
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
    "fields": ["name", "city"],
    "size": 5
  }' \
  "http://localhost:8094/api/index/idx-hotel-reviews/query" \
  | python3 -m json.tool
```

#### Salida esperada

La búsqueda en `reviews.content` debe retornar hoteles cuyos documentos contienen reseñas con las palabras "excellent" y "service". La respuesta incluirá el campo `reviews.author` (con `store: true`) en la sección `fields`.

La búsqueda combinada retornará hoteles con reseñas que mencionan "clean comfortable" Y tienen un rating Overall entre 4 y 5.

#### Verificación

```bash
# Verificar que la búsqueda en campo anidado funciona
curl -s -X POST \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{"query":{"match":"great","field":"reviews.content"},"size":1}' \
  "http://localhost:8094/api/index/idx-hotel-reviews/query" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
total = d.get('total_hits', 0)
print(f'Total hits en reviews.content: {total}')
if total > 0:
    print('Child field mapping para reviews.content: FUNCIONAL')
else:
    print('ADVERTENCIA: No se encontraron resultados. Verificar indexación.')
"
```

---

### Paso 5 — Crear un Índice FTS para Landmarks y Configurar un Index Alias

**Objetivo:** Crear un segundo índice para documentos `landmark` y luego crear un Index Alias `hotels-and-landmarks` que permita búsquedas federadas sobre ambos índices simultáneamente.

#### Instrucciones

**Parte A: Crear el índice para landmarks**

```bash
cat > /tmp/idx-landmark.json << 'EOF'
{
  "name": "idx-landmark",
  "type": "fulltext-index",
  "params": {
    "mapping": {
      "default_mapping": {
        "enabled": false,
        "dynamic": false
      },
      "types": {
        "landmark": {
          "enabled": true,
          "dynamic": false,
          "properties": {
            "name": {
              "fields": [
                {
                  "name": "name",
                  "type": "text",
                  "analyzer": "en",
                  "store": true,
                  "index": true
                }
              ]
            },
            "content": {
              "fields": [
                {
                  "name": "content",
                  "type": "text",
                  "analyzer": "en",
                  "store": false,
                  "index": true
                }
              ]
            },
            "city": {
              "fields": [
                {
                  "name": "city",
                  "type": "text",
                  "analyzer": "keyword",
                  "store": true,
                  "index": true
                }
              ]
            },
            "country": {
              "fields": [
                {
                  "name": "country",
                  "type": "text",
                  "analyzer": "keyword",
                  "store": true,
                  "index": true
                }
              ]
            },
            "geo": {
              "fields": [
                {
                  "name": "geo",
                  "type": "geopoint",
                  "store": false,
                  "index": true
                }
              ]
            }
          }
        }
      },
      "default_type": "_default",
      "default_analyzer": "standard",
      "type_field": "type"
    }
  },
  "sourceType": "gocbcore",
  "sourceName": "travel-sample",
  "sourceParams": {
    "scope_name": "inventory",
    "collection_names": ["landmark"]
  },
  "planParams": {
    "maxPartitionsPerPIndex": 1024,
    "indexPartitions": 1
  }
}
EOF

curl -s -X PUT \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d @/tmp/idx-landmark.json \
  "http://localhost:8094/api/index/idx-landmark" \
  | python3 -m json.tool
```

**Parte B: Crear el Index Alias `hotels-and-landmarks`**

```bash
cat > /tmp/alias-hotels-landmarks.json << 'EOF'
{
  "name": "hotels-and-landmarks",
  "type": "fulltext-alias",
  "params": {
    "targets": {
      "idx-hotel-explicit": {},
      "idx-landmark": {}
    }
  },
  "sourceType": "nil",
  "sourceName": ""
}
EOF

curl -s -X PUT \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d @/tmp/alias-hotels-landmarks.json \
  "http://localhost:8094/api/index/hotels-and-landmarks" \
  | python3 -m json.tool
```

**Parte C: Verificar el alias desde la Web Console**

1. En la Web Console, navega a **Search**.
2. Verifica que `hotels-and-landmarks` aparece en la lista con el ícono de alias (diferente al ícono de índice regular).
3. Haz clic en el alias y observa que muestra los índices objetivo.

**Parte D: Ejecutar una búsqueda federada sobre el alias**

```bash
# Búsqueda sobre el alias (busca en hoteles Y landmarks simultáneamente)
curl -s -X POST \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "match": "museum",
      "field": "name"
    },
    "fields": ["name", "city", "country"],
    "size": 10
  }' \
  "http://localhost:8094/api/index/hotels-and-landmarks/query" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f'Total hits en alias (hoteles + landmarks): {d.get(\"total_hits\", 0)}')
for h in d.get('hits', [])[:5]:
    fields = h.get('fields', {})
    print(f'  [{h[\"id\"]}] {fields.get(\"name\",\"N/A\")} - {fields.get(\"city\",\"N/A\")}')
"
```

#### Salida esperada

El alias debe retornar resultados combinados de ambos índices. Los IDs de documentos comenzarán con `hotel_` o `landmark_`, demostrando la búsqueda federada.

#### Verificación

```bash
# Verificar que el alias existe y apunta a los dos índices
curl -s -u Administrator:password \
  "http://localhost:8094/api/index/hotels-and-landmarks" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
idx_def = d.get('indexDef', {})
params = idx_def.get('params', {})
targets = params.get('targets', {})
print('Tipo de índice:', idx_def.get('type'))
print('Índices objetivo:', list(targets.keys()))
assert 'idx-hotel-explicit' in targets, 'ERROR: idx-hotel-explicit no está en el alias'
assert 'idx-landmark' in targets, 'ERROR: idx-landmark no está en el alias'
print('Alias configurado correctamente con ambos índices.')
"
```

---

### Paso 6 — Clonar un Índice FTS y Modificar su Configuración

**Objetivo:** Demostrar cómo clonar un índice existente para crear una variante con diferente configuración de analyzer, sin partir desde cero.

#### Instrucciones

**Método A: Clonación desde la Web Console**

1. En la Web Console, navega a **Search**.
2. Localiza el índice `idx-hotel-explicit`.
3. Haz clic en el menú de opciones del índice (ícono de tres puntos o botón **Clone**).
4. Asigna el nombre `idx-hotel-standard-analyzer` al clon.
5. Haz clic en **Clone Index**.
6. Una vez creado el clon, haz clic en **Edit** sobre el nuevo índice.
7. Modifica el campo `name` para cambiar su analyzer de `en` a `standard`.
8. Guarda los cambios.

**Método B: Clonación via REST API (recomendado para automatización)**

```bash
# Paso 1: Obtener la definición actual del índice
curl -s -u Administrator:password \
  "http://localhost:8094/api/index/idx-hotel-explicit" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
idx = d['indexDef']
# Remover campos que no se deben incluir al crear
for field in ['uuid', 'sourceUUID']:
    idx.pop(field, None)
# Cambiar el nombre
idx['name'] = 'idx-hotel-standard-analyzer'
# Modificar el analyzer del campo 'name' de 'en' a 'standard'
props = idx['params']['mapping']['types']['hotel']['properties']
if 'name' in props:
    for f in props['name']['fields']:
        if f.get('analyzer') == 'en':
            f['analyzer'] = 'standard'
            print('Analyzer del campo name cambiado a: standard', file=sys.stderr)
print(json.dumps(idx, indent=2))
" > /tmp/idx-hotel-standard-analyzer.json

# Verificar el archivo generado
echo "=== Verificando definición del clon ==="
python3 -c "
import json
with open('/tmp/idx-hotel-standard-analyzer.json') as f:
    d = json.load(f)
print('Nombre:', d['name'])
analyzer = d['params']['mapping']['types']['hotel']['properties']['name']['fields'][0]['analyzer']
print('Analyzer del campo name:', analyzer)
"

# Paso 2: Crear el índice clonado con la configuración modificada
curl -s -X PUT \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d @/tmp/idx-hotel-standard-analyzer.json \
  "http://localhost:8094/api/index/idx-hotel-standard-analyzer" \
  | python3 -m json.tool
```

**Comparación de comportamiento entre ambos índices:**

```bash
# Búsqueda con el índice original (analyzer 'en' - stemming inglés)
echo "=== Búsqueda con analyzer 'en' (stemming) ==="
curl -s -X POST \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{"query":{"match":"hotels","field":"name"},"size":3,"fields":["name"]}' \
  "http://localhost:8094/api/index/idx-hotel-explicit/query" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f'Hits (en analyzer): {d[\"total_hits\"]}')
for h in d.get('hits', []):
    print(f'  {h[\"fields\"].get(\"name\",\"\")}')
"

# Búsqueda con el índice clonado (analyzer 'standard' - sin stemming)
echo "=== Búsqueda con analyzer 'standard' (sin stemming) ==="
curl -s -X POST \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{"query":{"match":"hotels","field":"name"},"size":3,"fields":["name"]}' \
  "http://localhost:8094/api/index/idx-hotel-standard-analyzer/query" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f'Hits (standard analyzer): {d[\"total_hits\"]}')
for h in d.get('hits', []):
    print(f'  {h[\"fields\"].get(\"name\",\"\")}')
"
```

#### Salida esperada

Con el analyzer `en` (que aplica stemming), la búsqueda de `"hotels"` también encontrará documentos con `"hotel"` (forma raíz). Con el analyzer `standard` (sin stemming específico del inglés), la búsqueda de `"hotels"` puede retornar menos resultados porque no aplica la reducción a la raíz morfológica.

#### Verificación

```bash
# Confirmar que ambos índices existen
curl -s -u Administrator:password "http://localhost:8094/api/index" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
indices = list(d.get('indexDefs', {}).get('indexDefs', {}).keys())
print('Índices FTS disponibles:')
for idx in sorted(indices):
    print(f'  - {idx}')
assert 'idx-hotel-explicit' in indices
assert 'idx-hotel-standard-analyzer' in indices
print('Clonación verificada correctamente.')
"
```

---

### Paso 7 — Flex Indexes: Integración de FTS con SQL++

**Objetivo:** Demostrar cómo los Flex Indexes permiten al optimizador de queries de Couchbase usar automáticamente un índice FTS cuando una query SQL++ contiene predicados de texto, y cómo usar explícitamente la función `SEARCH()`.

#### Instrucciones

**Parte A: Uso explícito de SEARCH() en SQL++**

1. Abre `cbq` o usa la interfaz **Query** de la Web Console.

2. Ejecuta la siguiente query que usa `SEARCH()` explícitamente:

```sql
-- Búsqueda FTS explícita usando SEARCH() dentro de SQL++
SELECT META(h).id AS doc_id,
       h.name,
       h.city,
       h.country,
       SEARCH_SCORE(h) AS relevance_score
FROM `travel-sample`.inventory.hotel AS h
WHERE SEARCH(h, {
  "query": {
    "match": "boutique luxury",
    "field": "name",
    "analyzer": "en"
  },
  "index": "idx-hotel-explicit"
})
ORDER BY SEARCH_SCORE(h) DESC
LIMIT 10;
```

3. Ejecuta una query con `SEARCH()` que combine predicados FTS y filtros SQL++:

```sql
-- Combinación de FTS y predicados SQL++ estándar
SELECT META(h).id AS doc_id,
       h.name,
       h.city,
       h.country,
       SEARCH_SCORE(h) AS score
FROM `travel-sample`.inventory.hotel AS h
WHERE SEARCH(h, {
  "query": {
    "match": "historic charming",
    "field": "description"
  },
  "index": "idx-hotel-explicit"
})
AND h.country = "United Kingdom"
ORDER BY SEARCH_SCORE(h) DESC
LIMIT 5;
```

**Parte B: Configurar un Flex Index para uso automático por el optimizador**

Para que el optimizador SQL++ use automáticamente el índice FTS, el índice debe estar configurado como un **Flex Index**. Esto requiere que el índice tenga habilitada la opción de compatibilidad con el servicio Query.

```bash
# Crear un índice FTS compatible con Flex Index
cat > /tmp/idx-hotel-flex.json << 'EOF'
{
  "name": "idx-hotel-flex",
  "type": "fulltext-index",
  "params": {
    "mapping": {
      "default_mapping": {
        "enabled": false,
        "dynamic": false
      },
      "types": {
        "inventory.hotel": {
          "enabled": true,
          "dynamic": false,
          "properties": {
            "name": {
              "fields": [
                {
                  "name": "name",
                  "type": "text",
                  "analyzer": "en",
                  "store": true,
                  "index": true
                }
              ]
            },
            "description": {
              "fields": [
                {
                  "name": "description",
                  "type": "text",
                  "analyzer": "en",
                  "store": false,
                  "index": true
                }
              ]
            },
            "city": {
              "fields": [
                {
                  "name": "city",
                  "type": "text",
                  "analyzer": "keyword",
                  "store": true,
                  "index": true
                }
              ]
            }
          }
        }
      },
      "default_type": "_default",
      "default_analyzer": "standard",
      "type_field": "_type"
    }
  },
  "sourceType": "gocbcore",
  "sourceName": "travel-sample",
  "sourceParams": {
    "scope_name": "inventory",
    "collection_names": ["hotel"]
  },
  "planParams": {
    "maxPartitionsPerPIndex": 1024,
    "indexPartitions": 1
  }
}
EOF

curl -s -X PUT \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d @/tmp/idx-hotel-flex.json \
  "http://localhost:8094/api/index/idx-hotel-flex" \
  | python3 -m json.tool
```

**Parte C: Usar SEARCH() con el índice Flex**

```sql
-- Query que usa el índice FTS para búsqueda de texto
-- El optimizador puede usar idx-hotel-flex automáticamente
SELECT META(h).id,
       h.name,
       h.city,
       SEARCH_SCORE(h) AS fts_score
FROM `travel-sample`.inventory.hotel AS h
WHERE SEARCH(h.name, "historic")
  AND h.city IS NOT MISSING
ORDER BY fts_score DESC
LIMIT 10;
```

**Parte D: Analizar el plan de ejecución con EXPLAIN**

```sql
-- Verificar que el optimizador usa el índice FTS
EXPLAIN
SELECT META(h).id, h.name, h.city
FROM `travel-sample`.inventory.hotel AS h
WHERE SEARCH(h, {
  "query": {"match": "luxury", "field": "name"},
  "index": "idx-hotel-explicit"
})
LIMIT 5;
```

Observa en el plan de ejecución el operador `IndexFtsSearch` que indica el uso del índice FTS.

#### Salida esperada

La query con `SEARCH()` debe retornar hoteles ordenados por relevancia (`SEARCH_SCORE`). El `EXPLAIN` debe mostrar un plan con el operador `IndexFtsSearch` referenciando `idx-hotel-explicit`.

#### Verificación

```sql
-- Verificar que SEARCH() funciona y retorna scores
SELECT COUNT(*) AS total,
       AVG(SEARCH_SCORE(h)) AS avg_score,
       MAX(SEARCH_SCORE(h)) AS max_score
FROM `travel-sample`.inventory.hotel AS h
WHERE SEARCH(h, {
  "query": {"match": "hotel", "field": "name"},
  "index": "idx-hotel-explicit"
});
```

Debe retornar `total > 0` y `avg_score > 0`, confirmando que FTS está activo y calculando scores de relevancia.

---

### Paso 8 — Comparativa de Rendimiento: Mapping Dinámico vs. Mapping Explícito

**Objetivo:** Crear un índice con mapping dinámico, comparar su tamaño y tiempo de indexación contra el índice con mapping explícito, y analizar las implicaciones de cada enfoque.

#### Instrucciones

1. Crea un índice con mapping dinámico para la misma colección:

```bash
cat > /tmp/idx-hotel-dynamic.json << 'EOF'
{
  "name": "idx-hotel-dynamic",
  "type": "fulltext-index",
  "params": {
    "mapping": {
      "default_mapping": {
        "enabled": true,
        "dynamic": true
      },
      "default_type": "_default",
      "default_analyzer": "standard"
    }
  },
  "sourceType": "gocbcore",
  "sourceName": "travel-sample",
  "sourceParams": {
    "scope_name": "inventory",
    "collection_names": ["hotel"]
  },
  "planParams": {
    "maxPartitionsPerPIndex": 1024,
    "indexPartitions": 1
  }
}
EOF

curl -s -X PUT \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d @/tmp/idx-hotel-dynamic.json \
  "http://localhost:8094/api/index/idx-hotel-dynamic" \
  | python3 -m json.tool
```

2. Espera a que ambos índices terminen de indexar (aproximadamente 1-2 minutos para el dinámico). Monitorea el progreso:

```bash
# Monitorear el estado de indexación cada 10 segundos
for i in 1 2 3 4 5 6; do
  echo "=== Verificación $i ==="
  for idx in idx-hotel-explicit idx-hotel-dynamic; do
    count=$(curl -s -u Administrator:password \
      "http://localhost:8094/api/index/$idx/count" \
      | python3 -c "import sys,json; print(json.load(sys.stdin).get('count',0))")
    echo "  $idx: $count documentos indexados"
  done
  sleep 10
done
```

3. Una vez completada la indexación, obtén las estadísticas de ambos índices:

```bash
# Obtener estadísticas comparativas
for idx in idx-hotel-explicit idx-hotel-dynamic; do
  echo "=== Estadísticas de $idx ==="
  curl -s -u Administrator:password \
    "http://localhost:8094/api/index/$idx" \
    | python3 -c "
import sys, json
d = json.load(sys.stdin)
params = d.get('indexDef', {}).get('params', {})
mapping = params.get('mapping', {})
# Determinar tipo de mapping
default_dynamic = mapping.get('default_mapping', {}).get('dynamic', False)
types = mapping.get('types', {})
print(f'  Mapping dinámico (default): {default_dynamic}')
print(f'  Type mappings definidos: {list(types.keys())}')
"
done
```

4. Ejecuta la misma búsqueda en ambos índices y compara los resultados:

```bash
QUERY='{"query":{"match":"breakfast included","field":"description"},"size":5,"fields":["name"]}'

echo "=== Búsqueda en idx-hotel-explicit (mapping estático) ==="
time curl -s -X POST \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d "$QUERY" \
  "http://localhost:8094/api/index/idx-hotel-explicit/query" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'Hits: {d[\"total_hits\"]}')"

echo ""
echo "=== Búsqueda en idx-hotel-dynamic (mapping dinámico) ==="
time curl -s -X POST \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d "$QUERY" \
  "http://localhost:8094/api/index/idx-hotel-dynamic/query" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'Hits: {d[\"total_hits\"]}')"
```

5. Registra tus observaciones en la siguiente tabla (complétala con los valores obtenidos):

| Métrica | `idx-hotel-explicit` | `idx-hotel-dynamic` |
|---|---|---|
| Documentos indexados | ~917 | ~917 |
| Campos indexados | 6 (explícitos) | Todos (~20+) |
| Tiempo de indexación | (medir) | (medir, mayor) |
| Tamaño estimado del índice | Menor | Mayor (~3-5x) |
| Hits para "breakfast included" | (registrar) | (registrar, puede diferir) |
| Precisión de resultados | Alta (solo campos relevantes) | Variable (puede incluir ruido) |

#### Salida esperada

El índice dinámico tardará más en indexar y ocupará más espacio porque indexa todos los campos del documento (incluyendo campos como `id`, `url`, `email`, etc.). El índice explícito será más eficiente porque solo procesa los 6 campos declarados.

#### Verificación

```bash
# Resumen final de comparación
python3 << 'EOF'
import urllib.request, json, base64

creds = base64.b64encode(b"Administrator:password").decode()
headers = {"Authorization": f"Basic {creds}"}

indices = ["idx-hotel-explicit", "idx-hotel-dynamic"]
print("\n{'='*60}")
print("RESUMEN COMPARATIVO DE ÍNDICES FTS")
print("="*60)

for idx_name in indices:
    url = f"http://localhost:8094/api/index/{idx_name}"
    req = urllib.request.Request(url, headers=headers)
    with urllib.request.urlopen(req) as resp:
        data = json.loads(resp.read())
    
    idx_def = data.get("indexDef", {})
    mapping = idx_def.get("params", {}).get("mapping", {})
    default_dynamic = mapping.get("default_mapping", {}).get("dynamic", False)
    types_count = len(mapping.get("types", {}))
    
    print(f"\nÍndice: {idx_name}")
    print(f"  Mapping dinámico: {default_dynamic}")
    print(f"  Type mappings: {types_count}")
EOF
```

---

## 7. Validación y Pruebas Finales

Ejecuta las siguientes verificaciones para confirmar que todos los componentes del laboratorio funcionan correctamente:

```bash
#!/bin/bash
# Script de validación completa del laboratorio

echo "============================================"
echo "VALIDACIÓN FINAL - Lab 12-00-01"
echo "============================================"

PASS=0
FAIL=0

check() {
  local desc="$1"
  local result="$2"
  if [ "$result" = "OK" ]; then
    echo "  [PASS] $desc"
    PASS=$((PASS+1))
  else
    echo "  [FAIL] $desc -> $result"
    FAIL=$((FAIL+1))
  fi
}

# 1. Verificar existencia de todos los índices
for idx in idx-hotel-explicit idx-hotel-reviews idx-landmark idx-hotel-dynamic idx-hotel-standard-analyzer idx-hotel-flex; do
  status=$(curl -s -o /dev/null -w "%{http_code}" \
    -u Administrator:password \
    "http://localhost:8094/api/index/$idx")
  [ "$status" = "200" ] && check "Índice $idx existe" "OK" || check "Índice $idx existe" "HTTP $status"
done

# 2. Verificar alias
alias_status=$(curl -s -u Administrator:password \
  "http://localhost:8094/api/index/hotels-and-landmarks" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
t = d.get('indexDef', {}).get('type', '')
print('OK' if t == 'fulltext-alias' else f'ERROR: tipo={t}')
")
check "Alias hotels-and-landmarks es tipo fulltext-alias" "$alias_status"

# 3. Verificar que idx-hotel-explicit tiene documentos indexados
count=$(curl -s -u Administrator:password \
  "http://localhost:8094/api/index/idx-hotel-explicit/count" \
  | python3 -c "import sys,json; c=json.load(sys.stdin).get('count',0); print('OK' if c > 500 else f'Solo {c} docs')")
check "idx-hotel-explicit tiene >500 documentos indexados" "$count"

# 4. Verificar que Store Field Data funciona
store_check=$(curl -s -X POST \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{"query":{"match":"hotel","field":"name"},"fields":["name","city"],"size":1}' \
  "http://localhost:8094/api/index/idx-hotel-explicit/query" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
hits = d.get('hits', [])
if hits and 'name' in hits[0].get('fields', {}):
    print('OK')
else:
    print('ERROR: campos no encontrados en fields')
")
check "Store Field Data recupera campos 'name' y 'city'" "$store_check"

# 5. Verificar búsqueda en campo anidado reviews.content
nested_check=$(curl -s -X POST \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{"query":{"match":"great","field":"reviews.content"},"size":1}' \
  "http://localhost:8094/api/index/idx-hotel-reviews/query" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
total = d.get('total_hits', 0)
print('OK' if total > 0 else f'0 hits')
")
check "Búsqueda en campo anidado reviews.content funciona" "$nested_check"

# 6. Verificar búsqueda federada sobre alias
alias_search=$(curl -s -X POST \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{"query":{"match":"museum","field":"name"},"size":5}' \
  "http://localhost:8094/api/index/hotels-and-landmarks/query" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
total = d.get('total_hits', 0)
print('OK' if total > 0 else '0 hits en alias')
")
check "Búsqueda federada sobre alias retorna resultados" "$alias_search"

echo ""
echo "============================================"
echo "RESULTADO: $PASS pasaron | $FAIL fallaron"
echo "============================================"
```

---

## 8. Solución de Problemas

### Problema 1: El índice FTS no indexa documentos (count = 0 después de varios minutos)

**Síntoma:** El endpoint `/api/index/<nombre>/count` retorna `{"count": 0}` incluso después de esperar más de 2 minutos. En la Web Console, el índice muestra estado `Ready` pero con 0 documentos.

**Causa probable:** El `type_field` configurado en el mapping no coincide con el campo real del documento. Por ejemplo, si el mapping usa `"type_field": "docType"` pero los documentos de `travel-sample` usan `"type": "hotel"`, ningún documento coincidirá con el type mapping `hotel`.

**Solución:**

```bash
# 1. Verificar el campo de tipo real en los documentos
curl -s -u Administrator:password \
  "http://localhost:8091/pools/default/buckets/travel-sample/scopes/inventory/collections/hotel/docs/hotel_10025" \
  | python3 -m json.tool | grep -E '"type"|"docType"'

# Si el campo es "type" (no "docType"), actualizar el índice:
curl -s -u Administrator:password \
  "http://localhost:8094/api/index/idx-hotel-explicit" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
idx = d['indexDef']
# Corregir el type_field
idx['params']['mapping']['type_field'] = 'type'
for field in ['uuid', 'sourceUUID']:
    idx.pop(field, None)
print(json.dumps(idx, indent=2))
" > /tmp/idx-hotel-fixed.json

# Actualizar el índice
curl -s -X PUT \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d @/tmp/idx-hotel-fixed.json \
  "http://localhost:8094/api/index/idx-hotel-explicit" \
  | python3 -m json.tool

# Esperar y verificar
sleep 30
curl -s -u Administrator:password \
  "http://localhost:8094/api/index/idx-hotel-explicit/count" \
  | python3 -m json.tool
```

> **Nota:** En `travel-sample`, los documentos de tipo hotel usan el campo `"type": "hotel"`. Asegúrate de que `type_field` en el mapping sea `"type"` y no otro nombre.

---

### Problema 2: La búsqueda sobre el alias retorna error 400 o 0 resultados inesperadamente

**Síntoma:** Al ejecutar una query sobre el alias `hotels-and-landmarks`, la API retorna un error HTTP 400 con mensaje similar a `"alias target index not found"` o `"index not ready"`, o retorna 0 resultados cuando se esperan resultados de ambos índices.

**Causa probable:** Uno o más de los índices objetivo del alias aún están en estado `indexing` (no han terminado de construirse) o fueron eliminados. El alias no puede servir queries si alguno de sus índices objetivo no está en estado `Ready`.

**Solución:**

```bash
# 1. Verificar el estado de todos los índices objetivo
for idx in idx-hotel-explicit idx-landmark; do
  echo "=== Estado de $idx ==="
  curl -s -u Administrator:password \
    "http://localhost:8094/api/index/$idx/count" \
    | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f'  Documentos indexados: {d.get(\"count\", 0)}')
"
done

# 2. Si un índice fue eliminado, recrearlo antes de usar el alias
# (Usar los comandos del Paso 2 o Paso 5 según corresponda)

# 3. Si los índices existen pero el alias falla, verificar la definición del alias
curl -s -u Administrator:password \
  "http://localhost:8094/api/index/hotels-and-landmarks" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
targets = d.get('indexDef', {}).get('params', {}).get('targets', {})
print('Targets actuales del alias:', list(targets.keys()))
"

# 4. Si es necesario, recrear el alias con los índices correctos
curl -s -X PUT \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{
    "name": "hotels-and-landmarks",
    "type": "fulltext-alias",
    "params": {
      "targets": {
        "idx-hotel-explicit": {},
        "idx-landmark": {}
      }
    },
    "sourceType": "nil",
    "sourceName": ""
  }' \
  "http://localhost:8094/api/index/hotels-and-landmarks" \
  | python3 -m json.tool
```

---

## 9. Limpieza del Entorno

Una vez completado el laboratorio, ejecuta los siguientes comandos para eliminar los índices creados y liberar recursos:

```bash
#!/bin/bash
# Limpieza de índices FTS creados en el laboratorio

echo "Iniciando limpieza de índices FTS del Lab 12-00-01..."

INDICES=(
  "hotels-and-landmarks"
  "idx-hotel-explicit"
  "idx-hotel-reviews"
  "idx-landmark"
  "idx-hotel-dynamic"
  "idx-hotel-standard-analyzer"
  "idx-hotel-flex"
)

for idx in "${INDICES[@]}"; do
  echo -n "  Eliminando $idx... "
  status=$(curl -s -o /dev/null -w "%{http_code}" \
    -X DELETE \
    -u Administrator:password \
    "http://localhost:8094/api/index/$idx")
  
  if [ "$status" = "200" ]; then
    echo "OK"
  elif [ "$status" = "404" ]; then
    echo "No existía (omitido)"
  else
    echo "Error HTTP $status"
  fi
done

# Limpiar archivos temporales
rm -f /tmp/idx-hotel-explicit.json \
      /tmp/idx-hotel-reviews.json \
      /tmp/idx-landmark.json \
      /tmp/idx-hotel-dynamic.json \
      /tmp/idx-hotel-standard-analyzer.json \
      /tmp/idx-hotel-flex.json \
      /tmp/alias-hotels-landmarks.json \
      /tmp/idx-hotel-fixed.json

echo ""
echo "Limpieza completada."

# Verificar que no quedan índices del lab
echo ""
echo "Índices FTS restantes en el servidor:"
curl -s -u Administrator:password "http://localhost:8094/api/index" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
indices = list(d.get('indexDefs', {}).get('indexDefs', {}).keys())
if indices:
    for i in sorted(indices): print(f'  - {i}')
else:
    print('  (ninguno)')
"
```

> **Nota:** El bucket `travel-sample` y sus datos **no** se eliminan. Solo se eliminan los índices FTS creados durante este laboratorio. Los datos permanecen disponibles para laboratorios posteriores.

---

## 10. Resumen

### Conceptos Clave Aprendidos

En este laboratorio completaste la configuración avanzada de índices Full Text Search en Couchbase, cubriendo los siguientes conceptos fundamentales:

| Concepto | Lo que aprendiste |
|---|---|
| **Mapping explícito (`dynamic: false`)** | Controla exactamente qué campos se indexan, reduciendo el tamaño del índice hasta en un 65-75% comparado con mapping dinámico. |
| **Tipos de campo FTS** | `text`, `number`, `geopoint` requieren configuración diferente; el tipo determina qué operaciones de búsqueda son posibles sobre ese campo. |
| **Store Field Data** | Campos con `store: true` permiten recuperar valores desde el índice sin acceder al bucket, mejorando el rendimiento de queries de solo lectura. |
| **Child Field Mappings** | Los campos dentro de arrays (`reviews[].content`) se indexan mediante mappings jerárquicos anidados usando la estructura `properties` recursiva. |
| **Index Alias** | Un alias puede apuntar a múltiples índices FTS, permitiendo búsquedas federadas transparentes sobre diferentes colecciones o tipos de documentos. |
| **Clonación de índices** | La API REST permite obtener la definición de un índice existente, modificarla y crear un nuevo índice, acelerando la creación de variantes de configuración. |
| **Flex Indexes y SEARCH()** | La función `SEARCH()` en SQL++ permite combinar búsqueda FTS con predicados relacionales estándar, y el optimizador puede usar índices FTS automáticamente cuando están correctamente configurados. |
| **Analyzers y su impacto** | El analyzer `en` aplica stemming (reduces `"hotels"` → `"hotel"`), mientras que `standard` no lo hace; `keyword` trata el campo como un token único sin tokenizar. |

### Comparativa Final: Mapping Dinámico vs. Estático

```
┌─────────────────────────────────────────────────────────────────┐
│           MAPPING DINÁMICO vs. MAPPING ESTÁTICO                 │
├──────────────────────┬───────────────────┬──────────────────────┤
│ Característica       │ Dinámico          │ Estático (Explícito) │
├──────────────────────┼───────────────────┼──────────────────────┤
│ Configuración        │ Mínima            │ Requiere planificación│
│ Campos indexados     │ Todos             │ Solo los declarados  │
│ Tamaño del índice    │ Grande (~100%)    │ Reducido (~25-35%)   │
│ Velocidad indexación │ Más lenta         │ Más rápida           │
│ Precisión resultados │ Variable          │ Alta (controlada)    │
│ Uso recomendado      │ Prototipado       │ Producción           │
└──────────────────────┴───────────────────┴──────────────────────┘
```

### Recursos Adicionales

- [Documentación Couchbase FTS: Configuración de Mappings](https://docs.couchbase.com/server/current/fts/fts-creating-indexes.html)
- [Documentación Couchbase FTS: Type Mappings](https://docs.couchbase.com/server/current/fts/fts-type-mappings.html)
- [Documentación Couchbase FTS: Store Field Data](https://docs.couchbase.com/server/current/fts/fts-index-best-practices.html)
- [Documentación Couchbase FTS: Index Aliases](https://docs.couchbase.com/server/current/fts/fts-aliases.html)
- [Documentación Couchbase: Flex Indexes y SEARCH()](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/searchfun.html)
- [REST API para Full Text Search](https://docs.couchbase.com/server/current/rest-api/rest-fts-indexing.html)
- [Guía de Analyzers en Couchbase FTS](https://docs.couchbase.com/server/current/fts/fts-analyzers.html)

---
