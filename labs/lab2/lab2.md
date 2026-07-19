---
layout: lab
title: "Práctica 2: Instalación y exploración inicial de Couchbase Server"
permalink: /lab2/lab2/
images_base: /labs/lab2/img
duration: "50 minutos"
objective:
  - Instalar y ejecutar Couchbase Server en un contenedor Docker desde la terminal integrada de VS Code usando Git Bash.
  - Configurar un clúster de nodo único con los servicios básicos para el curso.
  - Cargar el dataset de ejemplo travel-sample y explorar su estructura en buckets, scopes y collections.
  - Crear índices mínimos y ejecutar consultas SQL++ sobre documentos JSON.
  - Validar el estado del entorno mediante comandos curl desde Git Bash en VS Code.
prerequisites:
  - Haber completado la Práctica 1 o conocer los conceptos básicos de NoSQL.
  - Tener Docker Desktop instalado y en ejecución.
  - Tener Git Bash instalado en Windows y configurarlo como terminal integrada en Visual Studio Code.
  - Tener un navegador web moderno como Chrome, Edge o Firefox.
  - Tener disponibles los puertos 8091 a 8097, 11210 y 18091 a 18097.
introduction:
  - En esta práctica instalarás Couchbase Server usando Docker desde la terminal integrada de Visual Studio Code con perfil Git Bash. Crearás un clúster local de nodo único, cargarás el dataset travel-sample, explorarás documentos JSON desde la Web Console y ejecutarás consultas SQL++ básicas. También validarás los servicios principales con curl desde VS Code para confirmar que el entorno queda listo para las siguientes prácticas del curso.
slug: lab2
lab_number: 2
final_result: >
  Al finalizar la práctica tendrás un entorno local de Couchbase Server funcionando en Docker, con el clúster lab-cluster configurado, el bucket travel-sample cargado, índices mínimos creados y consultas SQL++ ejecutadas correctamente desde la Web Console y desde la terminal integrada de VS Code usando Git Bash.
notes:
  - En Windows, ejecuta todos los comandos desde la terminal integrada de Visual Studio Code seleccionando el perfil Git Bash. No uses PowerShell para esta práctica, porque los comandos usan sintaxis Bash, barras invertidas, grep y curl.
  - No elimines el contenedor al finalizar la práctica. Este entorno se usará como base para la práctica 3.
  - La contraseña usada en el laboratorio es solo para entorno local de aprendizaje. No la uses en ambientes productivos.
references: []
prev: /lab1/lab1/
next: /lab3/lab3/
---

---
<!-- Aquí comienzan las instrucciones paso a paso de la práctica -->

## 📁 Preparación del directorio de trabajo

En esta práctica crearás el directorio raíz del curso y, dentro de él, el subdirectorio correspondiente a la práctica 2. En las prácticas siguientes conservarás el directorio raíz y crearás únicamente el subdirectorio de cada nueva práctica.

### 🗂️ Crear el directorio raíz y el subdirectorio de la práctica

- {% include step_label.html %} Abre **Docker Desktop** y espera a que el indicador del motor confirme que está activo, porque todos los contenedores y puertos del laboratorio dependerán de ese servicio.
- {% include step_label.html %} Abre **Visual Studio Code** y espera a que cargue completamente, ya que desde su terminal integrada administrarás los archivos, el contenedor y las validaciones de Couchbase.
- {% include step_label.html %} En VS Code, selecciona **Terminal → New Terminal** para abrir una consola integrada desde la cual ejecutarás los comandos Bash sin cambiar de aplicación ni de contexto.
- {% include step_label.html %} En la flecha desplegable situada junto al botón **+** del panel Terminal, selecciona **Git Bash** para utilizar la sintaxis, rutas y utilidades requeridas por esta práctica.
- {% include step_label.html %} Ejecuta el siguiente comando para crear de forma simultánea el directorio raíz del curso y la carpeta `lab2`, conservando una estructura ordenada para los archivos posteriores.

  ```bash
  mkdir -p /c/LABS/couchbase-nosql/lab2
  ```

- {% include step_label.html %} Ejecuta el comando siguiente para abrir `C:\LABS\couchbase-nosql` como carpeta de trabajo en VS Code y visualizar desde el Explorador todos los subdirectorios del curso.

  ```bash
  code /c/LABS/couchbase-nosql
  ```

- {% include step_label.html %} Si VS Code solicita confirmar la confianza o apertura de la carpeta, acepta la acción para habilitar la terminal, el Explorador y la edición de archivos dentro del directorio del curso.
- {% include step_label.html %} En el Explorador de VS Code, comprueba que aparezcan el directorio raíz `couchbase-nosql` y su subdirectorio `lab2`, porque allí se conservarán los archivos de esta práctica.

  ```text
  C:\LABS\couchbase-nosql
  └── lab2
  ```

- {% include step_label.html %} Ejecuta el siguiente comando para cambiar la ubicación activa de Git Bash al subdirectorio `lab2`, evitando crear archivos o ejecutar operaciones desde una ruta incorrecta.

  ```bash
  cd /c/LABS/couchbase-nosql/lab2
  ```

