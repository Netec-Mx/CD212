---
layout: lab
title: "Práctica 10: Ejecución de búsquedas básicas con Couchbase Search"
permalink: /lab10/lab10/
images_base: /labs/lab10/img
duration: "50 minutos"
objective:
  - Preparar el directorio de trabajo de la práctica 10 y validar que los servicios Web Console, Query y Search estén disponibles.
  - Establecer una línea base con SQL++ LIKE para identificar las limitaciones de las búsquedas de texto libre.
  - Crear un índice Search scoped sobre travel-sample.inventory.hotel con mapping estático y campos controlados.
  - Ejecutar consultas match, match_phrase, fuzzy, prefix y compuestas con boosts.
  - Interpretar total_hits, score, max_score, fragments y fields en respuestas FTS.
  - Ejecutar consultas Search mediante la REST API y comparar sus resultados con SQL++.
prerequisites:
  - Haber completado la Práctica 3 sobre consultas SQL++.
  - Tener Couchbase Server Enterprise 7.6.2 en ejecución mediante la imagen couchbase/server:enterprise-7.6.2.
  - Tener habilitados los servicios Data, Query, Index y Search.
  - Tener cargado el bucket travel-sample.
  - Tener disponible la collection travel-sample.inventory.hotel.
  - Tener acceso a la Web Console en http://localhost:8091.
  - Utilizar Visual Studio Code con Git Bash como terminal integrada.
  - Comprender documentos JSON, índices y consultas SQL++ básicas.
introduction:
  - En esta práctica utilizarás Couchbase Server Enterprise Edition para comparar una búsqueda de texto libre realizada con SQL++ LIKE frente a Couchbase Search. Crearás un índice Search scoped con mapping estático, configurarás analizadores adecuados para texto y campos exactos, ejecutarás distintos tipos de consultas FTS y analizarás la relevancia de los resultados. Finalmente reproducirás las consultas mediante la REST API para comprender cómo integrar Search desde una aplicación.
slug: lab10
lab_number: 10
final_result: >
  Al finalizar la práctica habrás creado el índice idx_lab10_hotel_search sobre la collection hotel, ejecutado búsquedas match, phrase, fuzzy, prefix y compuestas, interpretado scores y fragmentos resaltados, y comparado de forma práctica cuándo utilizar SQL++ y cuándo Couchbase Search.
notes:
  - Todos los comandos de terminal deben ejecutarse desde Git Bash dentro de Visual Studio Code.
  - Utiliza las credenciales Administrator y Password123! configuradas en las prácticas anteriores.
  - El índice Search debe conservarse al finalizar porque podrá utilizarse en prácticas posteriores.
  - El número exacto de resultados puede variar según la versión del dataset y la configuración del índice.
  - Los tiempos observados son referencias del entorno de laboratorio y no constituyen un benchmark formal.
references: []
prev: /lab9/lab9/
next: /lab11/lab11/
---

---
<!-- Aquí comienzan las instrucciones paso a paso de la práctica -->

## 📁 Preparación del directorio de trabajo

En esta práctica conservarás el directorio raíz del curso y crearás únicamente el subdirectorio correspondiente a `lab10`.

### 🗂️ Crear y abrir el subdirectorio de la práctica

- {% include step_label.html %} Abre **Docker Desktop** y confirma que el motor indique estado activo, porque `couchbase-lab` depende del daemon local para ejecutar todos sus servicios.
- {% include step_label.html %} Abre **Visual Studio Code** y espera su carga completa, ya que utilizarás el Explorador y la terminal integrada durante toda la práctica.
- {% include step_label.html %} Abre el directorio raíz desde Visual Studio Code para utilizar `C:\LABS\couchbase-nosql` como referencia visible y comprobar que coincida con el estado o valor requerido.

  ```text
  C:\LABS\couchbase-nosql
  ```

- {% include step_label.html %} Selecciona **Terminal → New Terminal** en Visual Studio Code para abrir la consola integrada desde la que ejecutarás las operaciones de la práctica.
- {% include step_label.html %} Comprueba en el selector del panel Terminal que **Git Bash** sea el perfil activo, porque los comandos utilizan sintaxis y rutas compatibles con Bash.
- {% include step_label.html %} Crea el subdirectorio desde Git Bash para crear de forma idempotente el directorio `/c/LABS/couchbase-nosql/lab10` donde se organizarán los archivos de esta práctica.

  ```bash
  mkdir -p /c/LABS/couchbase-nosql/lab10
  ```

- {% include step_label.html %} Cambia al subdirectorio desde Git Bash para cambiar la ubicación activa a `/c/LABS/couchbase-nosql/lab10` y evitar operaciones posteriores desde un directorio incorrecto.

  ```bash
  cd /c/LABS/couchbase-nosql/lab10
  ```

