---
layout: lab
title: "Práctica 3: Consultas básicas y filtrado de documentos con SQL++"
permalink: /lab3/lab3/
images_base: /labs/lab3/img
duration: "80 minutos"
objective:
  - Preparar el directorio de trabajo de la práctica 3 y validar que Couchbase Server Enterprise continúe disponible.
  - Ejecutar consultas SQL++ con SELECT, proyección de campos, LIMIT, OFFSET y ordenamiento estable.
  - Aplicar alias, concatenación, filtros y selección directa de documentos mediante USE KEYS.
  - Utilizar comparadores, BETWEEN, LIKE, DISTINCT y funciones de agregación sobre el dataset travel-sample.
  - Trabajar con campos MISSING y NULL, agrupaciones, HAVING, arreglos JSON con UNNEST y JOIN entre collections.
prerequisites:
  - Haber completado la Práctica 2.
  - Tener Docker Desktop en ejecución.
  - Tener activo el contenedor couchbase-lab con la imagen Couchbase Server Enterprise 7.6.2.
  - Tener cargado el bucket travel-sample.
  - Tener acceso a la Web Console en http://localhost:8091.
  - Utilizar Visual Studio Code con Git Bash como terminal integrada.
introduction:
  - En esta práctica trabajarás con SQL++ sobre el dataset travel-sample de Couchbase. Comenzarás con consultas básicas y paginación ordenada, avanzarás hacia filtros, alias, concatenación, comparadores y agregaciones, y terminarás consultando arreglos JSON con UNNEST y relacionando documentos de route y airline mediante JOIN. Todo el flujo se ejecutará de forma progresiva desde la Web Console y la terminal integrada de Visual Studio Code con Git Bash.
slug: lab3
lab_number: 3
final_result: >
  Al finalizar la práctica habrás ejecutado consultas SQL++ progresivas sobre las collections airline, airport y route del bucket travel-sample. Podrás seleccionar y ordenar documentos, aplicar filtros y funciones de agregación, diferenciar campos MISSING y NULL, descomponer arreglos JSON con UNNEST y enriquecer rutas mediante un JOIN con la collection airline.
notes:
  - Todos los comandos de terminal deben ejecutarse desde Git Bash dentro de Visual Studio Code.
  - No elimines ni detengas el contenedor couchbase-lab al finalizar la práctica.
  - Los resultados exactos pueden variar ligeramente según la versión del dataset travel-sample. Valida la estructura y las condiciones de cada consulta.
references: []
prev: /lab2/lab2/
next: /lab4/lab4/
---

---
<!-- Aquí comienzan las instrucciones paso a paso de la práctica -->

## 📁 Preparación del directorio de trabajo

En esta práctica conservarás el directorio raíz creado en la práctica 2 y únicamente crearás el subdirectorio correspondiente a `lab3`.

### 🗂️ Crear y abrir el subdirectorio de la práctica

- {% include step_label.html %} Abre **Docker Desktop** y confirma que el motor indique estado activo, porque el contenedor de Couchbase necesita el daemon local para iniciar y publicar sus puertos.
- {% include step_label.html %} Abre **Visual Studio Code** y espera su carga completa, ya que utilizarás el Explorador y la terminal integrada para desarrollar todas las actividades de esta práctica.
- {% include step_label.html %} En el menú **File → Open Folder** de VS Code, abre `C:\LABS\couchbase-nosql` para conservar `lab3` junto con los demás directorios del curso.

  ```text
  C:\LABS\couchbase-nosql
  ```

- {% include step_label.html %} Selecciona **Terminal → New Terminal** en VS Code para abrir la consola integrada desde la cual administrarás el contenedor y ejecutarás las validaciones.
- {% include step_label.html %} Comprueba en el selector del panel Terminal que el perfil activo sea **Git Bash**, porque los comandos utilizan rutas y utilidades compatibles con Bash.
- {% include step_label.html %} Ejecuta el comando siguiente en Git Bash para crear `lab3` dentro del directorio raíz sin generar un error cuando la carpeta ya exista.

  ```bash
  mkdir -p /c/LABS/couchbase-nosql/lab3
  ```

- {% include step_label.html %} Ejecuta `cd` en Git Bash para establecer `lab3` como directorio activo y evitar que los archivos posteriores se creen fuera de la práctica.

  ```bash
  cd /c/LABS/couchbase-nosql/lab3
  ```

- {% include step_label.html %} Ejecuta `pwd` en Git Bash y confirma que la salida sea `/c/LABS/couchbase-nosql/lab3` antes de iniciar cualquier operación con Couchbase.

  ```bash
  pwd
  ```

**Salida esperada:**