- {% include step_label.html %} Ejecuta `pwd` para mostrar la ruta actual de la terminal y confirma que Git Bash se encuentra exactamente en `/c/LABS/couchbase-nosql/lab2` antes de continuar.

  ```bash
  pwd
  ```

**Salida esperada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```text
/c/LABS/couchbase-nosql/lab2
```

> **IMPORTANTE:** En las prácticas siguientes no volverás a crear el directorio raíz `C:\LABS\couchbase-nosql`. Únicamente crearás el subdirectorio correspondiente, por ejemplo `lab3`, `lab4` o `lab5`, dentro del mismo directorio raíz.
{: .lab-note .important .compact}

---

## 🐳 Tarea 1. Ejecutar Couchbase Server con Docker y validar Web Console

En esta tarea iniciarás Couchbase Server dentro de un contenedor Docker usando la terminal integrada de VS Code con Git Bash. El objetivo es que puedas acceder a la Web Console desde tu navegador en `http://localhost:8091`.

### Tarea 1.1. Confirmar el entorno de trabajo

Como ya abriste **Visual Studio Code**, seleccionaste **Git Bash** y te ubicaste en el subdirectorio `lab2` durante la preparación del directorio, únicamente confirmarás que el entorno sigue listo antes de crear el contenedor.

- {% include step_label.html %} Verifica que **Docker Desktop** continúe activo y sin alertas, porque la creación y administración del contenedor fallarán si el motor se detuvo después de preparar el directorio.
- {% include step_label.html %} Confirma que la terminal integrada activa sea **Git Bash** y que su indicador no muestre PowerShell o Command Prompt, ya que los comandos siguientes utilizan sintaxis Bash.
- {% include step_label.html %} Ejecuta `pwd` para comprobar nuevamente la ubicación de trabajo y evita continuar si la salida no corresponde a `/c/LABS/couchbase-nosql/lab2`.

  ```bash
  pwd
  ```

**Salida esperada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```text
/c/LABS/couchbase-nosql/lab2
```

> **IMPORTANTE:** No uses PowerShell ni Command Prompt para esta práctica. Todos los comandos están escritos para ejecutarse desde **Git Bash dentro de la terminal integrada de VS Code**.
{: .lab-note .important .compact}

### Tarea 1.2. Verificar que Docker Desktop esté activo desde VS Code

Ahora validarás que Docker responde correctamente desde la terminal integrada de VS Code.

- {% include step_label.html %} Ejecuta `docker --version` para confirmar que el cliente de Docker está instalado, accesible desde Git Bash y disponible para enviar instrucciones al motor local.
  ```bash
  docker --version
  ```

**Salida esperada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

  ```text
  Docker version 24.x.x, build xxxxxxx
  ```

- {% include step_label.html %} Ejecuta `docker info | grep "Server Version"` para verificar que el motor de Docker responde, no solo el cliente, y que puede crear contenedores en este equipo.
  ```bash
  docker info | grep "Server Version"
  ```

**Salida esperada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

  ```text
  Server Version: 24.x.x
  ```

> **IMPORTANTE:** Si `docker info` muestra un error, revisa que Docker Desktop esté abierto y que no esté detenido. No continúes hasta que Docker responda correctamente.
{: .lab-note .important .compact}

### Tarea 1.3. Crear el contenedor de Couchbase Server

Ahora crearás el contenedor local del laboratorio. Usarás la imagen `couchbase/server:enterprise-7.6.2` para disponer de los servicios de Couchbase Server Enterprise necesarios durante el curso.

- {% include step_label.html %} Ejecuta el comando siguiente para localizar cualquier contenedor llamado `couchbase-lab` y revisar su estado e imagen antes de decidir si debe iniciarse o reemplazarse.

  ```bash
  docker ps -a --filter "name=^/couchbase-lab$"
  ```

- {% include step_label.html %} Si el contenedor encontrado utiliza una imagen Community, ejecuta el comando indicado para eliminarlo y evitar reutilizar datos o servicios incompatibles con Enterprise 7.6.2.

  ```bash
  docker rm -f couchbase-lab
  ```

> **IMPORTANTE:** Cambiar únicamente la imagen no convierte un contenedor Community existente en Enterprise. Es necesario eliminarlo y recrearlo. Esta acción elimina la configuración y los datos almacenados dentro del contenedor anterior.
{: .lab-note .important .compact}

- {% include step_label.html %} Ejecuta `docker pull` para descargar explícitamente la imagen `couchbase/server:enterprise-7.6.2` y asegurar que el laboratorio utilice la edición y versión requeridas.

  ```bash
  docker pull couchbase/server:enterprise-7.6.2
  ```

- {% include step_label.html %} Ejecuta el bloque siguiente para crear en segundo plano el contenedor `couchbase-lab`, publicar los puertos de administración y servicios, y arrancar Couchbase Enterprise 7.6.2.

  ```bash
  docker run -d \
    --name couchbase-lab \
    -p 8091-8097:8091-8097 \
    -p 11210:11210 \
    -p 18091-18097:18091-18097 \
    couchbase/server:enterprise-7.6.2
  ```