- {% include step_label.html %} Confirma tu ubicación desde Git Bash para mostrar la ruta activa y confirmar que Git Bash está ubicado en el subdirectorio asignado a esta práctica.

  ```bash
  pwd
  ```

**Salida esperada:**

Para validar `crear y abrir el subdirectorio de trabajo`, verifica la referencia siguiente y confirma que la respuesta permita mostrar la ruta activa y confirmar que Git Bash está ubicado en el subdirectorio asignado a esta práctica; detente si aparece un error.

```text
/c/LABS/couchbase-nosql/lab10
```

---

## 🔎 Tarea 1. Verificar Search y establecer una línea base con SQL++

En esta tarea confirmarás que Couchbase Search esté disponible y ejecutarás una búsqueda de texto mediante SQL++ LIKE para identificar sus limitaciones.

### Tarea 1.1. Definir variables de entorno

- {% include step_label.html %} Define en Git Bash las variables de conexión, credenciales, servicios e índice para reutilizar valores coherentes en todas las solicitudes de la práctica.

Ejecuta:



```bash
export CB_HOST="localhost"
export CB_ADMIN="Administrator"
export CB_PASS="Password123!"
export CB_URL="http://${CB_HOST}:8091"
export CB_QUERY_URL="http://${CB_HOST}:8093/query/service"
export CB_SEARCH_URL="http://${CB_HOST}:8094"
export FTS_INDEX="idx_lab10_hotel_search"
```

### Tarea 1.2. Verificar el contenedor

- {% include step_label.html %} Consulta en Git Bash el estado de `couchbase-lab` y confirma que aparezca activo antes de acceder a los servicios de Couchbase.

{%raw%}
```bash
docker ps --filter "name=couchbase-lab" \
  --format "table {{.Names}}\t{{.Status}}"
```
{%endraw%}

Si el contenedor no está activo:

- {% include step_label.html %} Inicia `couchbase-lab` solamente si el listado anterior indica que está detenido y espera la confirmación de Docker antes de continuar.

```bash
docker start couchbase-lab
```

### Tarea 1.3. Verificar que el contenedor utiliza Enterprise Edition

- {% include step_label.html %} Inspecciona la imagen configurada en `couchbase-lab` y verifica que sea exactamente `couchbase/server:enterprise-7.6.2`.

{%raw%}
```bash
docker inspect couchbase-lab --format "Imagen activa: {{.Config.Image}}"
```
{%endraw%}

**Salida esperada:**

Para validar `Verificar que el contenedor utiliza Enterprise Edition`, verifica la referencia siguiente y confirma que la respuesta permita consultar la configuración de `couchbase-lab` y verificar que utiliza la imagen Enterprise 7.6.2 requerida; detente si aparece un error.

```text
Imagen activa: couchbase/server:enterprise-7.6.2
```

> **IMPORTANTE:** Si aparece `couchbase/server:community-7.6.2`, el contenedor anterior de Community Edition continúa activo. Vuelve a la Práctica 2 y recrea el entorno con Enterprise Edition antes de continuar.
{: .lab-note .important .compact}

### Tarea 1.4. Verificar Web Console, Query y Search

- {% include step_label.html %} Solicita la página principal de Web Console y confirma el código HTTP 200, que demuestra disponibilidad del servicio administrativo.

```bash
curl -s -o /dev/null \
  -w "Web Console: HTTP %{http_code}\n" \
  "${CB_URL}/ui/index.html"
```

- {% include step_label.html %} Consulta `/admin/ping` del servicio Query con autenticación y confirma HTTP 200 antes de enviar instrucciones SQL++.

```bash
curl -sS -o /dev/null \
  -u "${CB_ADMIN}:${CB_PASS}" \
  -w "Query Service: HTTP %{http_code}\n" \
  "${CB_QUERY_URL%/query/service}/admin/ping"
```

- {% include step_label.html %} Consulta `/api/index` del servicio Search con autenticación y confirma HTTP 200 para validar que Full Text Search responde.

```bash
curl -s -o /dev/null \
  -u "${CB_ADMIN}:${CB_PASS}" \
  -w "Search Service: HTTP %{http_code}\n" \
  "${CB_SEARCH_URL}/api/index"
```

**Resultado esperado:**

Para validar `Verificar Web Console, Query y Search`, verifica la referencia siguiente y confirma que la respuesta permita solicitar `el endpoint REST indicado` y revisar su código HTTP, estado o campos JSON antes de continuar; detente si aparece un error.