Confirma que `pwd` devuelva exactamente `/c/LABS/couchbase-nosql/lab3`; una ruta diferente indica que debes corregir la ubicación antes de continuar.

```text
/c/LABS/couchbase-nosql/lab3
```

> **IMPORTANTE:** Conserva el directorio raíz `C:\LABS\couchbase-nosql`. En las prácticas siguientes solo crearás un nuevo subdirectorio dentro de esta ubicación.
{: .lab-note .important .compact}

---

## 🔎 Tarea 1. Preparar el entorno y ejecutar consultas básicas

En esta tarea confirmarás que Couchbase Server Enterprise y el dataset `travel-sample` continúan disponibles. Después ejecutarás consultas con `SELECT`, proyección de campos, `LIMIT`, `OFFSET` y paginación ordenada.

### Tarea 1.1. Verificar que el contenedor couchbase-lab está activo

- {% include step_label.html %} Ejecuta `docker ps` en Git Bash para localizar `couchbase-lab` y comprobar en las columnas de estado y puertos que el contenedor continúa activo.

  {%raw%}
  ```bash
  docker ps --filter "name=couchbase-lab" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
  ```
  {%endraw%}

**Salida esperada:**

Comprueba que la fila de `couchbase-lab` muestre `Up` en `STATUS`; si no aparece o figura detenido, inicia el contenedor antes de seguir.

```text
NAMES           STATUS
couchbase-lab   Up ...
```

- {% include step_label.html %} Ejecuta `docker inspect` en Git Bash para leer la imagen configurada y confirma que el valor sea exactamente `couchbase/server:enterprise-7.6.2`.

  {%raw%}
  ```bash
  docker inspect couchbase-lab --format '{{.Config.Image}}'
  ```
  {%endraw%}

**Salida esperada:**

Verifica que la salida sea `couchbase/server:enterprise-7.6.2`; cualquier etiqueta Community o versión distinta requiere volver a la práctica 2.

```text
couchbase/server:enterprise-7.6.2
```

> **IMPORTANTE:** Si aparece `couchbase/server:community-7.6.2`, el contenedor todavía corresponde a Community Edition. Regresa a la Práctica 2 y realiza el reemplazo del contenedor antes de continuar.
{: .lab-note .important .compact}

- {% include step_label.html %} Ejecuta `docker start couchbase-lab` únicamente si el contenedor está detenido; espera la confirmación de Docker antes de validar sus servicios.

  ```bash
  docker start couchbase-lab
  ```

- {% include step_label.html %} Ejecuta la solicitud `curl` en Git Bash contra el puerto 8091 y confirma el código HTTP 200, que demuestra que la Web Console ya responde.

  ```bash
  curl -s -o /dev/null -w "Web Console: HTTP %{http_code}\n" \
    http://localhost:8091/ui/index.html
  ```

**Salida esperada:**

Confirma que la respuesta muestre `Web Console: HTTP 200`; otro código o una salida vacía indica que Couchbase aún no está disponible en 8091.

```text
Web Console: HTTP 200
```

### Tarea 1.2. Verificar el dataset travel-sample

- {% include step_label.html %} Abre `http://localhost:8091` en el navegador para acceder a la Web Console del nodo local y espera a que aparezca la pantalla de autenticación.

  ```text
  http://localhost:8091
  ```

- {% include step_label.html %} Introduce `Administrator` y `Password123!` en la pantalla de inicio de sesión para acceder al clúster con las credenciales definidas en la práctica 2.

| Campo | Valor |
|---|---|
| Usuario | `Administrator` |
| Contraseña | `Password123!` |

- {% include step_label.html %} Selecciona **Query** en la navegación lateral de la Web Console para abrir Query Workbench, donde ejecutarás las sentencias SQL++ de esta práctica.
- {% include step_label.html %} Ejecuta la consulta de conteo en Query Workbench para confirmar que `travel-sample.inventory.airline` contiene documentos disponibles para los ejercicios.

  ```sql
  SELECT RAW COUNT(*)
  FROM `travel-sample`.inventory.airline;
  ```

**Salida esperada aproximada:**

Comprueba que el arreglo contenga un número mayor que cero; el valor puede variar, pero cero indica que `travel-sample` no está listo.

```json
[
  187
]
```

> **NOTA:** El número exacto puede variar. Lo importante es obtener un valor mayor a 0.
{: .lab-note .info .compact}

### Tarea 1.3. Ejecutar SELECT con documentos completos

- {% include step_label.html %} Ejecuta la consulta en Query Workbench para recuperar cinco documentos completos de `airline` y observar el objeto que SQL++ genera con `SELECT *`.

  ```sql
  SELECT *
  FROM `travel-sample`.inventory.airline
  ORDER BY META().id
  LIMIT 5;
  ```