**Salida esperada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

El valor mostrado será un identificador largo generado por Docker. No necesitas copiarlo.

  ```text
  <identificador_largo_del_contenedor>
  ```

> **NOTA:** Si el contenedor existente ya utiliza `couchbase/server:enterprise-7.6.2`, puedes ejecutar `docker start couchbase-lab` y continuar con la validación.
{: .lab-note .info .compact}

### Tarea 1.4. Validar que el contenedor está en ejecución

- {% include step_label.html %} Ejecuta el comando siguiente para confirmar que `couchbase-lab` aparece con estado `Up` y que Docker publicó correctamente los rangos de puertos definidos para Couchbase.

 {%raw%}
Ejecuta este bloque completo en la terminal o editor indicado, respeta el orden de las líneas y espera la respuesta antes de continuar.

  ```bash
  docker ps --filter "name=couchbase-lab" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
  ```
  {%endraw%}

**Salida esperada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```text
NAMES           STATUS          PORTS
couchbase-lab   Up ...          0.0.0.0:8091-8097->8091-8097/tcp, ...
```

- {% include step_label.html %} Espera entre 30 y 60 segundos para que los procesos internos de Couchbase inicialicen la Web Console y los servicios; el estado `Up` de Docker no garantiza que ya respondan.
- {% include step_label.html %} Ejecuta la solicitud HTTP siguiente para comprobar que la Web Console responde por el puerto 8091; un código 200 confirma que el servidor web terminó de inicializar.
  ```bash
  curl -s http://localhost:8091/ui/index.html -o /dev/null -w "HTTP Status: %{http_code}\n"
  ```

**Salida esperada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```text
HTTP Status: 200
```

### Tarea 1.5. Abrir la Web Console

- {% include step_label.html %} Abre un navegador compatible para acceder a la consola administrativa, desde la cual crearás el clúster, asignarás servicios y cargarás el dataset de ejemplo.
- {% include step_label.html %} Escribe `http://localhost:8091` en la barra de direcciones para conectar el navegador con la Web Console publicada por el contenedor en el puerto administrativo 8091.

```text
http://localhost:8091
```

- {% include step_label.html %} Confirma que aparezca la pantalla inicial de Couchbase Server con la opción **Setup New Cluster**, lo que demuestra que el nodo está disponible y aún no ha sido configurado.