```text
Web Console: HTTP 200
Query Service: HTTP 200
Search Service: HTTP 200
```

### Tarea 1.5. Verificar la collection hotel

- {% include step_label.html %} Consulta mediante Query REST el total de documentos en `travel-sample.inventory.hotel` y conserva `total` como referencia del índice.

Ejecuta:



```bash
curl -s \
  -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_QUERY_URL}" \
  --data-urlencode \
  'statement=SELECT COUNT(*) AS total
             FROM `travel-sample`.inventory.hotel;' \
  | python -m json.tool
```

Registra el valor de `total`. Ese número será la referencia para comparar con los documentos indexados.

### Tarea 1.6. Ejecutar la búsqueda SQL++ de referencia

- {% include step_label.html %} Consulta en Query Workbench hoteles con `luxury` o `airport` en `description` y registra resultados, orden, tiempo y operador del plan.

Abre la Web Console y entra en **Query** para utilizar Query Workbench con el keyspace de ejemplo cargado.



```sql
SELECT h.name,
       h.city,
       h.country,
       h.description
FROM `travel-sample`.inventory.hotel AS h
WHERE LOWER(h.description) LIKE "%luxury%"
   OR LOWER(h.description) LIKE "%airport%"
ORDER BY h.name
LIMIT 10;
```

Observa:

- Cantidad de resultados mostrados.
- Tiempo aproximado.
- Orden de salida.
- Tipo de operador mostrado en el plan.

### Tarea 1.7. Probar una búsqueda con error tipográfico

- {% include step_label.html %} Repite la consulta SQL++ con `luxurry` y comprueba que `LIKE` compara caracteres sin aplicar tolerancia ortográfica ni relevancia.

```sql
SELECT h.name,
       h.description
FROM `travel-sample`.inventory.hotel AS h
WHERE LOWER(h.description) LIKE "%luxurry%"
LIMIT 5;
```

No debes asumir un resultado específico. El propósito es demostrar que `LIKE` compara caracteres y no realiza corrección ortográfica.

### Tarea 1.8. Registrar observaciones

- {% include step_label.html %} Crea `search-comparison.md` y registra las limitaciones observadas en SQL++ respecto a análisis lingüístico, score y errores tipográficos.

Crea el archivo:

```text
search-comparison.md
```

Agrega:

```markdown
# Comparación SQL++ y Couchbase Search

## Línea base con SQL++

| Criterio | Observación |
|---|---|
| Resultados ordenados por relevancia | No |
| Análisis lingüístico | No |
| Corrección de errores tipográficos | No |
| Score de relevancia | No |
| Tipo de acceso observado en el plan | |
| Tiempo aproximado | |
```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

{% include support-prompt.html task="tarea1" %}

---

## 🧱 Tarea 2. Crear el índice Search scoped

En esta tarea crearás un índice Search limitado a travel-sample.inventory.hotel. Se utilizará mapping estático para indexar solo los campos necesarios.

### Tarea 2.1. Abrir Search

- {% include step_label.html %} Abre `http://localhost:8091` desde Visual Studio Code para acceder a la Web Console local del contenedor Couchbase Enterprise.

  ```text
  http://localhost:8091
  ```

- {% include step_label.html %} Inicia sesión en la Web Console con `Administrator` y `Password123!` para acceder al clúster mediante las credenciales administrativas definidas.

| Campo | Valor |
|---|---|
| Usuario | `Administrator` |
| Contraseña | `Password123!` |

- {% include step_label.html %} Selecciona **Search** en la navegación lateral de la Web Console para abrir la administración de índices Full Text Search del clúster.
- {% include step_label.html %} En **Search**, selecciona **Add Index** o **Create Search Index** para abrir el formulario donde definirás el índice scoped de `hotel`.

### Tarea 2.2. Configurar el bucket y el scope

- {% include step_label.html %} Configura el índice `idx_lab10_hotel_search` sobre el bucket `travel-sample` y el scope `inventory`, con cero réplicas y una partición.

En la pantalla **Add Search Index**, configura los parámetros generales:

| Sección de la interfaz | Campo | Valor |
|---|---|---|
| General | Index Name | `idx_lab10_hotel_search` |
| General | Bucket | `travel-sample` |
| Customize Index | Use non-default scope/collection(s) | Activado |
| Customize Index | Scope | `inventory` |
| Advanced | Index Replicas | `0` |
| Advanced | Index Partitions | `1` |

Mantén el motor de almacenamiento predeterminado:

| Parámetro interno | Valor |
|---|---|
| Storage | `scorch` |