**Salida esperada:**

Verifica que se devuelvan cinco objetos y que cada uno contenga la propiedad `airline`; esa envoltura corresponde al resultado de `SELECT *`.

Debes obtener 5 documentos. Cada resultado debe incluir una propiedad `airline` con los campos del documento.

```json
[
  {
    "airline": {
      "id": 10,
      "type": "airline",
      "name": "40-Mile Air",
      "iata": "Q5",
      "country": "United States"
    }
  }
]
```

### Tarea 1.4. Proyectar campos específicos

- {% include step_label.html %} Ejecuta la proyección en Query Workbench para devolver solamente `name`, `country` e `iata`, reduciendo los campos incluidos en cada resultado JSON.

  ```sql
  SELECT name, country, iata
  FROM `travel-sample`.inventory.airline
  ORDER BY META().id
  LIMIT 10;
  ```

**Salida esperada:**

Confirma que cada objeto incluya únicamente `name`, `country` e `iata`; otros campos indicarían que no se ejecutó la proyección mostrada.

```json
[
  {
    "name": "40-Mile Air",
    "country": "United States",
    "iata": "Q5"
  }
]
```

Los resultados deben contener únicamente `name`, `country` e `iata`.

### Tarea 1.5. Aplicar paginación ordenada

- {% include step_label.html %} Ejecuta la primera consulta de paginación en Query Workbench con `OFFSET 0` para obtener los primeros cinco documentos según el identificador estable.

  ```sql
  SELECT META(a).id AS document_id,
         a.name,
         a.country
  FROM `travel-sample`.inventory.airline AS a
  ORDER BY META(a).id
  LIMIT 5
  OFFSET 0;
  ```

- {% include step_label.html %} Ejecuta la segunda consulta con `OFFSET 5` y el mismo `ORDER BY` para recuperar la página siguiente sin repetir identificadores de la primera salida.

  ```sql
  SELECT META(a).id AS document_id,
         a.name,
         a.country
  FROM `travel-sample`.inventory.airline AS a
  ORDER BY META(a).id
  LIMIT 5
  OFFSET 5;
  ```

**Validación:**

Compara ambas páginas: cada una debe contener cinco filas, no deben repetir `document_id` y deben conservar el mismo criterio de ordenamiento.

- Cada consulta debe devolver 5 documentos.
- Las claves `document_id` de la página 1 no deben repetirse en la página 2.
- Ambas consultas deben utilizar exactamente el mismo `ORDER BY`.