> **IMPORTANTE:** Si el navegador no carga, espera 1 minuto más y vuelve a intentar. Couchbase puede tardar un poco en inicializar después de crear el contenedor.
{: .lab-note .important .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

{% include support-prompt.html task="tarea1" %}

---

## ⚙️ Tarea 2. Configurar clúster de nodo único y cargar travel-sample

En esta tarea configurarás Couchbase como un clúster local de un solo nodo. También cargarás el dataset `travel-sample`, que usarás en esta y en las siguientes prácticas.

### Tarea 2.1. Iniciar configuración del clúster

- {% include step_label.html %} Selecciona **Setup New Cluster** para iniciar la creación de un clúster independiente de un solo nodo y establecer sus credenciales administrativas iniciales.
- {% include step_label.html %} Completa los campos mostrados con el nombre `lab-cluster` y las credenciales indicadas, verificando cada valor para evitar errores posteriores de autenticación en `curl`.

| Campo | Valor |
|---|---|
| Cluster Name | `lab-cluster` |
| Admin Username | `Administrator` |
| Password | `Password123!` |
| Confirm Password | `Password123!` |

- {% include step_label.html %} Selecciona **Next: Accept Terms** para conservar los datos introducidos y avanzar a la pantalla donde Couchbase solicita aceptar las condiciones de uso de Enterprise.
- {% include step_label.html %} Acepta los términos de uso requeridos para habilitar la configuración del nodo y continuar hacia la asignación manual de servicios, memoria y almacenamiento.

> **IMPORTANTE:** No uses **Finish With Defaults**. En esta práctica necesitas configurar manualmente los servicios y la memoria para evitar ambigüedades.
{: .lab-note .important .compact}

### Tarea 2.2. Configurar servicios y memoria

- {% include step_label.html %} Selecciona **Configure Disk, Memory, Services** para evitar los valores automáticos y controlar qué servicios de Couchbase Enterprise se ejecutarán en el nodo local.
- {% include step_label.html %} Activa Data, Index, Query, Search, Analytics y Eventing para que el mismo nodo proporcione todos los servicios utilizados por esta práctica y laboratorios posteriores.

| Servicio | Estado |
|---|---|
| Data | Activado |
| Index | Activado |
| Query | Activado |
| Search | Activado |
| Analytics | Activado |
| Eventing | Activado |

- {% include step_label.html %} Asigna las cuotas indicadas a cada servicio para reservar memoria suficiente en un entorno local, manteniendo un equilibrio entre funcionalidad y consumo de Docker Desktop.

| Servicio | Memoria sugerida |
|---|---:|
| Data | 1024 MB |
| Index | 512 MB |
| Search | 512 MB |
| Analytics | 1024 MB |
| Eventing | 256 MB |

- {% include step_label.html %} Conserva las rutas de datos e índices propuestas por Couchbase, porque dentro del contenedor ya apuntan a ubicaciones válidas y no requieren volúmenes externos en esta práctica.
- {% include step_label.html %} Selecciona **Save & Finish** para aplicar las cuotas, registrar las credenciales, iniciar los servicios elegidos y abrir el Dashboard del clúster recién creado.

**Resultado visual esperado:**

Confirma que la interfaz muestre los elementos descritos, sin alertas activas ni estados pendientes, antes de continuar con la siguiente acción.

Debes llegar al **Dashboard** principal del clúster `lab-cluster`. El nodo debe aparecer activo y saludable.

### Tarea 2.3. Validar el clúster desde Git Bash en VS Code

- {% include step_label.html %} Regresa a Git Bash en VS Code para validar por REST la configuración guardada, utilizando las mismas credenciales administrativas definidas en la Web Console.
- {% include step_label.html %} Ejecuta la solicitud siguiente al endpoint `/pools/default` para comprobar que el clúster se llama `lab-cluster` y que la cuota principal de Data quedó registrada.
  ```bash
  curl -s -u Administrator:Password123! \
    http://localhost:8091/pools/default \
    | python -m json.tool | grep -E '"clusterName"|"memoryQuota"'
  ```

**Salida esperada aproximada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```text
"clusterName": "lab-cluster",
"memoryQuota": 1024,
```

> **NOTA:** Si tu equipo usa `python3` en lugar de `python`, reemplaza `python -m json.tool` por `python3 -m json.tool` en los comandos de validación.
{: .lab-note .info .compact}

### Tarea 2.4. Cargar el dataset travel-sample

- {% include step_label.html %} Abre el menú lateral de la Web Console para acceder a las áreas de administración y localizar la sección desde la cual se instalan datasets de ejemplo.
- {% include step_label.html %} Selecciona **Settings** en el menú lateral para abrir la configuración general del clúster, donde Couchbase Enterprise agrupa la instalación de Sample Buckets.
- {% include step_label.html %} Abre **Sample Buckets** para consultar los datasets disponibles y verificar que `travel-sample` todavía no esté instalado en el clúster local.
- {% include step_label.html %} Marca únicamente **travel-sample** para seleccionar el dataset de viajes que contiene los scopes y collections utilizados en las consultas de esta práctica.
- {% include step_label.html %} Selecciona **Load Sample Data** para crear el bucket, importar sus documentos y preparar automáticamente la estructura de scopes y collections incluida en el ejemplo.
- {% include step_label.html %} Espera hasta que la interfaz confirme que la carga terminó; no cambies de sección mientras Couchbase crea el bucket y distribuye más de treinta mil documentos.

### Tarea 2.5. Validar la carga de travel-sample

- {% include step_label.html %} Ejecuta la solicitud REST siguiente para leer las estadísticas del bucket `travel-sample` y confirmar que existe, tiene documentos y terminó su carga correctamente.

  ```bash
  curl -s -u Administrator:Password123! \
    http://localhost:8091/pools/default/buckets/travel-sample \
    | python -m json.tool | grep -E '"name"|"itemCount"'
  ```

**Salida esperada aproximada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```text
"name": "travel-sample",
"itemCount": <valor_mayor_a_30000>,
```

> **IMPORTANTE:** El número exacto de documentos puede variar ligeramente según la versión del dataset. Lo importante es que `itemCount` sea mayor a 30000.
{: .lab-note .important .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

{% include support-prompt.html task="tarea2" %}

---

## 🧭 Tarea 3. Explorar bucket, scope, collections y documentos JSON

En esta tarea revisarás cómo se organiza la información dentro de Couchbase. Identificarás el bucket `travel-sample`, el scope `inventory`, sus collections y algunos documentos JSON reales.

### Tarea 3.1. Explorar el bucket travel-sample

- {% include step_label.html %} Selecciona **Buckets** en la navegación lateral para abrir el inventario de buckets y comenzar la exploración jerárquica del dataset recién instalado.
- {% include step_label.html %} Localiza `travel-sample` y verifica que su estado sea saludable, porque las siguientes acciones dependen de que el bucket esté disponible y sin operaciones pendientes.
- {% include step_label.html %} Abre el nombre del bucket o **Scopes & Collections** para visualizar la jerarquía interna y diferenciar el contenedor lógico de sus scopes y collections.
- {% include step_label.html %} Identifica el scope `inventory` y sus collections principales para reconocer cómo Couchbase organiza documentos relacionados sin utilizar tablas relacionales tradicionales.

```text
travel-sample
└── inventory
    ├── airline
    ├── airport
    ├── hotel
    ├── landmark
    └── route
```

**Resultado esperado:**

Confirma que la interfaz muestre los elementos descritos, sin alertas activas ni estados pendientes, antes de continuar con la siguiente acción.

Debes observar un bucket llamado `travel-sample`, un scope llamado `inventory` y varias collections relacionadas con viajes.

### Tarea 3.2. Explorar la collection airline

- {% include step_label.html %} Abre la collection **airline** dentro del scope `inventory` para limitar la exploración a documentos de aerolíneas y evitar mezclar tipos de entidades.
- {% include step_label.html %} Selecciona la vista de documentos de `airline` para consultar identificadores, contenido JSON y metadatos almacenados específicamente en esa collection.
- {% include step_label.html %} Busca el identificador `airline_10` y abre el resultado exacto para revisar un documento conocido que también será validado posteriormente mediante SQL++.
- {% include step_label.html %} Examina el JSON mostrado y reconoce sus pares clave-valor simples, observando que todos los atributos de la aerolínea se encuentran en el primer nivel del documento.
  ```json
  {
    "id": 10,
    "type": "airline",
    "name": "40-Mile Air",
    "iata": "Q5",
    "icao": "MLA",
    "callsign": "MILE-AIR",
    "country": "United States"
  }
  ```

- {% include step_label.html %} Registra tres campos, sus valores y tipos aparentes para relacionar la representación JSON observada con las columnas proyectadas después en las consultas SQL++.

### Tarea 3.3. Explorar la collection hotel

- {% include step_label.html %} Regresa al scope `inventory` mediante la navegación de Scopes & Collections para seleccionar otra collection sin salir del bucket `travel-sample`.
- {% include step_label.html %} Abre la collection **hotel** para explorar documentos con una estructura más flexible y compararlos con los registros sencillos observados en `airline`.
- {% include step_label.html %} Selecciona cualquier documento disponible de `hotel` y ábrelo en la vista JSON, comprobando que pertenece realmente a `travel-sample.inventory.hotel`.
- {% include step_label.html %} Examina el documento y localiza objetos como `geo` o arreglos como `reviews`, porque estos elementos muestran cómo Couchbase representa información anidada.
  ```json
  {
    "type": "hotel",
    "name": "Medway Youth Hostel",
    "city": "Medway",
    "country": "United Kingdom",
    "geo": {
      "lat": 51.35785,
      "lon": 0.55818
    },
    "reviews": []
  }
  ```

- {% include step_label.html %} Compara los niveles, campos y tipos de ambos documentos para distinguir una entidad plana de otra que incorpora objetos y arreglos dentro del mismo JSON.
- {% include step_label.html %} Responde las preguntas propuestas utilizando lo observado en ambos documentos, relacionando la flexibilidad del esquema con necesidades distintas de cada entidad.

```text
¿Qué documento parece más simple?
¿Qué documento utiliza campos anidados?
¿Por qué esta flexibilidad puede ser útil en una base documental?
```

### Tarea 3.4. Validar un documento usando SQL++ con META().id

En lugar de usar un endpoint REST directo al documento, validarás el documento `airline_10` usando SQL++. Esto evita problemas con scopes y collections.

- {% include step_label.html %} Ejecuta la solicitud SQL++ siguiente desde Git Bash para recuperar `airline_10` por `META().id` y validar el documento dentro de su bucket, scope y collection exactos.

  ```bash
  curl -s -u Administrator:Password123! \
    http://localhost:8093/query/service \
    -d 'statement=SELECT META(a).id AS document_id, a.name, a.iata, a.country FROM `travel-sample`.inventory.airline AS a WHERE META(a).id = "airline_10"' \
    | python -m json.tool | grep -E '"document_id"|"name"|"iata"|"country"|"status"'
  ```

**Salida esperada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```text
"document_id": "airline_10",
"name": "40-Mile Air",
"iata": "Q5",
"country": "United States",
"status": "success"
```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

{% include support-prompt.html task="tarea3" %}

---

## 🔎 Tarea 4. Crear índices mínimos y ejecutar consultas SQL++

En esta tarea prepararás índices mínimos para evitar errores de consulta y después ejecutarás consultas SQL++ básicas sobre el dataset `travel-sample`.

### Tarea 4.1. Abrir el Query Editor

- {% include step_label.html %} Selecciona **Query** en la navegación lateral para abrir el Query Workbench, herramienta de Couchbase destinada a escribir y ejecutar sentencias SQL++.
- {% include step_label.html %} Confirma que el área central muestre el editor SQL++ y un panel de resultados, porque allí crearás índices y revisarás el estado de cada sentencia ejecutada.
- {% include step_label.html %} Elimina cualquier texto anterior del editor para evitar que una sentencia residual se ejecute junto con los índices o consultas definidos en los pasos siguientes.

### Tarea 4.2. Crear índices mínimos

Los índices permiten que el servicio Query pueda consultar las collections de forma correcta y eficiente. Para esta práctica crearás índices simples sobre `airline` y `hotel`.

- {% include step_label.html %} Copia y ejecuta el bloque completo para crear índices primarios y secundarios sobre `airline` y `hotel`, habilitando búsquedas generales y filtros por país.

  ```sql
  CREATE PRIMARY INDEX IF NOT EXISTS idx_airline_primary
  ON `travel-sample`.inventory.airline;

  CREATE PRIMARY INDEX IF NOT EXISTS idx_hotel_primary
  ON `travel-sample`.inventory.hotel;

  CREATE INDEX IF NOT EXISTS idx_airline_country
  ON `travel-sample`.inventory.airline(country);

  CREATE INDEX IF NOT EXISTS idx_hotel_country
  ON `travel-sample`.inventory.hotel(country);
  ```

**Salida esperada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

Couchbase debe mostrar que las sentencias se ejecutaron correctamente. Si un índice ya existe, el uso de `IF NOT EXISTS` evitará que la práctica falle.

> **NOTA:** Crear índices en este punto hace que las consultas siguientes sean más estables. En laboratorios posteriores analizarás con más detalle cómo diseñar índices correctamente.
{: .lab-note .info .compact}

### Tarea 4.3. Consultar aerolíneas por país

- {% include step_label.html %} Ejecuta la consulta para filtrar aerolíneas de Estados Unidos, ordenar sus nombres y limitar la salida, verificando una proyección controlada de tres campos.

  ```sql
  SELECT name, iata, country
  FROM `travel-sample`.inventory.airline
  WHERE country = "United States"
  ORDER BY name
  LIMIT 10;
  ```

**Salida esperada aproximada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```json
[
  {
    "country": "United States",
    "iata": "Q5",
    "name": "40-Mile Air"
  }
]
```

El resultado puede incluir más aerolíneas. Lo importante es que aparezcan documentos con `country`, `iata` y `name`.

### Tarea 4.4. Contar aerolíneas por país

- {% include step_label.html %} Ejecuta la agregación para contar aerolíneas por país y ordenar los grupos de mayor a menor, comprobando el uso conjunto de `GROUP BY`, `COUNT` y `ORDER BY`.

  ```sql
  SELECT country, COUNT(*) AS total_airlines
  FROM `travel-sample`.inventory.airline
  GROUP BY country
  ORDER BY total_airlines DESC
  LIMIT 5;
  ```

**Salida esperada aproximada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```json
[
  {
    "country": "United States",
    "total_airlines": 100
  },
  {
    "country": "United Kingdom",
    "total_airlines": 20
  }
]
```

> **IMPORTANTE:** Los valores exactos pueden cambiar según la versión del dataset. Valida la estructura del resultado, no memorices los números.
{: .lab-note .important .compact}

### Tarea 4.5. Consultar campos anidados en documentos hotel

- {% include step_label.html %} Ejecuta la consulta sobre `hotel` para proyectar `geo.lat` y `geo.lon`, demostrando cómo SQL++ accede mediante notación de punto a campos JSON anidados.

  ```sql
  SELECT name, city, geo.lat AS latitude, geo.lon AS longitude
  FROM `travel-sample`.inventory.hotel
  WHERE country = "United Kingdom"
    AND geo IS NOT NULL
  LIMIT 5;
  ```

**Salida esperada aproximada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```json
[
  {
    "city": "Medway",
    "latitude": 51.35785,
    "longitude": 0.55818,
    "name": "Medway Youth Hostel"
  }
]
```

### Tarea 4.6. Ejecutar una consulta SQL++ desde Git Bash en VS Code

Ahora ejecutarás una consulta usando el endpoint REST del servicio Query.

- {% include step_label.html %} Ejecuta la solicitud REST al servicio Query para enviar una sentencia SQL++ desde Git Bash y confirmar que el endpoint 8093 devuelve resultados y estado `success`.

  ```bash
  curl -s -u Administrator:Password123! \
    http://localhost:8093/query/service \
    -d 'statement=SELECT name, iata FROM `travel-sample`.inventory.airline WHERE country="United States" LIMIT 3' \
    | python -m json.tool | grep -E '"name"|"iata"|"status"'
  ```

**Salida esperada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```text
"name": "40-Mile Air",
"iata": "Q5",
"status": "success"
```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

{% include support-prompt.html task="tarea4" %}

---

## ✅ Tarea 5. Validar servicios con REST API desde Git Bash en VS Code

En esta tarea usarás `curl` para comprobar que los servicios principales de Couchbase están activos. Esta validación te ayuda a confirmar que el entorno quedó listo para continuar con la práctica 3.

### Tarea 5.1. Validar Web Console

- {% include step_label.html %} Ejecuta la solicitud siguiente contra el puerto 8091 para verificar que la página de la Web Console está disponible y devuelve el código HTTP 200.

  ```bash
  curl -s -o /dev/null -w "Web Console (8091): HTTP %{http_code}\n" \
    http://localhost:8091/ui/index.html
  ```

**Salida esperada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```text
Web Console (8091): HTTP 200
```

### Tarea 5.2. Validar autenticación contra el clúster

- {% include step_label.html %} Ejecuta la solicitud autenticada a `/pools` para comprobar que las credenciales administrativas son válidas y que la API REST del clúster acepta el acceso.

  ```bash
  curl -s -u Administrator:Password123! \
    -o /dev/null -w "Cluster REST API: HTTP %{http_code}\n" \
    http://localhost:8091/pools
  ```

**Salida esperada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```text
Cluster REST API: HTTP 200
```

### Tarea 5.3. Validar el servicio Query

- {% include step_label.html %} Ejecuta la sentencia `SELECT 1` mediante el puerto 8093 para validar el servicio Query sin depender del contenido de un bucket o de la existencia de índices.

  ```bash
  curl -s -u Administrator:Password123! \
    http://localhost:8093/query/service \
    -d 'statement=SELECT 1 AS query_service_ok' \
    | python -m json.tool | grep -E '"query_service_ok"|"status"'
  ```

**Salida esperada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```text
"query_service_ok": 1,
"status": "success"
```

### Tarea 5.4. Validar el servicio Search

- {% include step_label.html %} Ejecuta la solicitud al endpoint de índices del puerto 8094 para confirmar que el servicio Search está accesible y responde a llamadas REST autenticadas.

  ```bash
  curl -s -u Administrator:Password123! \
    -o /dev/null -w "Search Service (8094): HTTP %{http_code}\n" \
    http://localhost:8094/api/index
  ```

**Salida esperada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```text
Search Service (8094): HTTP 200
```

### Tarea 5.5. Validar el servicio Analytics

- {% include step_label.html %} Ejecuta la consulta constante mediante el puerto 8095 para comprobar que Analytics procesa sentencias y devuelve un estado `success` en el nodo configurado.

  ```bash
  curl -s -u Administrator:Password123! \
    http://localhost:8095/analytics/service \
    -d 'statement=SELECT 1 AS analytics_service_ok;' \
    | python -m json.tool | grep -E '"analytics_service_ok"|"status"'
  ```

**Salida esperada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```text
"analytics_service_ok": 1,
"status": "success"
```

### Tarea 5.6. Validar el servicio Eventing

- {% include step_label.html %} Ejecuta la solicitud al puerto 8096 para confirmar que el servicio Eventing está activo y que su endpoint de estado responde con código HTTP 200.

  ```bash
  curl -s -u Administrator:Password123! \
    -o /dev/null -w "Eventing Service (8096): HTTP %{http_code}\n" \
    http://localhost:8096/api/v1/status
  ```

**Salida esperada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```text
Eventing Service (8096): HTTP 200
```

### Tarea 5.7. Validar el bucket travel-sample

- {% include step_label.html %} Ejecuta la solicitud REST del bucket para verificar nuevamente su nombre, tipo `membase` y cantidad de documentos antes de cerrar la práctica.

  ```bash
  curl -s -u Administrator:Password123! \
    http://localhost:8091/pools/default/buckets/travel-sample \
    | python -m json.tool | grep -E '"name"|"itemCount"|"bucketType"'
  ```

**Salida esperada aproximada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```text
"name": "travel-sample",
"bucketType": "membase",
"itemCount": <valor_mayor_a_30000>,
```

### Tarea 5.8. Ejecutar verificación final

- {% include step_label.html %} Copia y ejecuta el bloque completo para reunir en una sola salida las validaciones de Web Console, autenticación, dataset, Query, Search, Analytics y Eventing.

  ```bash
  echo "=== Verificación final de la Práctica 2 ==="

  STATUS_WEB=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8091/ui/index.html)
  if [ "$STATUS_WEB" = "200" ]; then
    echo "✅ Web Console accesible"
  else
    echo "❌ Web Console no accesible"
  fi

  STATUS_AUTH=$(curl -s -u Administrator:Password123! -o /dev/null -w "%{http_code}" http://localhost:8091/pools)
  if [ "$STATUS_AUTH" = "200" ]; then
    echo "✅ Autenticación correcta"
  else
    echo "❌ Error de autenticación"
  fi

  ITEMS=$(curl -s -u Administrator:Password123! \
    http://localhost:8091/pools/default/buckets/travel-sample \
    | python -c "import sys,json; d=json.load(sys.stdin); print(d.get('basicStats',{}).get('itemCount',0))" 2>/dev/null)

  if [ "$ITEMS" -gt "30000" ] 2>/dev/null; then
    echo "✅ travel-sample cargado con $ITEMS documentos"
  else
    echo "❌ travel-sample no está cargado correctamente"
  fi

  QUERY_STATUS=$(curl -s -u Administrator:Password123! \
    http://localhost:8093/query/service \
    -d 'statement=SELECT 1 AS ok' \
    | python -c "import sys,json; d=json.load(sys.stdin); print(d.get('status','error'))" 2>/dev/null)

  if [ "$QUERY_STATUS" = "success" ]; then
    echo "✅ Servicio Query operativo"
  else
    echo "❌ Servicio Query no operativo"
  fi

  SEARCH_STATUS=$(curl -s -u Administrator:Password123! -o /dev/null -w "%{http_code}" http://localhost:8094/api/index)
  if [ "$SEARCH_STATUS" = "200" ]; then
    echo "✅ Servicio Search accesible"
  else
    echo "❌ Servicio Search no accesible"
  fi

  ANALYTICS_STATUS=$(curl -s -u Administrator:Password123! \
    http://localhost:8095/analytics/service \
    -d 'statement=SELECT 1 AS ok;' \
    | python -c "import sys,json; d=json.load(sys.stdin); print(d.get('status','error'))" 2>/dev/null)

  if [ "$ANALYTICS_STATUS" = "success" ]; then
    echo "✅ Servicio Analytics operativo"
  else
    echo "❌ Servicio Analytics no operativo"
  fi

  EVENTING_STATUS=$(curl -s -u Administrator:Password123! -o /dev/null -w "%{http_code}" http://localhost:8096/api/v1/status)
  if [ "$EVENTING_STATUS" = "200" ]; then
    echo "✅ Servicio Eventing accesible"
  else
    echo "❌ Servicio Eventing no accesible"
  fi

  echo "=== Fin de verificación ==="
  ```

**Salida esperada:**

Comprueba la salida obtenida con la referencia siguiente; valida los campos, el estado y los valores relevantes antes de continuar con la subtarea.

```text
=== Verificación final de la Práctica 2 ===
✅ Web Console accesible
✅ Autenticación correcta
✅ travel-sample cargado con <cantidad_detectada> documentos
✅ Servicio Query operativo
✅ Servicio Search accesible
✅ Servicio Analytics operativo
✅ Servicio Eventing accesible
=== Fin de verificación ===
```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

{% include support-prompt.html task="tarea5" %}

---

## 🛠️ Resolución de problemas

### Problema 1. Docker no responde desde Git Bash en VS Code

**Síntoma:** Al ejecutar `docker info`, aparece un error de conexión.

**Causa probable:** Docker Desktop no está abierto o el motor de Docker todavía no termina de iniciar.

**Solución:**

Aplica las acciones siguientes en el orden presentado, revisa cada respuesta y detente si aparece un error diferente del síntoma documentado.

```bash
docker --version
docker info
```

Si `docker info` falla, abre Docker Desktop, espera a que indique que el motor está activo y vuelve a ejecutar los comandos.

### Problema 2. El puerto 8091 no responde

**Síntoma:** El navegador no abre `http://localhost:8091` o `curl` no devuelve HTTP 200.

**Causa probable:** Couchbase aún está inicializando, el contenedor está detenido o existe conflicto de puertos.

**Solución:**

Aplica las acciones siguientes en el orden presentado, revisa cada respuesta y detente si aparece un error diferente del síntoma documentado.

```bash
docker ps -a --filter "name=couchbase-lab"
docker logs couchbase-lab --tail 30
```

Si el contenedor está detenido, ejecútalo de nuevo:

Ejecuta este bloque completo en la terminal o editor indicado, respeta el orden de las líneas y espera la respuesta antes de continuar.

```bash
docker start couchbase-lab
```

Espera 60 segundos y vuelve a validar:

Ejecuta este bloque completo en la terminal o editor indicado, respeta el orden de las líneas y espera la respuesta antes de continuar.

```bash
curl -s http://localhost:8091/ui/index.html -o /dev/null -w "HTTP Status: %{http_code}\n"
```

### Problema 3. Las consultas SQL++ fallan con error de índice

**Síntoma:** El Query Editor muestra un error similar a `No index available`.

**Causa probable:** Los índices mínimos no fueron creados o todavía se están construyendo.

**Solución:**

Aplica las acciones siguientes en el orden presentado, revisa cada respuesta y detente si aparece un error diferente del síntoma documentado.

Ejecuta nuevamente en el Query Editor:

Ejecuta este bloque completo en la terminal o editor indicado, respeta el orden de las líneas y espera la respuesta antes de continuar.

```sql
CREATE PRIMARY INDEX IF NOT EXISTS idx_airline_primary
ON `travel-sample`.inventory.airline;

CREATE PRIMARY INDEX IF NOT EXISTS idx_hotel_primary
ON `travel-sample`.inventory.hotel;

CREATE INDEX IF NOT EXISTS idx_airline_country
ON `travel-sample`.inventory.airline(country);

CREATE INDEX IF NOT EXISTS idx_hotel_country
ON `travel-sample`.inventory.hotel(country);
```

Después espera unos segundos y vuelve a ejecutar la consulta.

### Problema 4. Git Bash en VS Code muestra error con python

**Síntoma:** El comando `python -m json.tool` no funciona.

**Causa probable:** Python no está agregado al `PATH` o tu instalación usa el comando `python3`.

**Solución:**

Aplica las acciones siguientes en el orden presentado, revisa cada respuesta y detente si aparece un error diferente del síntoma documentado.

Prueba:

Ejecuta este bloque completo en la terminal o editor indicado, respeta el orden de las líneas y espera la respuesta antes de continuar.

```bash
python --version
python3 --version
```

Si `python3` funciona, reemplaza en los comandos:

Ejecuta este bloque completo en la terminal o editor indicado, respeta el orden de las líneas y espera la respuesta antes de continuar.

```bash
python -m json.tool
```

por:

Ejecuta este bloque completo en la terminal o editor indicado, respeta el orden de las líneas y espera la respuesta antes de continuar.

```bash
python3 -m json.tool
```