> **NOTA:** En esta versión de la interfaz, `scorch` no se selecciona desde un campo visible en la parte principal. Puede confirmarse en **Index Definition Preview**, donde aparece `"indexType": "scorch"`.
{: .lab-note .info .compact}

### Tarea 2.3. Crear el Type Mapping para `hotel`

En la sección **Mappings** aparece inicialmente:

```text
# default | dynamic
```

Ese mapping predeterminado incluiría campos y documentos de forma dinámica, por lo que no debe permanecer activo para este índice.

- {% include step_label.html %} En **Mappings**, desactiva `# default | dynamic` para impedir la indexación automática de campos y limitar el índice al mapping estático de `hotel`.

- {% include step_label.html %} En **Mappings**, selecciona **+ Add Type Mapping** para asociar el índice con la collection `hotel` y configurar únicamente sus campos requeridos.

- {% include step_label.html %} Selecciona la colección `hotel`, el analizador `standard` y las opciones estáticas indicadas para limitar los documentos y campos indexados.

| Campo de la interfaz | Valor |
|---|---|
| Collection | `hotel` |
| Analyzer | `standard` |
| Only index specified fields | Activado |
| Enabled | Activado |

- {% include step_label.html %} Confirma el diálogo con **OK** y verifica que `hotel | static` aparezca en Mappings, sin depender del campo documental `type`.

Al finalizar, en **Mappings** debe aparecer un mapping correspondiente a la colección `hotel`, identificado aproximadamente como:

```text
hotel | static
```

> **IMPORTANTE:** No escribas un valor basado en el campo `type`. El índice ya queda limitado por el Type Mapping de la colección `hotel`; no se necesita agregar un identificador documental adicional.
{: .lab-note .important .compact}

La opción **Only index specified fields** convierte el mapping en estático. Esto permite indexar únicamente los campos que agregues manualmente y evita incluir todos los campos de los documentos.

### Tarea 2.4. Agregar el campo `name`

- {% include step_label.html %} En **Mappings**, coloca el puntero sobre el mapping `hotel` para mostrar sus controles y agregar el child field solicitado en esta subtarea.

- {% include step_label.html %} Selecciona el botón **+** del mapping `hotel` para abrir el menú de elementos que pueden agregarse a la definición estática.

- {% include step_label.html %} En el mapping `hotel`, selecciona **+ → Insert child field** para añadir el campo indicado con su tipo, analizador y opciones de indexación.

- {% include step_label.html %} Define `name` como texto con analizador `standard`, activa Index, Store e inclusión en `_all`, y conserva DocValues predeterminado.

| Campo de la interfaz | Valor |
|---|---|
| Field | `name` |
| Type | `text` |
| Analyzer | `standard` |
| Index | Activado |
| Store | Activado |
| Include in `_all` | Activado |
| DocValues | Mantener valor predeterminado |

- {% include step_label.html %} Confirma `name` con **OK** y comprueba que aparezca como child field de `hotel` con las opciones de indexación establecidas.

Los campos simples como texto, números o arreglos de valores se agregan mediante **Insert child field**. Los objetos JSON anidados requieren un child mapping diferente.

### Tarea 2.5. Agregar el campo `description`

- {% include step_label.html %} Coloca el puntero sobre `hotel` para mostrar sus controles y preparar la incorporación del campo textual `description`.

- {% include step_label.html %} Selecciona **+ → Insert child field** en `hotel` para abrir el formulario de un campo simple destinado a `description`.

- {% include step_label.html %} Define `description` como texto con analizador `en`, activa Index, Store e inclusión en `_all`, y conserva DocValues predeterminado.

| Campo de la interfaz | Valor |
|---|---|
| Field | `description` |
| Type | `text` |
| Analyzer | `en` |
| Index | Activado |
| Store | Activado |
| Include in `_all` | Activado |
| DocValues | Mantener valor predeterminado |

- {% include step_label.html %} Confirma `description` con **OK** y comprueba que aparezca bajo `hotel` con el analizador `en` y las opciones indicadas.

El analizador `en` procesa texto en inglés y aplica reglas lingüísticas como normalización y stemming, lo que mejora búsquedas sobre variantes de una misma palabra.

### Tarea 2.6. Agregar los campos `city` y `country`

Agrega primero `city`:

- {% include step_label.html %} Inserta un child field para `city` y configúralo como texto `keyword`, con Index y Store activos, sin incluirlo en `_all`.

| Campo de la interfaz | Valor |
|---|---|
| Field | `city` |
| Type | `text` |
| Analyzer | `keyword` |
| Index | Activado |
| Store | Activado |
| Include in `_all` | Desactivado |
| DocValues | Activado o valor predeterminado |