> **IMPORTANTE:** La paginación requiere un orden estable. Usar `LIMIT` y `OFFSET` sin `ORDER BY` puede producir resultados inconsistentes.
{: .lab-note .important .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

{% include support-prompt.html task="tarea1" %}

---

## 🧩 Tarea 2. Aplicar alias, concatenación, filtros y USE KEYS

En esta tarea renombrarás campos, construirás cadenas de texto, aplicarás filtros y recuperarás documentos mediante sus claves.

### Tarea 2.1. Renombrar campos con AS

- {% include step_label.html %} Ejecuta la consulta con alias en Query Workbench para renombrar cuatro campos y comprobar cómo `AS` adapta las propiedades del resultado sin modificar documentos.

  ```sql
  SELECT name     AS nombre_aerolinea,
         country  AS pais,
         iata     AS codigo_iata,
         callsign AS indicativo
  FROM `travel-sample`.inventory.airline
  ORDER BY name
  LIMIT 8;
  ```

**Salida esperada:**

Verifica que los resultados utilicen `nombre_aerolinea`, `pais`, `codigo_iata` e `indicativo`, confirmando la aplicación de todos los alias.

```json
[
  {
    "nombre_aerolinea": "40-Mile Air",
    "pais": "United States",
    "codigo_iata": "Q5",
    "indicativo": "MILE-AIR"
  }
]
```

### Tarea 2.2. Concatenar campos

- {% include step_label.html %} Ejecuta la expresión con `||` en Query Workbench para combinar nombre y código IATA, excluyendo valores ausentes o nulos que afectarían la cadena.

  ```sql
  SELECT name || " (" || iata || ")" AS aerolinea_con_codigo,
         country
  FROM `travel-sample`.inventory.airline
  WHERE iata IS NOT MISSING
    AND iata IS NOT NULL
  ORDER BY name
  LIMIT 10;
  ```

**Salida esperada:**

Confirma que `aerolinea_con_codigo` combine nombre e IATA entre paréntesis y que ninguna fila presente una concatenación incompleta.

```json
[
  {
    "aerolinea_con_codigo": "40-Mile Air (Q5)",
    "country": "United States"
  }
]
```

- {% include step_label.html %} Ejecuta la consulta con `CONCAT()` en Query Workbench para construir una descripción con nombre y país, y compara su resultado con el operador `||`.

  ```sql
  SELECT CONCAT(name, " - ", country) AS descripcion_completa,
         icao
  FROM `travel-sample`.inventory.airline
  ORDER BY name
  LIMIT 10;
  ```

### Tarea 2.3. Aplicar filtros con WHERE

- {% include step_label.html %} Ejecuta el filtro por `United States` en Query Workbench y revisa que las diez filas devueltas cumplan la condición establecida en `WHERE`.

  ```sql
  SELECT name, iata, callsign
  FROM `travel-sample`.inventory.airline
  WHERE country = "United States"
  ORDER BY name
  LIMIT 10;
  ```

**Validación:**

Revisa que cada fila tenga `country` igual a `United States`; cualquier otro país indica que el filtro no se aplicó a la consulta ejecutada.

Todos los documentos deben pertenecer a `United States`.

- {% include step_label.html %} Ejecuta el filtro `LIKE "A%"` en Query Workbench para seleccionar códigos IATA que comiencen con A y verificar el comportamiento del comodín `%`.

  ```sql
  SELECT name, iata, country
  FROM `travel-sample`.inventory.airline
  WHERE iata LIKE "A%"
  ORDER BY name
  LIMIT 10;
  ```

**Validación:**

Comprueba que cada valor de `iata` comience con la letra A, ya que `%` debe admitir cualquier secuencia de caracteres después de ese prefijo.

Todos los valores de `iata` deben comenzar con la letra `A`.

### Tarea 2.4. Recuperar documentos con USE KEYS

- {% include step_label.html %} Ejecuta `USE KEYS "airline_10"` en Query Workbench para acceder directamente al documento por su clave, sin recorrer otros elementos de la collection.

  ```sql
  SELECT *
  FROM `travel-sample`.inventory.airline
  USE KEYS "airline_10";
  ```

**Salida esperada:**

Confirma que la consulta devuelva un solo documento correspondiente a `40-Mile Air`, identificado directamente mediante la clave `airline_10`.

Debes obtener un único documento correspondiente a `40-Mile Air`.

- {% include step_label.html %} Ejecuta la consulta con el arreglo de claves en Query Workbench para recuperar hasta tres aerolíneas y contrastar cada clave con `META(a).id`.

  ```sql
  SELECT META(a).id AS document_id,
         a.name,
         a.country,
         a.iata
  FROM `travel-sample`.inventory.airline AS a
  USE KEYS ["airline_10", "airline_10123", "airline_10226"]
  ORDER BY META(a).id;
  ```

**Validación:**

Verifica que se obtengan hasta tres filas y que cada `document_id` pertenezca al arreglo de claves; una clave inexistente puede reducir la cantidad.

Debes obtener hasta 3 documentos. Confirma que cada `document_id` coincide con alguna de las claves solicitadas.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

{% include support-prompt.html task="tarea2" %}

---

## 📊 Tarea 3. Usar comparadores, BETWEEN, LIKE, DISTINCT y agregaciones

En esta tarea consultarás aeropuertos y aerolíneas utilizando rangos, patrones de texto y funciones de agregación.

### Tarea 3.1. Filtrar con comparadores

- {% include step_label.html %} Ejecuta la consulta de aeropuertos en Query Workbench para filtrar altitudes superiores a 5000 pies y ordenar primero los valores más elevados.

  ```sql
  SELECT airportname,
         city,
         country,
         geo.alt AS altitud_pies
  FROM `travel-sample`.inventory.airport
  WHERE geo.alt > 5000
  ORDER BY geo.alt DESC
  LIMIT 10;
  ```

**Validación:**

Comprueba que todas las altitudes sean mayores que 5000 y estén ordenadas de forma descendente; no continúes si alguna fila incumple el filtro.

- Todos los valores de `altitud_pies` deben ser mayores a 5000.
- Los resultados deben aparecer de mayor a menor altitud.

### Tarea 3.2. Aplicar BETWEEN

- {% include step_label.html %} Ejecuta la consulta con `BETWEEN` en Query Workbench para seleccionar altitudes entre 1000 y 3000 pies, incluidos ambos límites del intervalo.

  ```sql
  SELECT airportname,
         city,
         country,
         geo.alt AS altitud_pies
  FROM `travel-sample`.inventory.airport
  WHERE geo.alt BETWEEN 1000 AND 3000
  ORDER BY geo.alt ASC
  LIMIT 15;
  ```

**Validación:**

Confirma que cada altitud esté entre 1000 y 3000, incluidos los extremos, porque `BETWEEN` aplica un intervalo cerrado en SQL++.

Todos los valores deben estar entre 1000 y 3000, incluidos ambos extremos.

### Tarea 3.3. Aplicar LIKE con wildcards

- {% include step_label.html %} Ejecuta el primer patrón `LIKE` en Query Workbench para localizar aerolíneas cuyo nombre contenga la secuencia `Air` en cualquier posición.

  ```sql
  SELECT name, country, iata
  FROM `travel-sample`.inventory.airline
  WHERE name LIKE "%Air%"
  ORDER BY name
  LIMIT 15;
  ```

- {% include step_label.html %} Ejecuta el segundo patrón `LIKE` en Query Workbench para recuperar aeropuertos cuyo nombre comience con `San` y comparar ambos usos del comodín.

  ```sql
  SELECT airportname, city, country
  FROM `travel-sample`.inventory.airport
  WHERE airportname LIKE "San%"
  ORDER BY airportname
  LIMIT 10;
  ```

**Validación:**

Verifica que la primera consulta contenga `Air` en cada nombre y que la segunda comience con `San`; así se distinguen ambos patrones `LIKE`.

- La primera consulta debe devolver nombres que contengan `Air`.
- La segunda debe devolver aeropuertos cuyo nombre comience con `San`.

### Tarea 3.4. Consultar valores distintos

- {% include step_label.html %} Ejecuta `SELECT DISTINCT country` en Query Workbench para eliminar países duplicados y obtener una lista ordenada de valores presentes en `airline`.

  ```sql
  SELECT DISTINCT country
  FROM `travel-sample`.inventory.airline
  WHERE country IS NOT MISSING
  ORDER BY country;
  ```

**Resultado esperado:**

Comprueba que la lista de países esté ordenada y no contenga duplicados, resultado esperado al aplicar `DISTINCT` sobre el campo `country`.

Una lista de países sin duplicados.

- {% include step_label.html %} Ejecuta `COUNT(DISTINCT country)` en Query Workbench para calcular en una sola fila cuántos países diferentes existen dentro de la collection.

  ```sql
  SELECT COUNT(DISTINCT country) AS paises_unicos
  FROM `travel-sample`.inventory.airline;
  ```

### Tarea 3.5. Aplicar funciones de agregación

- {% include step_label.html %} Ejecuta la consulta agregada en Query Workbench para calcular cantidad, promedio, mínimo y máximo de altitud sobre aeropuertos con valores disponibles.

  ```sql
  SELECT COUNT(*)     AS total_aeropuertos,
         AVG(geo.alt) AS altitud_promedio,
         MIN(geo.alt) AS altitud_minima,
         MAX(geo.alt) AS altitud_maxima
  FROM `travel-sample`.inventory.airport
  WHERE geo.alt IS NOT MISSING
    AND geo.alt IS NOT NULL;
  ```

**Salida esperada:**

Confirma que la salida sea un único objeto con las cuatro métricas solicitadas y que los campos numéricos tengan valores calculados.

Debes obtener un único objeto con cuatro métricas.

**Validación:**

Verifica que `altitud_promedio` quede entre `altitud_minima` y `altitud_maxima`; una relación distinta señalaría una ejecución incorrecta.

```text
altitud_minima <= altitud_promedio <= altitud_maxima
```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

{% include support-prompt.html task="tarea3" %}

---

## 🧮 Tarea 4. Trabajar con MISSING, NULL, GROUP BY, HAVING y funciones básicas

En esta tarea diferenciarás campos ausentes de valores nulos, crearás agrupaciones y aplicarás funciones integradas de texto y tipo.

### Tarea 4.1. Diferenciar MISSING y NULL

- {% include step_label.html %} Ejecuta la consulta en Query Workbench para identificar documentos donde `iata` esté ausente o sea nulo y distinguir ambos estados en SQL++.

  ```sql
  SELECT name, country, iata
  FROM `travel-sample`.inventory.airline
  WHERE iata IS MISSING
     OR iata IS NULL
  ORDER BY name
  LIMIT 10;
  ```

**Resultado esperado:**

Comprueba que cada documento devuelto tenga `iata` ausente o con valor `null`; no deben aparecer códigos IATA definidos en esta salida.

Documentos donde `iata` no existe o tiene valor `null`.

- {% include step_label.html %} Ejecuta el conteo en Query Workbench para medir cuántas aerolíneas contienen un valor IATA presente y no nulo, excluyendo datos incompletos.

  ```sql
  SELECT COUNT(*) AS aerolineas_con_iata
  FROM `travel-sample`.inventory.airline
  WHERE iata IS NOT MISSING
    AND iata IS NOT NULL;
  ```

> **NOTA:** `MISSING` significa que el campo no existe. `NULL` significa que el campo existe, pero no tiene un valor.
{: .lab-note .info .compact}

### Tarea 4.2. Agrupar con GROUP BY

- {% include step_label.html %} Ejecuta la agrupación en Query Workbench para contar aerolíneas por país y ordenar los grupos desde la mayor cantidad hasta la menor.

  ```sql
  SELECT country,
         COUNT(*) AS total_aerolineas
  FROM `travel-sample`.inventory.airline
  WHERE country IS NOT MISSING
  GROUP BY country
  ORDER BY total_aerolineas DESC
  LIMIT 10;
  ```

**Validación:**

Confirma que cada país aparezca una sola vez junto con `total_aerolineas`, y que las cantidades estén ordenadas de mayor a menor.

Debes obtener países agrupados con su cantidad de aerolíneas.

### Tarea 4.3. Filtrar grupos con HAVING

- {% include step_label.html %} Ejecuta la consulta con `HAVING` en Query Workbench para conservar únicamente países con más de cinco aerolíneas después de formar los grupos.

  ```sql
  SELECT country,
         COUNT(*) AS total_aerolineas
  FROM `travel-sample`.inventory.airline
  WHERE country IS NOT MISSING
  GROUP BY country
  HAVING COUNT(*) > 5
  ORDER BY total_aerolineas DESC;
  ```

**Validación:**

Verifica que todos los grupos tengan `total_aerolineas` mayor que cinco, porque `HAVING` se evalúa después de calcular cada agrupación.

Todos los resultados deben tener `total_aerolineas` mayor a 5.

### Tarea 4.4. Aplicar funciones básicas de texto

- {% include step_label.html %} Ejecuta las funciones de texto en Query Workbench para comparar mayúsculas, minúsculas, longitud y los primeros cinco caracteres de cada nombre.

  ```sql
  SELECT name,
         UPPER(name)        AS nombre_mayusculas,
         LOWER(country)     AS pais_minusculas,
         LENGTH(name)       AS longitud_nombre,
         SUBSTR(name, 0, 5) AS primeros_cinco_caracteres
  FROM `travel-sample`.inventory.airline
  ORDER BY name
  LIMIT 10;
  ```

**Validación:**

Compara el nombre original con sus transformaciones y confirma que longitud y subcadena correspondan con el valor de cada fila.

Cada resultado debe mostrar el nombre original y sus transformaciones.

### Tarea 4.5. Aplicar funciones de tipo

- {% include step_label.html %} Ejecuta las conversiones en Query Workbench para inspeccionar los tipos de `id` y `geo`, además de validar la transformación entre número y texto.

  ```sql
  SELECT airportname,
         id,
         TOSTRING(id)              AS id_como_texto,
         TONUMBER(TOSTRING(id))     AS id_reconvertido,
         TYPE(id)                   AS tipo_id,
         TYPE(geo)                  AS tipo_geo
  FROM `travel-sample`.inventory.airport
  ORDER BY META().id
  LIMIT 5;
  ```

**Validación:**

Comprueba que `tipo_id` sea `number`, `tipo_geo` sea `object` y `id_reconvertido` conserve el mismo valor numérico de `id`.

- `tipo_id` debe mostrar `number`.
- `tipo_geo` debe mostrar `object`.
- `id_reconvertido` debe coincidir con `id`.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

{% include support-prompt.html task="tarea4" %}

---

## 🔗 Tarea 5. Consultar arreglos con UNNEST y realizar un JOIN

En esta tarea trabajarás con el arreglo `schedule` de los documentos `route` y relacionarás rutas con aerolíneas mediante `JOIN ... ON KEYS`.

### Tarea 5.1. Explorar un arreglo JSON

- {% include step_label.html %} Ejecuta la consulta en Query Workbench para recuperar una ruta con `schedule` y examinar el arreglo JSON antes de descomponer sus elementos.

  ```sql
  SELECT r.airline,
         r.sourceairport,
         r.destinationairport,
         r.schedule
  FROM `travel-sample`.inventory.route AS r
  WHERE r.schedule IS NOT MISSING
  ORDER BY META(r).id
  LIMIT 1;
  ```

**Resultado esperado:**

Verifica que `schedule` sea un arreglo de objetos con campos como `day`, `flight` y `utc`; esta estructura será la entrada de `UNNEST`.

Debes observar un campo `schedule` que contiene un arreglo de objetos con valores como `day`, `flight` y `utc`.

### Tarea 5.2. Descomponer el arreglo con UNNEST

- {% include step_label.html %} Ejecuta la consulta con `UNNEST` en Query Workbench para convertir cada elemento de `schedule` en una fila asociada con una ruta que sale de SFO.

  ```sql
  SELECT r.airline,
         r.sourceairport,
         r.destinationairport,
         s.day    AS dia,
         s.flight AS vuelo,
         s.utc    AS hora_utc
  FROM `travel-sample`.inventory.route AS r
  UNNEST r.schedule AS s
  WHERE r.sourceairport = "SFO"
  LIMIT 15;
  ```

**Resultado esperado:**

Confirma que cada elemento de `schedule` aparezca como una fila independiente y conserve los datos de la ruta de origen asociada.

Cada elemento del arreglo `schedule` debe aparecer como una fila independiente.

### Tarea 5.3. Contar elementos del arreglo

- {% include step_label.html %} Ejecuta el primer conteo con `UNNEST` en Query Workbench para obtener el total de elementos `schedule` correspondientes a las rutas con origen SFO.

  ```sql
  SELECT COUNT(*) AS total_schedule
  FROM `travel-sample`.inventory.route AS r
  UNNEST r.schedule AS s
  WHERE r.sourceairport = "SFO";
  ```

- {% include step_label.html %} Ejecuta la agrupación por día en Query Workbench para contar los vuelos descompuestos y preparar la comparación con el total anterior.

  ```sql
  SELECT s.day AS dia_semana,
         COUNT(*) AS total_vuelos
  FROM `travel-sample`.inventory.route AS r
  UNNEST r.schedule AS s
  WHERE r.sourceairport = "SFO"
  GROUP BY s.day
  ORDER BY s.day;
  ```

**Validación:**

Suma los valores de `total_vuelos` por día y comprueba que coincidan con `total_schedule`; una diferencia indica filtros distintos.

La suma de todos los valores `total_vuelos` debe coincidir con `total_schedule`.

### Tarea 5.4. Verificar la relación route-airline

- {% include step_label.html %} Ejecuta la consulta sobre `route` en Query Workbench para comprobar que `airlineid` almacena las claves usadas al relacionar rutas y aerolíneas.

  ```sql
  SELECT r.airline,
         r.airlineid,
         r.sourceairport,
         r.destinationairport
  FROM `travel-sample`.inventory.route AS r
  WHERE r.airlineid IS NOT MISSING
  ORDER BY META(r).id
  LIMIT 5;
  ```

**Resultado esperado:**

Verifica que `airlineid` contenga claves con formato `airline_<número>`, porque esos valores enlazan documentos de ambas collections.

El campo `airlineid` debe contener claves como `airline_10`.

### Tarea 5.5. Realizar JOIN entre route y airline

- {% include step_label.html %} Ejecuta el `JOIN ... ON KEYS` en Query Workbench para enlazar cada ruta desde SFO con su aerolínea y proyectar información de ambas collections.

  ```sql
  SELECT r.sourceairport,
         r.destinationairport,
         r.distance,
         a.name     AS nombre_aerolinea,
         a.country  AS pais_aerolinea,
         a.callsign AS indicativo
  FROM `travel-sample`.inventory.route AS r
  JOIN `travel-sample`.inventory.airline AS a
    ON KEYS r.airlineid
  WHERE r.sourceairport = "SFO"
  ORDER BY r.distance DESC
  LIMIT 15;
  ```

**Salida esperada:**

Confirma que las rutas tengan `sourceairport` igual a `SFO` y datos no nulos de aerolínea, demostrando que el `JOIN` encontró coincidencias.

Debes obtener rutas que salen de `SFO` enriquecidas con información de la aerolínea.

**Validación:**

Revisa que `nombre_aerolinea` y `pais_aerolinea` tengan valores, y que cada `sourceairport` sea `SFO` antes de validar mediante REST.

- `nombre_aerolinea` no debe ser `null`.
- `pais_aerolinea` debe contener un país.
- Cada registro debe tener `sourceairport` igual a `SFO`.

### Tarea 5.6. Ejecutar validación final desde Git Bash

- {% include step_label.html %} Ejecuta la solicitud `curl` desde Git Bash para enviar el `JOIN` al servicio Query por el puerto 8093 y comprobar que la respuesta termine en `success`.

  ```bash
  curl -s -u Administrator:Password123! \
    http://localhost:8093/query/service \
    --data-urlencode 'statement=SELECT r.sourceairport, r.destinationairport, a.name AS airline_name FROM `travel-sample`.inventory.route AS r JOIN `travel-sample`.inventory.airline AS a ON KEYS r.airlineid WHERE r.sourceairport = "SFO" LIMIT 3' \
    | python -m json.tool | grep -E '"sourceairport"|"destinationairport"|"airline_name"|"status"'
  ```

**Salida esperada:**

Comprueba que la respuesta incluya tres rutas como máximo y termine con `"status": "success"`; otro estado indica un error del servicio Query.

```text
"sourceairport": "SFO",
"destinationairport": "...",
"airline_name": "...",
"status": "success"
```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

{% include support-prompt.html task="tarea5" %}

---

## 🛠️ Resolución de problemas

### Problema 1. El contenedor couchbase-lab no está activo

**Síntoma:** `docker ps` no muestra el contenedor.

**Solución:**

Ejecuta los comandos en Git Bash para iniciar `couchbase-lab` y confirma después que `docker ps` muestre el contenedor con estado `Up`.

```bash
docker start couchbase-lab
docker ps --filter "name=couchbase-lab"
```

### Problema 2. La consulta devuelve No index available

**Síntoma:** Couchbase indica que no existe un índice disponible.

**Solución:**

Consulta `system:indexes` en Query Workbench para identificar índices ausentes o en un estado distinto de `online` antes de crear otros.

En Query Workbench, consulta los índices registrados para `travel-sample.inventory` y revisa su nombre, collection y estado antes de crear índices adicionales:

```sql
SELECT name, keyspace_id, state
FROM system:indexes
WHERE bucket_id = "travel-sample"
  AND scope_id = "inventory"
ORDER BY keyspace_id, name;
```

Si falta un índice primario para alguna collection utilizada en consultas generales, créalo de forma idempotente:

```sql
CREATE PRIMARY INDEX IF NOT EXISTS idx_airport_primary
ON `travel-sample`.inventory.airport;

CREATE PRIMARY INDEX IF NOT EXISTS idx_route_primary
ON `travel-sample`.inventory.route;
```

### Problema 3. USE KEYS devuelve menos documentos de los esperados

**Causa probable:** Alguna clave no existe en la versión actual del dataset.

**Solución:**

Consulta identificadores existentes en `airline` y sustituye únicamente las claves ausentes del ejemplo, conservando el propósito de `USE KEYS`.

En Query Workbench, consulta claves reales de `airline` para reemplazar identificadores ausentes sin modificar la lógica del ejercicio con `USE KEYS`:

```sql
SELECT META(a).id AS document_id,
       a.name
FROM `travel-sample`.inventory.airline AS a
ORDER BY META(a).id
LIMIT 10;
```

Sustituye únicamente las claves inexistentes por identificadores devueltos y vuelve a ejecutar `USE KEYS` para confirmar que recupera todos los documentos solicitados.

### Problema 4. UNNEST devuelve 0 resultados

**Causa probable:** El filtro no coincide con rutas que tengan el arreglo `schedule`.

**Solución:**

Ejecuta la consulta diagnóstica para localizar aeropuertos con arreglos no vacíos y utiliza uno de sus valores como nuevo filtro de origen.

```sql
SELECT r.sourceairport,
       ARRAY_LENGTH(r.schedule) AS elementos
FROM `travel-sample`.inventory.route AS r
WHERE r.schedule IS NOT MISSING
  AND ARRAY_LENGTH(r.schedule) > 0
ORDER BY elementos DESC
LIMIT 10;
```

Selecciona un `sourceairport` devuelto con `elementos` mayor que cero, reemplaza `SFO` temporalmente y confirma que `UNNEST` produzca filas.

### Problema 5. El contenedor todavía utiliza Community Edition

**Síntoma:** El comando `docker inspect` muestra `couchbase/server:community-7.6.2`.

**Causa probable:** No se realizó el reemplazo del contenedor durante la Práctica 2.

**Solución:**

Regresa a la práctica 2 y recrea el contenedor con la imagen Enterprise indicada; después confirma la etiqueta mediante `docker inspect`.

Regresa a la Práctica 2 y ejecuta los pasos indicados para eliminar el contenedor Community y recrearlo con la imagen:

```text
couchbase/server:enterprise-7.6.2
```

Después de recrear el contenedor, ejecuta la inspección siguiente en Git Bash y confirma la etiqueta Enterprise antes de retomar la práctica:

```bash
docker inspect couchbase-lab   --format '{{.Config.Image}}'
```

### Problema 6. El comando python -m json.tool falla

**Solución:**

Ejecuta ambas variantes en Git Bash para identificar el nombre disponible de Python y utiliza ese mismo ejecutable en las validaciones restantes.

```bash
python --version
python3 --version
```

Si `python3` responde correctamente, sustituye el ejecutable de la canalización original para conservar el formateo JSON de las respuestas REST:

```bash
python -m json.tool
```

Utiliza en su lugar la variante siguiente y confirma que la respuesta JSON se presente con sangría y sin errores de módulo:

```bash
python3 -m json.tool
```