- {% include step_label.html %} Confirma `city` con **OK** y comprueba que aparezca bajo `hotel` con analizador `keyword`, Store activo y exclusión de `_all`.

Después agrega `country` mediante el mismo procedimiento:

| Campo de la interfaz | Valor |
|---|---|
| Field | `country` |
| Type | `text` |
| Analyzer | `keyword` |
| Index | Activado |
| Store | Activado |
| Include in `_all` | Desactivado |
| DocValues | Activado o valor predeterminado |

- {% include step_label.html %} Inserta `country` con analizador `keyword`, confirma con **OK** y comprueba que quede almacenado, indexado y excluido de `_all`.

El analizador `keyword` trata el contenido como un término completo. Es apropiado para campos categóricos como ciudad o país cuando se buscan coincidencias exactas.

- {% include step_label.html %} Expande el Type Mapping `hotel` en **Mappings** para revisar los child fields configurados y confirmar que no se agregaron propiedades inesperadas.

- {% include step_label.html %} En el Type Mapping `hotel`, confirma el modo estático y verifica que existan exactamente `name`, `description`, `city` y `country`.

```text
name
description
city
country
```

- {% include step_label.html %} En **Mappings**, comprueba que `# default | dynamic` continúe desactivado para evitar incorporar collections o campos fuera de la definición estática.

- {% include step_label.html %} Contrasta la definición final con la tabla y confirma nombre, keyspace, mapping estático, cuatro campos, réplicas, particiones y `scorch`.

| Elemento | Resultado esperado |
|---|---|
| Index Name | `idx_lab10_hotel_search` |
| Bucket | `travel-sample` |
| Scope | `inventory` |
| Type Mapping | `hotel` |
| Mapping dinámico | Desactivado |
| Campos definidos | `4` |
| Index Replicas | `0` |
| Index Partitions | `1` |
| Storage | `scorch` |

- {% include step_label.html %} Selecciona **Create Index** para guardar la definición e iniciar la indexación; no cierres la Web Console mientras se crea el índice Search.

- {% include step_label.html %} Regresa a la lista de índices y espera que el índice esté disponible, sin errores, y que aumente el conteo de documentos indexados.

**Verificación:** El índice debe quedar asociado al keyspace `travel-sample.inventory.hotel`. No debe mostrar `_default` como colección ni permanecer con el mapping `# default | dynamic` activado.

### Tarea 2.7. Verificar la definición

- {% include step_label.html %} Consulta la definición scoped mediante Search REST y verifica el keyspace, mapping estático, campos, particiones y motor `scorch`.

```bash
curl -s \
  -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${FTS_INDEX}" \
  | python -m json.tool
```

### Tarea 2.8. Verificar el conteo

- {% include step_label.html %} Consulta el recurso `/count` del índice scoped y compara su valor con el total de `hotel` cuando finalice la indexación.

```bash
curl -s \
  -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${FTS_INDEX}/count" \
  | python -m json.tool
```

El conteo debe acercarse al total de la collection hotel cuando la indexación termine.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

{% include support-prompt.html task="tarea2" %}

---

## 🔤 Tarea 3. Ejecutar match, match phrase y fuzzy search

En esta tarea ejecutarás tres consultas básicas y observarás cómo cambian los resultados según la intención de búsqueda.

### Tarea 3.1. Crear match-query.json

- {% include step_label.html %} Crea `match-query.json` con una consulta OR sobre `description`, resaltado HTML y los campos almacenados que se mostrarán en cada hit.

Crea:

```text
match-query.json
```

Agrega:

```json
{
  "query": {
    "match": "pool wifi breakfast",
    "field": "description",
    "operator": "or"
  },
  "size": 5,
  "from": 0,
  "highlight": {
    "style": "html",
    "fields": ["description"]
  },
  "fields": ["name", "city", "country", "description"]
}
```

### Tarea 3.2. Ejecutar match desde la REST API

- {% include step_label.html %} Envía `match-query.json` al endpoint scoped `/query` y revisa `total_hits`, scores, campos almacenados y fragmentos resaltados.

```bash
curl -s \
  -u "${CB_ADMIN}:${CB_PASS}" \
  -X POST \
  -H "Content-Type: application/json" \
  "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${FTS_INDEX}/query" \
  -d @match-query.json \
  | python -m json.tool
```

Identifica:

- `total_hits`
- `max_score`
- `hits`
- `hits[].score`
- `hits[].fields`
- `hits[].fragments`

### Tarea 3.3. Crear phrase-query.json

- {% include step_label.html %} Crea `phrase-query.json` con `match_phrase` sobre `description` para buscar `free parking` como términos consecutivos y ordenados.

```json
{
  "query": {
    "match_phrase": "free parking",
    "field": "description"
  },
  "size": 5,
  "from": 0,
  "highlight": {
    "style": "html",
    "fields": ["description"]
  },
  "fields": ["name", "city", "country", "description"]
}
```

Ejecuta:

- {% include step_label.html %} Envía `phrase-query.json` al endpoint scoped `/query` y verifica que los hits respeten la frase analizada `free parking`.

```bash
curl -s \
  -u "${CB_ADMIN}:${CB_PASS}" \
  -X POST \
  -H "Content-Type: application/json" \
  "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${FTS_INDEX}/query" \
  -d @phrase-query.json \
  | python -m json.tool
```

### Tarea 3.4. Crear fuzzy-query.json

- {% include step_label.html %} Crea `fuzzy-query.json` con el término `luxurry`, campo `description` y fuzziness 1 para tolerar una edición de caracteres.

```json
{
  "query": {
    "match": "luxurry",
    "field": "description",
    "fuzziness": 1
  },
  "size": 5,
  "fields": ["name", "city", "country", "description"]
}
```

Ejecuta:

- {% include step_label.html %} Envía `fuzzy-query.json` al endpoint scoped `/query` y contrasta sus hits con el resultado obtenido mediante SQL++ `LIKE`.

```bash
curl -s \
  -u "${CB_ADMIN}:${CB_PASS}" \
  -X POST \
  -H "Content-Type: application/json" \
  "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${FTS_INDEX}/query" \
  -d @fuzzy-query.json \
  | python -m json.tool
```

### Tarea 3.5. Comparar las tres consultas

- {% include step_label.html %} Completa en `search-comparison.md` la tabla de intención y resultados para distinguir match, frase exacta analizada y tolerancia fuzzy.

Agrega:

```markdown
## Consultas FTS básicas

| Consulta | Intención | Observación |
|---|---|---|
| match | Buscar cualquiera de varios términos | |
| match_phrase | Buscar términos juntos y en orden | |
| fuzzy match | Tolerar una diferencia ortográfica controlada | |
```

> **IMPORTANTE:** Stemming y fuzzy search son conceptos distintos. El stemming normaliza variantes lingüísticas; fuzziness tolera diferencias de caracteres.
{: .lab-note .important .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

{% include support-prompt.html task="tarea3" %}

---

## 🧠 Tarea 4. Ejecutar prefix y una consulta compuesta con boosts

En esta tarea utilizarás una consulta prefix para buscar tokens que comienzan con un prefijo y una consulta compuesta para influir en el ranking.

### Tarea 4.1. Crear prefix-query.json

- {% include step_label.html %} Crea `prefix-query.json` para localizar tokens de `description` que comiencen con `beach` y devolver cinco documentos con campos almacenados.

```json
{
  "query": {
    "prefix": "beach",
    "field": "description"
  },
  "size": 5,
  "fields": ["name", "city", "country", "description"]
}
```

### Tarea 4.2. Ejecutar prefix

- {% include step_label.html %} Envía `prefix-query.json` al endpoint scoped `/query` y confirma que Search evalúe el prefijo sobre tokens indexados, no sobre el campo completo.

```bash
curl -s \
  -u "${CB_ADMIN}:${CB_PASS}" \
  -X POST \
  -H "Content-Type: application/json" \
  "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${FTS_INDEX}/query" \
  -d @prefix-query.json \
  | python -m json.tool
```

`prefix` busca tokens indexados que comienzan con `beach`; no exige que el texto completo del campo comience con esa palabra.

### Tarea 4.3. Crear compound-query.json

- {% include step_label.html %} Crea `compound-query.json` con tres disyunciones y boosts distintos para ponderar coincidencias en `name` y `description`.

```json
{
  "query": {
    "disjuncts": [
      {
        "match": "luxury",
        "field": "name",
        "boost": 3
      },
      {
        "match": "luxury",
        "field": "description",
        "boost": 2
      },
      {
        "match": "airport",
        "field": "description",
        "boost": 1.5
      }
    ],
    "min": 1
  },
  "size": 5,
  "from": 0,
  "highlight": {
    "style": "html",
    "fields": ["name", "description"]
  },
  "fields": ["name", "city", "country", "description"]
}
```

### Tarea 4.4. Ejecutar la consulta compuesta

- {% include step_label.html %} Envía `compound-query.json` al endpoint scoped `/query` y observa cómo los boosts modifican el score y el orden de los hits.

```bash
curl -s \
  -u "${CB_ADMIN}:${CB_PASS}" \
  -X POST \
  -H "Content-Type: application/json" \
  "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${FTS_INDEX}/query" \
  -d @compound-query.json \
  | python -m json.tool
```

### Tarea 4.5. Analizar los resultados

- {% include step_label.html %} Registra los tres primeros hits, sus scores, coincidencias y boosts aplicables para explicar el orden producido por la consulta compuesta.

Selecciona los tres primeros hits y registra:

```markdown
## Análisis de relevancia

| Posición | Hotel | Score | Coincidencia principal | Boost aplicable |
|---|---|---|---|---|
| 1 | | | | |
| 2 | | | | |
| 3 | | | | |
```

El score sirve principalmente para ordenar resultados dentro de la misma consulta. No representa un porcentaje de relevancia.

### Tarea 4.6. Crear explain-query.json

- {% include step_label.html %} Crea `explain-query.json` con `explain: true` y tamaño uno para inspeccionar el cálculo de relevancia del primer resultado.

```json
{
  "query": {
    "match": "luxury airport",
    "field": "description",
    "operator": "or"
  },
  "size": 1,
  "explain": true,
  "fields": ["name", "description"]
}
```

Ejecuta:

- {% include step_label.html %} Envía `explain-query.json` al endpoint scoped `/query` y revisa valor, descripción y detalles de la explicación del score.

```bash
curl -s \
  -u "${CB_ADMIN}:${CB_PASS}" \
  -X POST \
  -H "Content-Type: application/json" \
  "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${FTS_INDEX}/query" \
  -d @explain-query.json \
  | python -m json.tool
```

Revisa:

- `score`
- `explanation.value`
- `explanation.description`
- `explanation.details`

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

{% include support-prompt.html task="tarea4" %}

---

## 📊 Tarea 5. Comparar SQL++ y Couchbase Search

En esta tarea ejecutarás consultas equivalentes y documentarás cuándo utilizar cada tecnología.

### Tarea 5.1. Ejecutar la consulta SQL++ equivalente

- {% include step_label.html %} Consulta en Query Workbench hoteles con `luxury` o `airport` mediante `LIKE` y conserva las diez filas para compararlas con Search.

```sql
SELECT h.name,
       h.city,
       h.country,
       h.description
FROM `travel-sample`.inventory.hotel AS h
WHERE LOWER(h.description) LIKE "%luxury%"
   OR LOWER(h.description) LIKE "%airport%"
LIMIT 10;
```

### Tarea 5.2. Crear equivalent-search.json

- {% include step_label.html %} Crea `equivalent-search.json` con disyunciones para `luxury` y `airport`, mínimo una coincidencia y los mismos campos proyectados en SQL++.

```json
{
  "query": {
    "disjuncts": [
      {
        "match": "luxury",
        "field": "description"
      },
      {
        "match": "airport",
        "field": "description"
      }
    ],
    "min": 1
  },
  "size": 10,
  "fields": ["name", "city", "country", "description"]
}
```

### Tarea 5.3. Ejecutar la consulta Search equivalente

- {% include step_label.html %} Envía `equivalent-search.json` al endpoint scoped `/query` y compara relevancia, orden y scores con las filas obtenidas mediante SQL++.

```bash
curl -s \
  -u "${CB_ADMIN}:${CB_PASS}" \
  -X POST \
  -H "Content-Type: application/json" \
  "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${FTS_INDEX}/query" \
  -d @equivalent-search.json \
  | python -m json.tool
```

### Tarea 5.4. Completar la comparación

- {% include step_label.html %} Completa la tabla final para distinguir filtros estructurados de SQL++ y capacidades textuales de Search, incluidos score, fuzzy y boost.

Agrega:

```markdown
## Comparación final

| Criterio | SQL++ LIKE | Couchbase Search |
|---|---|---|
| Filtro por caracteres | Sí | No directamente |
| Análisis lingüístico | No | Sí |
| Relevancia | No | Sí |
| Score | No | Sí |
| Highlight | No | Sí |
| Fuzzy search | No | Sí, con fuzziness |
| Boost por condición | No | Sí |
| Filtros estructurados y agregaciones | Sí | Limitados |
| Uso recomendado | Datos estructurados | Texto libre |
```

### Tarea 5.5. Validar definición, conteo y consulta

- {% include step_label.html %} Recupera la definición scoped del índice y confirma que el mapping de `hotel`, sus cuatro campos y el motor `scorch` permanezcan correctos.

```bash
curl -s \
  -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${FTS_INDEX}" \
  | python -m json.tool
```

- {% include step_label.html %} Recupera `/count` y confirma que el índice contenga documentos antes de efectuar la última consulta de funcionamiento.

```bash
curl -s \
  -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${FTS_INDEX}/count" \
  | python -m json.tool
```

- {% include step_label.html %} Envía una consulta match de tamaño uno para `pool` y confirma que `/query` responda con estructura válida y al menos un hit.

```bash
curl -s \
  -u "${CB_ADMIN}:${CB_PASS}" \
  -X POST \
  -H "Content-Type: application/json" \
  "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${FTS_INDEX}/query" \
  -d '{"query":{"match":"pool","field":"description"},"size":1}' \
  | python -m json.tool
```

### Tarea 5.6. Escribir una conclusión

- {% include step_label.html %} Redacta una conclusión que recomiende SQL++ para condiciones estructuradas y Search para texto libre con relevancia y análisis lingüístico.

Agrega:

```markdown
## Conclusión

SQL++ debe utilizarse principalmente para filtros exactos, rangos,
agregaciones y operaciones estructuradas.

Couchbase Search debe utilizarse cuando se requiere búsqueda de texto
libre, relevancia, análisis lingüístico, fuzzy search, prefix search,
highlight y boosts.
```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

{% include support-prompt.html task="tarea5" %}

---

## 🛠️ Resolución de problemas

### Problema 1. Search Service responde con Connection refused

Verifica el puerto:

- {% include step_label.html %} Solicita `/api/index` con salida detallada de curl para comprobar conexión al puerto 8094, autenticación y respuesta del servicio Search.

```bash
curl -v \
  -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_SEARCH_URL}/api/index"
```

Si no responde, el contenedor fue configurado sin Search. Revisa los servicios activos desde la Web Console.

### Problema 2. El índice existe pero tiene count igual a 0

Verifica:

- Bucket `travel-sample`.
- Scope `inventory`.
- Collection `hotel`.
- Default mapping habilitado.
- Dynamic deshabilitado.
- Fields agregados correctamente.

Revisa la definición:

- {% include step_label.html %} Recupera la definición scoped y revisa bucket, scope, colección, mapping y campos para localizar la causa de un conteo igual a cero.

```bash
curl -s \
  -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_SEARCH_URL}/api/bucket/travel-sample/scope/inventory/index/${FTS_INDEX}" \
  | python -m json.tool
```

### Problema 3. Los fields no aparecen en la respuesta

Confirma que `Store` esté habilitado para:

```text
name
description
city
country
```

El campo puede estar indexado pero no almacenado.

### Problema 4. No aparecen fragments

Confirma que la consulta incluya:

```json
"highlight": {
  "style": "html",
  "fields": ["description"]
}
```

También verifica que el término exista en el campo solicitado.

### Problema 5. fuzzy search no devuelve resultados

Prueba primero:

```json
{
  "query": {
    "match": "luxury",
    "field": "description"
  },
  "size": 5
}
```

Después cambia a:

```json
{
  "query": {
    "match": "luxurry",
    "field": "description",
    "fuzziness": 1
  },
  "size": 5
}
```

### Problema 6. La URL REST devuelve Index not found

Verifica que uses la ruta scoped:

```text
/api/bucket/travel-sample/scope/inventory/index/idx_lab10_hotel_search
```

No utilices únicamente:

```text
/api/index/idx_lab10_hotel_search
```

para este índice scoped.

### Problema 7. El contenedor activo utiliza Community Edition

**Síntoma:** la validación de imagen muestra `couchbase/server:community-7.6.2`.

**Solución:**

- {% include step_label.html %} Inspecciona la imagen de `couchbase-lab` y confirma Enterprise 7.6.2; si muestra Community, recrea el contenedor antes de continuar.

{%raw%}
```bash
docker inspect couchbase-lab   --format "Imagen activa: {{.Config.Image}}"
```
{%endraw%}

La salida correcta debe ser:

```text
Imagen activa: couchbase/server:enterprise-7.6.2
```

Si continúa apareciendo Community Edition, vuelve a la Práctica 2 y recrea el contenedor con Enterprise Edition antes de ejecutar esta práctica.

### Problema 8. python -m json.tool falla

Ejecuta:

- {% include step_label.html %} Consulta las versiones disponibles de `python` y `python3` para identificar el ejecutable instalado en Git Bash antes de procesar JSON.

```bash
python --version
python3 --version
```

Si `python3` funciona, reemplaza:

- {% include step_label.html %} Prueba el módulo `json.tool` con el comando `python` para confirmar si ese intérprete puede formatear respuestas JSON recibidas por curl.

```bash
python -m json.tool
```

por:

- {% include step_label.html %} Sustituye el formateador por `python3 -m json.tool` cuando esa variante sea la disponible y confirma que la salida JSON resulte legible.

```bash
python3 -m json.tool
```