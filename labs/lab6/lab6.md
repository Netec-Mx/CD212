---
layout: lab
title: "Práctica 6: Creación y despliegue de una Function de Eventing"
permalink: /lab6/lab6/
images_base: /labs/lab6/img
duration: "60 minutos"
objective:
  - Preparar el directorio de trabajo de la práctica 6 y validar que Couchbase Server Enterprise Edition y el servicio Eventing estén disponibles.
  - Crear el bucket app-data, el scope workshop, las collections de trabajo y los índices necesarios para las validaciones SQL++.
  - Crear y configurar una Function de Eventing con source collection, metadata collection y bindings de destino.
  - Implementar los handlers OnUpdate y OnDelete para auditar mutaciones y validar pedidos.
  - Desplegar la Function, generar eventos de prueba, revisar los documentos creados y analizar los logs.
  - Revisar las métricas disponibles y completar el ciclo operativo mediante undeploy.
prerequisites:
  - Haber completado la Práctica 5.
  - Tener Docker Desktop en ejecución.
  - Tener activo el contenedor couchbase-lab creado con la imagen couchbase/server:enterprise-7.6.2.
  - Tener acceso a la Web Console en http://localhost:8091.
  - Tener habilitados los servicios Data, Query, Index y Eventing en Couchbase Server Enterprise Edition.
  - Utilizar Visual Studio Code con Git Bash como terminal integrada.
  - Conocer la estructura básica de un documento JSON y la sintaxis fundamental de SQL++.
introduction:
  - En esta práctica utilizarás Couchbase Server Enterprise Edition para crear una Function de Couchbase Eventing llamada ProcessOrderEvents. La Function escuchará mutaciones sobre la collection orders y reaccionará de forma automática. Cuando un pedido sea válido, registrará un evento de auditoría; cuando tenga un status no permitido, copiará el documento a una collection de cuarentena para revisión. También registrarás eliminaciones mediante OnDelete, revisarás los logs de ejecución y completarás el ciclo deploy, prueba, observación y undeploy desde la Web Console.
slug: lab6
lab_number: 6
final_result: >
  Al finalizar la práctica habrás creado una Function de Eventing funcional que procesa cambios sobre documentos de pedidos. La Function registrará auditorías, identificará estados inválidos, copiará documentos a cuarentena, reaccionará ante eliminaciones y generará logs verificables. También habrás validado el procesamiento mediante SQL++, revisado las métricas disponibles y realizado el undeploy controlado de la Function.
notes:
  - Todos los comandos de terminal deben ejecutarse desde Git Bash dentro de Visual Studio Code.
  - Utiliza las credenciales Administrator y Password123! configuradas en las prácticas anteriores.
  - La Function de esta práctica escribe únicamente en collections diferentes a la fuente para evitar ciclos de recursión.
  - No elimines los buckets ni las collections al finalizar. Solo realiza undeploy de la Function.
  - Los nombres exactos de algunas opciones de Eventing pueden variar ligeramente según la versión de Couchbase Server Enterprise Edition.
references: []
prev: /lab5/lab5/
next: /lab7/lab7/
---

---
<!-- Aquí comienzan las instrucciones paso a paso de la práctica -->

## 📁 Preparación del directorio de trabajo

En esta práctica conservarás el directorio raíz del curso y crearás únicamente el subdirectorio correspondiente a `lab6`.

### 🗂️ Crear y abrir el subdirectorio de la práctica

- {% include step_label.html %} Abre **Docker Desktop** y confirma que el motor indique estado activo, porque `couchbase-lab` depende del daemon local para ejecutar todos sus servicios.
- {% include step_label.html %} Abre **Visual Studio Code** y espera su carga completa, ya que utilizarás el Explorador y la terminal integrada durante toda la práctica.
- {% include step_label.html %} Selecciona **File → Open Folder** en VS Code y abre `C:\LABS\couchbase-nosql` para trabajar dentro de la estructura común de los laboratorios.

  ```text
  C:\LABS\couchbase-nosql
  ```

- {% include step_label.html %} Selecciona **Terminal → New Terminal** en Visual Studio Code para abrir la consola integrada desde la que ejecutarás las operaciones de la práctica.
- {% include step_label.html %} Comprueba en el selector del panel Terminal que **Git Bash** sea el perfil activo, porque los comandos utilizan sintaxis y rutas compatibles con Bash.
- {% include step_label.html %} Crea el subdirectorio de la práctica 6 desde Git Bash para crear de forma idempotente el directorio `/c/LABS/couchbase-nosql/lab6` donde se organizarán los archivos de esta práctica.

  ```bash
  mkdir -p /c/LABS/couchbase-nosql/lab6
  ```

- {% include step_label.html %} Cambia al subdirectorio desde Git Bash para cambiar la ubicación activa a `/c/LABS/couchbase-nosql/lab6` y evitar operaciones posteriores desde un directorio incorrecto.

  ```bash
  cd /c/LABS/couchbase-nosql/lab6
  ```

- {% include step_label.html %} Confirma la ruta actual desde Git Bash para mostrar la ruta activa y confirmar que Git Bash está ubicado en el subdirectorio asignado a esta práctica.

  ```bash
  pwd
  ```

**Salida esperada:**

Para validar `crear y abrir el subdirectorio de trabajo`, verifica la referencia siguiente y confirma que la respuesta permita mostrar la ruta activa y confirmar que Git Bash está ubicado en el subdirectorio asignado a esta práctica; detente si aparece un error.

```text
/c/LABS/couchbase-nosql/lab6
```

---

## ⚙️ Tarea 1. Preparar el entorno de Eventing y crear las collections

En esta tarea validarás que el contenedor utiliza Couchbase Server Enterprise Edition, confirmarás que Eventing está habilitado y crearás la estructura de datos que utilizará la Function.

### Tarea 1.1. Verificar el contenedor Couchbase

- {% include step_label.html %} Consulta los contenedores y comprueba que `couchbase-lab` permanezca activo antes de utilizar sus servicios; confirma la respuesta esperada antes de continuar con la siguiente operación.

  {%raw%}
  ```bash
  docker ps --filter "name=couchbase-lab" --format "table {{.Names}}\t{{.Status}}"
  ```
  {%endraw%}

**Salida esperada:**

Para validar `Verificar el contenedor Couchbase`, verifica la referencia siguiente y confirma que la respuesta permita consultar los contenedores y comprobar que `couchbase-lab` permanezca activo antes de utilizar sus servicios; detente si aparece un error.

```text
NAMES           STATUS
couchbase-lab   Up ...
```

- {% include step_label.html %} Si no está activo, inícialo desde Git Bash para iniciar `couchbase-lab` cuando esté detenido y esperar la confirmación de Docker antes de continuar.

  ```bash
  docker start couchbase-lab
  ```

- {% include step_label.html %} Valida que la Web Console responda desde Git Bash para solicitar la Web Console por el puerto 8091 e interpretar el código HTTP como prueba de disponibilidad.

  ```bash
  curl -s -o /dev/null -w "Web Console: HTTP %{http_code}\n" \
    http://localhost:8091/ui/index.html
  ```

**Salida esperada:**

Para validar `Verificar el contenedor Couchbase`, verifica la referencia siguiente y confirma que la respuesta permita solicitar la Web Console por el puerto 8091 e interpretar el código HTTP como prueba de disponibilidad; detente si aparece un error.

```text
Web Console: HTTP 200
```

### Tarea 1.2. Verificar que el contenedor utiliza Enterprise Edition

- {% include step_label.html %} Consulta la configuración de `couchbase-lab` y verifica que utiliza la imagen Enterprise 7.6.2 requerida; confirma la respuesta esperada antes de continuar con la siguiente operación.

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

> **IMPORTANTE:** Si aparece `couchbase/server:community-7.6.2`, el contenedor anterior de Community Edition continúa activo. Vuelve a la Práctica 2 y recrea el entorno con Enterprise Edition, porque el servicio Eventing utilizado en esta práctica debe estar habilitado en ese contenedor.
{: .lab-note .important .compact}

### Tarea 1.3. Verificar el servicio Eventing

- {% include step_label.html %} Solicita `/pools/default/nodeServices` y revisa su código HTTP, estado o campos JSON antes de continuar; confirma la respuesta esperada antes de continuar con la siguiente operación.

  ```bash
  curl -s -u Administrator:Password123! \
    http://localhost:8091/pools/default/nodeServices \
    | python -m json.tool | grep -i eventing
  ```

**Resultado esperado:**

Para validar `Verificar el servicio Eventing`, verifica la referencia siguiente y confirma que la respuesta permita solicitar `/pools/default/nodeServices` y revisar su código HTTP, estado o campos JSON antes de continuar; detente si aparece un error.

Debes observar una referencia al servicio Eventing o al puerto asociado.

> **IMPORTANTE:** Si no aparece Eventing, no continúes creando Functions. Revisa la configuración del clúster realizada en la práctica 2.
{: .lab-note .important .compact}

### Tarea 1.4. Crear el bucket app-data

- {% include step_label.html %} Consulta `/pools/default/buckets` desde Git Bash para comprobar si `app-data` existe antes de intentar crearlo mediante la API REST.

  ```bash
  curl -s -u Administrator:Password123! \
    -X POST http://localhost:8091/pools/default/buckets \
    -d name=app-data \
    -d ramQuota=256 \
    -d bucketType=couchbase \
    -d replicaNumber=0
  ```

- {% include step_label.html %} Espera unos segundos desde Git Bash para aplicar el bloque que comienza con `sleep 5` y revisa su efecto específico antes de continuar.

  ```bash
  sleep 5
  ```

- {% include step_label.html %} Verifica que el bucket exista desde Git Bash para solicitar `/pools/default/buckets/app-data` y revisa su código HTTP, estado o campos JSON antes de continuar.

  ```bash
  curl -s -u Administrator:Password123! \
    http://localhost:8091/pools/default/buckets/app-data \
    | python -m json.tool | grep -E '"name"|"bucketType"'
  ```

### Tarea 1.5. Crear el scope y las collections

- {% include step_label.html %} Crea el scope `workshop` desde Git Bash para solicitar `/pools/default/buckets/app-data/scopes` y revisa su código HTTP, estado o campos JSON antes de continuar.

  ```bash
  curl -s -u Administrator:Password123! \
    -X POST \
    "http://localhost:8091/pools/default/buckets/app-data/scopes" \
    -d name=workshop
  ```

- {% include step_label.html %} Crea la collection fuente `orders` desde Git Bash para solicitar `/pools/default/buckets/app-data/scopes/workshop/collections` y revisa su código HTTP, estado o campos JSON antes de continuar.

  ```bash
  curl -s -u Administrator:Password123! \
    -X POST \
    "http://localhost:8091/pools/default/buckets/app-data/scopes/workshop/collections" \
    -d name=orders
  ```

- {% include step_label.html %} Crea la collection de auditoría desde Git Bash para solicitar `/pools/default/buckets/app-data/scopes/workshop/collections` y revisa su código HTTP, estado o campos JSON antes de continuar.

  ```bash
  curl -s -u Administrator:Password123! \
    -X POST \
    "http://localhost:8091/pools/default/buckets/app-data/scopes/workshop/collections" \
    -d name=audit-log
  ```

- {% include step_label.html %} Crea la collection de cuarentena desde Git Bash para solicitar `/pools/default/buckets/app-data/scopes/workshop/collections` y revisa su código HTTP, estado o campos JSON antes de continuar.

  ```bash
  curl -s -u Administrator:Password123! \
    -X POST \
    "http://localhost:8091/pools/default/buckets/app-data/scopes/workshop/collections" \
    -d name=quarantine
  ```

### Tarea 1.6. Crear el bucket de metadata

- {% include step_label.html %} Consulta nuevamente `/pools/default/buckets` y confirma que el bucket de metadata requerido por Eventing quedó disponible en el clúster.

  ```bash
  curl -s -u Administrator:Password123! \
    -X POST http://localhost:8091/pools/default/buckets \
    -d name=eventing-meta \
    -d ramQuota=256 \
    -d bucketType=couchbase \
    -d replicaNumber=0
  ```

> **NOTA:** El bucket `eventing-meta` almacenará checkpoints y estado interno de Eventing. No contiene datos de negocio y no debe modificarse manualmente.
{: .lab-note .info .compact}

### Tarea 1.7. Verificar la estructura creada

- {% include step_label.html %} Solicita `/pools/default/buckets/app-data/scopes` y revisa su código HTTP, estado o campos JSON antes de continuar.

  ```bash
  curl -s -u Administrator:Password123! \
    "http://localhost:8091/pools/default/buckets/app-data/scopes" \
    | python -m json.tool
  ```

**Validación:**

Para validar `Verificar la estructura creada`, verifica la referencia siguiente y confirma que la respuesta permita solicitar `/pools/default/buckets/app-data/scopes` y revisar su código HTTP, estado o campos JSON antes de continuar; detente si aparece un error.

Debes observar:

```text
Bucket: app-data
Scope: workshop
Collections:
- orders
- audit-log
- quarantine
```

### Tarea 1.8. Crear índices para las validaciones SQL++

- {% include step_label.html %} Abre `http://localhost:8091` en el navegador y espera la pantalla de autenticación para acceder a la Web Console del nodo local.

  ```text
  http://localhost:8091
  ```

- {% include step_label.html %} Inicia sesión en la Web Console con las credenciales del laboratorio y selecciona **Query** para ejecutar las validaciones SQL++ requeridas.
- {% include step_label.html %} Crea el índice `idx_lab6_orders_primary` para el patrón de consulta analizado; confirma la respuesta esperada antes de continuar con la siguiente operación.

  ```sql
  CREATE PRIMARY INDEX IF NOT EXISTS idx_lab6_orders_primary
  ON `app-data`.workshop.orders;
  ```

- {% include step_label.html %} Crea el índice `idx_lab6_audit_primary` para el patrón de consulta analizado; conserva el bloque completo y revisa la respuesta antes de continuar.

  ```sql
  CREATE PRIMARY INDEX IF NOT EXISTS idx_lab6_audit_primary
  ON `app-data`.workshop.`audit-log`;
  ```

- {% include step_label.html %} Crea el índice `idx_lab6_quarantine_primary` para el patrón de consulta analizado; conserva el bloque completo y revisa la respuesta antes de continuar.

  ```sql
  CREATE PRIMARY INDEX IF NOT EXISTS idx_lab6_quarantine_primary
  ON `app-data`.workshop.quarantine;
  ```

- {% include step_label.html %} Crea índices secundarios para las validaciones desde Query Workbench para crear el índice `idx_lab6_audit_source` para el patrón de consulta analizado.

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab6_audit_source
  ON `app-data`.workshop.`audit-log`(
    sourceDocId,
    eventType,
    timestamp
  );
  ```

- {% include step_label.html %} Crea el índice `idx_lab6_quarantine_original` para el patrón de consulta analizado; conserva el bloque completo y revisa la respuesta antes de continuar.

  ```sql
  CREATE INDEX IF NOT EXISTS idx_lab6_quarantine_original
  ON `app-data`.workshop.quarantine(
    originalDocId,
    quarantined
  );
  ```

- {% include step_label.html %} Verifica desde Query Workbench para inventariar en `system:indexes` los índices del bucket, scope y collections especificados por los filtros.

  ```sql
  SELECT name, keyspace_id, state
  FROM system:indexes
  WHERE bucket_id = "app-data"
    AND scope_id = "workshop"
  ORDER BY keyspace_id, name;
  ```

**Resultado esperado:**

Para validar `Crear índices para las validaciones SQL++`, verifica la referencia siguiente y confirma que la respuesta permita inventariar en `system:indexes` los índices del bucket, scope y collections especificados por los filtros; detente si aparece un error.

Todos los índices deben aparecer con `state = "online"`.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

{% include support-prompt.html task="tarea1" %}

---

## 🧩 Tarea 2. Crear y configurar la Function ProcessOrderEvents

En esta tarea crearás una Function que escuche cambios en `orders` y tenga acceso de escritura a `audit-log` y `quarantine`.

### Tarea 2.1. Abrir Eventing

- {% include step_label.html %} Selecciona **Eventing** en la navegación lateral para crear `ProcessOrderEvents` y configurar su fuente, metadata, bindings y código JavaScript.
- {% include step_label.html %} En **Eventing**, selecciona **Add Function** para abrir el formulario de creación y registrar la Function con su nombre y keyspace de origen.

### Tarea 2.2. Configurar la fuente y metadata

- {% include step_label.html %} Revisa en **Eventing** el bucket, scope y collection de origen, además del keyspace de metadata, porque ambos determinan qué mutaciones procesa la Function.

Completa el formulario con los siguientes valores:

| Sección de la interfaz   | Campo                   | Valor                                                   |
| ------------------------ | ----------------------- | ------------------------------------------------------- |
| Function Scope           | Bucket                  | `app-data`                                              |
| Function Scope           | Scope                   | `workshop`                                              |
| Listen To Location       | Bucket                  | `app-data`                                              |
| Listen To Location       | Scope                   | `workshop`                                              |
| Listen To Location       | Collection              | `orders`                                                |
| Eventing Storage         | Bucket                  | `eventing-meta`                                         |
| Eventing Storage         | Scope                   | `_default`                                              |
| Eventing Storage         | Collection              | `_default`                                              |
| Function Name            | Nombre                  | `ProcessOrderEvents`                                    |
| Deployment Feed Boundary | Límite de procesamiento | `Everything`                                            |
| Description              | Descripción opcional    | `Procesa los eventos generados en la colección orders.` |

### Tarea 2.3. Crear el binding de auditoría

- {% include step_label.html %} En la sección **Bindings** de la Function, agrega el binding solicitado para que el código pueda escribir en la collection de destino mediante su alias.
- {% include step_label.html %} Configura el binding de auditoría con el alias, bucket, scope, collection y acceso indicados para permitir escrituras controladas desde JavaScript.

| Campo | Valor |
|---|---|
| Alias | `auditLog` |
| Bucket | `app-data` |
| Scope | `workshop` |
| Collection | `audit-log` |
| Access | `Read/Write` |

### Tarea 2.4. Crear el binding de cuarentena

- {% include step_label.html %} En **Bindings**, agrega el segundo alias de collection y verifica bucket, scope y collection para evitar escrituras en un keyspace incorrecto.

| Campo | Valor |
|---|---|
| Alias | `quarantineCol` |
| Bucket | `app-data` |
| Scope | `workshop` |
| Collection | `quarantine` |
| Access | `Read/Write` |

### Tarea 2.5. Agregar el código JavaScript

- {% include step_label.html %} Selecciona el editor de código de `ProcessOrderEvents` para sustituir la plantilla inicial por la lógica JavaScript que procesará mutaciones.
- {% include step_label.html %} Reemplaza la plantilla del editor por el código JavaScript proporcionado y conserva sin cambios `OnUpdate`, `OnDelete`, aliases y mensajes de log.

```javascript
// Function: ProcessOrderEvents
//
// Bindings requeridos:
// auditLog      -> app-data.workshop.audit-log  (Read/Write)
// quarantineCol -> app-data.workshop.quarantine (Read/Write)

/**
 * Devuelve la lista de estados permitidos.
 *
 * Couchbase Eventing no permite variables globales, por lo que la lista
 * se devuelve desde una función auxiliar.
 */
function getValidStatuses() {
    return [
        "pending",
        "processing",
        "completed",
        "cancelled"
    ];
}

/**
 * Genera un identificador para la mutación.
 */
function getMutationId(meta) {
    if (meta && meta.cas !== undefined && meta.cas !== null) {
        return String(meta.cas);
    }

    return String(Date.now());
}

/**
 * Se ejecuta cuando se crea o modifica un documento en:
 * app-data.workshop.orders
 */
function OnUpdate(doc, meta) {
    if (!doc || typeof doc !== "object") {
        log(
            "ProcessOrderEvents",
            "Documento omitido porque no contiene un objeto JSON válido.",
            meta && meta.id ? meta.id : "unknown"
        );
        return;
    }

    if (doc.type !== "order") {
        log(
            "ProcessOrderEvents",
            "Documento omitido porque type no es order.",
            meta.id
        );
        return;
    }

    var validStatuses = getValidStatuses();
    var now = new Date().toISOString();
    var status = doc.status;
    var isValid = validStatuses.indexOf(status) !== -1;
    var mutationId = getMutationId(meta);

    var auditKey =
        "audit::" + meta.id + "::" + mutationId;

    var auditRecord = {
        type: "audit",
        eventType: "MUTATION",
        sourceDocId: meta.id,
        sourceCas: mutationId,
        timestamp: now,
        userId:
            doc.userId !== undefined && doc.userId !== null
                ? doc.userId
                : "system",
        status:
            status !== undefined && status !== null
                ? status
                : null,
        validStatus: isValid,
        itemCount:
            Array.isArray(doc.items)
                ? doc.items.length
                : 0,
        totalAmount:
            typeof doc.totalAmount === "number"
                ? doc.totalAmount
                : 0
    };

    auditLog[auditKey] = auditRecord;

    log(
        "ProcessOrderEvents",
        "Auditoría creada.",
        "Documento:",
        meta.id,
        "Status válido:",
        isValid
    );

    if (!isValid) {
        var displayedStatus =
            status === undefined || status === null
                ? "undefined"
                : String(status);

        var quarantineKey =
            "quarantine::" + meta.id + "::" + mutationId;

        var quarantineRecord = {
            type: "quarantine",
            quarantined: true,
            originalDocId: meta.id,
            sourceCas: mutationId,
            quarantinedAt: now,
            quarantineReason:
                "Invalid status value: '" +
                displayedStatus +
                "'. Allowed values: " +
                validStatuses.join(", "),
            originalDocument: doc
        };

        quarantineCol[quarantineKey] = quarantineRecord;

        log(
            "ProcessOrderEvents",
            "Documento copiado a cuarentena.",
            meta.id
        );
    }
}

/**
 * Se ejecuta cuando un documento se elimina o expira.
 */
function OnDelete(meta, options) {
    var now = new Date().toISOString();
    var expired =
        options !== undefined &&
        options !== null &&
        options.expired === true;

    var eventType = expired
        ? "EXPIRATION"
        : "DELETE";

    var mutationId = getMutationId(meta);

    var auditKey =
        "audit::" +
        meta.id +
        "::" +
        eventType.toLowerCase() +
        "::" +
        mutationId;

    var auditRecord = {
        type: "audit",
        eventType: eventType,
        sourceDocId: meta.id,
        sourceCas: mutationId,
        timestamp: now,
        expired: expired
    };

    auditLog[auditKey] = auditRecord;

    log(
        "ProcessOrderEvents",
        expired
            ? "Expiración registrada."
            : "Eliminación registrada.",
        meta.id
    );
}
```

### Tarea 2.6. Configurar la Function

- {% include step_label.html %} Revisa la configuración general de `ProcessOrderEvents` y confirma que la fuente, metadata y lenguaje JavaScript coincidan con los valores indicados.

En **Settings**, utiliza:

| Parámetro | Valor |
|---|---|
| Worker Count | `1` |
| Log Level | `INFO` |

> **NOTA:** Un worker es suficiente para este laboratorio. En producción, el valor depende del volumen de eventos y de los recursos del clúster.
{: .lab-note .info .compact}

### Tarea 2.7. Guardar la Function

- {% include step_label.html %} Selecciona **Save and Return** para guardar código, bindings y configuración; confirma que la Web Console regrese a la lista de Functions sin errores.
- {% include step_label.html %} Confirma en la lista de **Eventing** que `ProcessOrderEvents` muestre `Undeployed`, señal de que la definición está guardada pero todavía no procesa eventos.

  ```text
  Undeployed
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

{% include support-prompt.html task="tarea2" %}

---

## 🚀 Tarea 3. Desplegar la Function y validar auditoría

En esta tarea desplegarás la Function, insertarás y actualizarás pedidos y comprobarás que se generen registros de auditoría.

### Tarea 3.1. Desplegar desde eventos nuevos

- {% include step_label.html %} En **Eventing**, localiza `ProcessOrderEvents` en la lista de Functions y comprueba su estado actual antes de solicitar el despliegue.
- {% include step_label.html %} Selecciona **Deploy** para compilar y activar `ProcessOrderEvents`; confirma el diálogo y espera el estado desplegado antes de insertar documentos.

- {% include step_label.html %} Espera en **Eventing** hasta que `ProcessOrderEvents` muestre `Deployed`; no insertes pedidos mientras la compilación o distribución permanezca pendiente.

  ```text
  Deployed
  ```

### Tarea 3.2. Insertar un pedido válido

- {% include step_label.html %} Selecciona **Query** en la navegación lateral de la Web Console para abrir Query Workbench y ejecutar las sentencias SQL++ de la práctica.
- {% include step_label.html %} Inserta `order::1001` en `app-data.workshop.orders` para activar `OnUpdate` y generar la primera entrada en la collection de auditoría.

  ```sql
  INSERT INTO `app-data`.workshop.orders (KEY, VALUE)
  VALUES (
    "order::1001",
    {
      "type": "order",
      "orderId": "1001",
      "userId": "user::ana.garcia",
      "status": "pending",
      "items": [
        {
          "productId": "prod::A1",
          "name": "Laptop",
          "qty": 1,
          "price": 1200.00
        },
        {
          "productId": "prod::B2",
          "name": "Mouse",
          "qty": 2,
          "price": 25.00
        }
      ],
      "totalAmount": 1250.00
    }
  );
  ```

- {% include step_label.html %} Espera entre tres y cinco segundos para que Eventing procese `order::1001` y cree la primera entrada de auditoría antes de consultarla.

### Tarea 3.3. Verificar la auditoría

- {% include step_label.html %} Consulta `app-data.workshop.audit-log` con el filtro `a.sourceDocId = "order::1001"` y comprueba las filas y el orden obtenidos.

  ```sql
  SELECT META(a).id AS auditId,
         a.eventType,
         a.sourceDocId,
         a.timestamp,
         a.status,
         a.validStatus,
         a.itemCount,
         a.totalAmount
  FROM `app-data`.workshop.`audit-log` AS a
  WHERE a.sourceDocId = "order::1001"
  ORDER BY a.timestamp DESC;
  ```

**Resultado esperado:**

Para validar `Verificar la auditoría`, verifica la referencia siguiente y confirma que la respuesta permita consultar ``app-data`.workshop.`audit-log`` con el filtro `a.sourceDocId = "order::1001"` y comprobar las filas y el orden obtenidos; detente si aparece un error.

Debe existir al menos un registro con:

```text
eventType = MUTATION
sourceDocId = order::1001
status = pending
validStatus = true
```

### Tarea 3.4. Actualizar el pedido

- {% include step_label.html %} Actualiza los documentos de `app-data.workshop.orders` seleccionados por la clave o condición y verifica los campos modificados.

  ```sql
  UPDATE `app-data`.workshop.orders
  SET status = "processing"
  WHERE META().id = "order::1001";
  ```

- {% include step_label.html %} Espera entre tres y cinco segundos después del `UPDATE` para que Eventing registre el nuevo estado de `order::1001` en auditoría.
- {% include step_label.html %} Consulta nuevamente `audit-log` y verifica que la entrada de `order::1001` refleje `status = processing` después de actualizar el pedido.

**Validación:**

Para validar `Actualizar el pedido`, verifica la referencia siguiente y confirma que la respuesta permita actualizar los documentos de ``app-data`.workshop.orders` seleccionados por la clave o condición y verificar los campos modificados; detente si aparece un error.

Debes observar un nuevo registro con:

```text
status = processing
validStatus = true
```

### Tarea 3.5. Revisar logs

- {% include step_label.html %} Regresa a **Eventing** y abre los logs de `ProcessOrderEvents` para relacionar el mensaje de auditoría con la mutación de `order::1001`.
- {% include step_label.html %} Abre los logs de `ProcessOrderEvents` y localiza los mensajes producidos al crear y actualizar `order::1001` durante la prueba de auditoría.
- {% include step_label.html %} Busca en los logs el mensaje `Auditoría creada para order::1001` y confirma que corresponda con el documento procesado durante la prueba.

  ```text
  Auditoría creada para order::1001
  ```

**Validación:**

Para validar `Revisar logs`, verifica la referencia siguiente y confirma que la respuesta permita utilizar `Auditoría creada para order::1001` como referencia visible y comprobar que coincida con el estado o valor requerido; detente si aparece un error.

Debe existir al menos un mensaje por cada mutación procesada.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

{% include support-prompt.html task="tarea3" %}

---

## 🛡️ Tarea 4. Validar estados y copiar documentos a cuarentena

En esta tarea comprobarás que la misma Function identifica pedidos con estados inválidos y crea una copia controlada en `quarantine`.

### Tarea 4.1. Insertar un pedido válido

- {% include step_label.html %} Inserta `order::2001` con todos los campos requeridos para comprobar que la Function acepta el pedido válido sin copiarlo a cuarentena.

  ```sql
  INSERT INTO `app-data`.workshop.orders (KEY, VALUE)
  VALUES (
    "order::2001",
    {
      "type": "order",
      "orderId": "2001",
      "userId": "user::maria.lopez",
      "status": "completed",
      "items": [
        {
          "productId": "prod::C3",
          "name": "Monitor",
          "qty": 1,
          "price": 450.00
        }
      ],
      "totalAmount": 450.00
    }
  );
  ```

- {% include step_label.html %} Espera tres segundos para que Eventing valide `order::2001`; después confirma que el pedido correcto no aparezca en cuarentena.
- {% include step_label.html %} Consulta `quarantine` para `order::2001` y confirma que el pedido válido no fue copiado, porque cumple los campos requeridos por la Function.

  ```sql
  SELECT COUNT(*) AS total
  FROM `app-data`.workshop.quarantine
  WHERE originalDocId = "order::2001";
  ```

**Resultado esperado:**

Para validar `Insertar un pedido válido`, verifica la referencia siguiente y confirma que la respuesta permita consultar ``app-data`.workshop.quarantine` con el filtro `originalDocId = "order::2001"` y comprobar las filas y el orden obtenidos; detente si aparece un error.

```json
[
  {
    "total": 0
  }
]
```

### Tarea 4.2. Insertar un pedido inválido

- {% include step_label.html %} Inserta `order::2002` con datos incompletos para activar la regla de validación y comprobar su copia controlada hacia `quarantine`.

  ```sql
  INSERT INTO `app-data`.workshop.orders (KEY, VALUE)
  VALUES (
    "order::2002",
    {
      "type": "order",
      "orderId": "2002",
      "userId": "user::pedro.ramirez",
      "status": "shipped",
      "items": [
        {
          "productId": "prod::D4",
          "name": "SSD 1TB",
          "qty": 1,
          "price": 120.00
        }
      ],
      "totalAmount": 120.00
    }
  );
  ```

- {% include step_label.html %} Espera entre tres y cinco segundos para que Eventing rechace `order::2002`, lo copie a cuarentena y escriba la auditoría correspondiente.

### Tarea 4.3. Verificar la cuarentena

- {% include step_label.html %} Consulta `app-data.workshop.quarantine` con el filtro `q.originalDocId = "order::2002"` y comprueba las filas y el orden obtenidos.

  ```sql
  SELECT META(q).id AS quarantineId,
         q.originalDocId,
         q.quarantined,
         q.quarantinedAt,
         q.quarantineReason,
         q.originalDocument.status AS invalidStatus
  FROM `app-data`.workshop.quarantine AS q
  WHERE q.originalDocId = "order::2002"
  ORDER BY q.quarantinedAt DESC;
  ```

**Resultado esperado:**

Para validar `Verificar la cuarentena`, verifica la referencia siguiente y confirma que la respuesta permita consultar ``app-data`.workshop.quarantine` con el filtro `q.originalDocId = "order::2002"` y comprobar las filas y el orden obtenidos; detente si aparece un error.

Debes obtener al menos un documento con:

```text
originalDocId = order::2002
quarantined = true
invalidStatus = shipped
```

### Tarea 4.4. Confirmar la auditoría del evento inválido

- {% include step_label.html %} Consulta `app-data.workshop.audit-log` con el filtro `sourceDocId = "order::2002"` y comprueba las filas y el orden obtenidos.

  ```sql
  SELECT eventType,
         sourceDocId,
         status,
         validStatus,
         timestamp
  FROM `app-data`.workshop.`audit-log`
  WHERE sourceDocId = "order::2002"
  ORDER BY timestamp DESC;
  ```

**Validación:**

Para validar `Confirmar la auditoría del evento inválido`, verifica la referencia siguiente y confirma que la respuesta permita consultar ``app-data`.workshop.`audit-log`` con el filtro `sourceDocId = "order::2002"` y comprobar las filas y el orden obtenidos; detente si aparece un error.

Debes observar:

```text
eventType = MUTATION
status = shipped
validStatus = false
```

> **IMPORTANTE:** La Function copia el documento a cuarentena, pero conserva el original en `orders`. Esto permite revisión sin pérdida inmediata de información.
{: .lab-note .important .compact}

### Tarea 4.5. Revisar logs de validación

- {% include step_label.html %} Regresa a **Eventing** y abre los logs para comprobar que la Function informó la copia de `order::2002` hacia la collection de cuarentena.
- {% include step_label.html %} Abre los logs de `ProcessOrderEvents` y busca el mensaje de validación que explica por qué `order::2002` fue enviado a cuarentena.
- {% include step_label.html %} Busca en los logs el mensaje de cuarentena para `order::2002` y relaciónalo con la validación que rechazó el documento incompleto.

  ```text
  Documento copiado a cuarentena: order::2002
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

{% include support-prompt.html task="tarea4" %}

---

## 📈 Tarea 5. Probar OnDelete, revisar métricas y hacer undeploy

En esta tarea generarás un evento de eliminación, revisarás la actividad de la Function y completarás el ciclo operativo mediante undeploy.

### Tarea 5.1. Eliminar un pedido

- {% include step_label.html %} Elimina únicamente los documentos seleccionados por la condición y comprueba el comportamiento asociado a esa operación.

  ```sql
  DELETE FROM `app-data`.workshop.orders
  WHERE META().id = "order::1001";
  ```

- {% include step_label.html %} Espera entre tres y cinco segundos después del `DELETE` para que `OnDelete` registre el evento de eliminación de `order::1001`.

### Tarea 5.2. Validar el evento DELETE

- {% include step_label.html %} Consulta `app-data.workshop.audit-log` con el filtro `sourceDocId = "order::1001" AND eventType = "DELETE"` y comprueba las filas y el orden obtenidos.

  ```sql
  SELECT eventType,
         sourceDocId,
         timestamp,
         expired
  FROM `app-data`.workshop.`audit-log`
  WHERE sourceDocId = "order::1001"
    AND eventType = "DELETE"
  ORDER BY timestamp DESC;
  ```

**Resultado esperado:**

Para validar `Validar el evento DELETE`, verifica la referencia siguiente y confirma que la respuesta permita consultar ``app-data`.workshop.`audit-log`` con el filtro `sourceDocId = "order::1001" AND eventType = "DELETE"` y comprobar las filas y el orden obtenidos; detente si aparece un error.

Debe existir un registro con:

```text
eventType = DELETE
sourceDocId = order::1001
expired = false
```

### Tarea 5.3. Revisar métricas de Eventing

- {% include step_label.html %} Selecciona **Eventing** en la navegación lateral y abre `ProcessOrderEvents` para revisar las métricas acumuladas durante las pruebas.
- {% include step_label.html %} Selecciona `ProcessOrderEvents` en **Eventing** para abrir su panel de detalles y revisa contadores, backlog y resultados de ejecución.
- {% include step_label.html %} En el panel de `ProcessOrderEvents`, revisa contadores de éxito, fallos, backlog y ejecución para confirmar que la Function procesó los eventos esperados.

  - Eventos procesados.
  - Fallos.
  - Mutaciones pendientes.
  - Actividad reciente.
  - Logs generados.

> **NOTA:** Los nombres exactos de las métricas pueden variar según la versión de Couchbase Server.
{: .lab-note .info .compact}

### Tarea 5.4. Ejecutar validaciones finales

- {% include step_label.html %} Consulta `app-data.workshop.audit-log` con el filtro `type = "audit"` y comprueba las filas y el orden obtenidos.

  ```sql
  SELECT eventType,
         COUNT(*) AS total,
         MIN(timestamp) AS primerEvento,
         MAX(timestamp) AS ultimoEvento
  FROM `app-data`.workshop.`audit-log`
  WHERE type = "audit"
  GROUP BY eventType
  ORDER BY eventType;
  ```

- {% include step_label.html %} Consulta `app-data.workshop.quarantine` con el filtro `q.quarantined = true` y comprueba las filas y el orden obtenidos.

  ```sql
  SELECT META(q).id AS quarantineId,
         q.originalDocId,
         q.quarantineReason
  FROM `app-data`.workshop.quarantine AS q
  WHERE q.quarantined = true
  ORDER BY q.quarantinedAt;
  ```

### Tarea 5.5. Hacer undeploy

- {% include step_label.html %} Regresa a **Eventing**, localiza `ProcessOrderEvents` y verifica su estado antes de solicitar el undeploy de la Function.
- {% include step_label.html %} Localiza `ProcessOrderEvents` en la lista de **Eventing** y confirma que esté `Deployed` antes de iniciar el proceso de undeploy.
- {% include step_label.html %} Selecciona **Undeploy** para detener el procesamiento de nuevas mutaciones y espera la confirmación antes de probar un pedido posterior.
- {% include step_label.html %} Confirma el undeploy en el diálogo de la Web Console para detener la Function sin eliminar su código, bindings ni configuración guardada.
- {% include step_label.html %} Espera en **Eventing** hasta que `ProcessOrderEvents` muestre `Undeployed`, confirmando que dejó de recibir mutaciones nuevas.

  ```text
  Undeployed
  ```

> **IMPORTANTE:** Undeploy detiene el procesamiento de eventos, pero conserva la Function, su código y su configuración para revisión posterior.
{: .lab-note .important .compact}

### Tarea 5.6. Validar que ya no procesa eventos

- {% include step_label.html %} Inserta `order::9999` después del undeploy para comprobar que la collection fuente acepta el documento pero Eventing ya no crea auditoría.

  ```sql
  INSERT INTO `app-data`.workshop.orders (KEY, VALUE)
  VALUES (
    "order::9999",
    {
      "type": "order",
      "orderId": "9999",
      "userId": "user::validation",
      "status": "pending",
      "items": [],
      "totalAmount": 0
    }
  );
  ```

- {% include step_label.html %} Espera cinco segundos después de la inserción para demostrar que una Function en estado `Undeployed` ya no genera documentos de auditoría.
- {% include step_label.html %} Consulta `app-data.workshop.audit-log` con el filtro `sourceDocId = "order::9999"` y comprueba las filas y el orden obtenidos.

  ```sql
  SELECT COUNT(*) AS total
  FROM `app-data`.workshop.`audit-log`
  WHERE sourceDocId = "order::9999";
  ```

**Resultado esperado:**

Para validar `Validar que ya no procesa eventos`, verifica la referencia siguiente y confirma que la respuesta permita consultar ``app-data`.workshop.`audit-log`` con el filtro `sourceDocId = "order::9999"` y comprobar las filas y el orden obtenidos; detente si aparece un error.

```json
[
  {
    "total": 0
  }
]
```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

{% include support-prompt.html task="tarea5" %}

---

## 🛠️ Resolución de problemas

### Problema 1. Eventing no aparece en la Web Console

**Causa probable:** El servicio Eventing no está habilitado en el nodo.

**Validación:**

Para validar `Validar que ya no procesa eventos`, verifica la referencia siguiente y confirma que la respuesta permita comprobar el resultado descrito para `Validar que ya no procesa eventos`; detente si aparece un error.

```bash
curl -s -u Administrator:Password123! \
  http://localhost:8091/pools/default/nodeServices \
  | python -m json.tool | grep -i eventing
```

Si no aparece, verifica que el contenedor utilice la imagen Enterprise y revisa la configuración de servicios realizada en la práctica 2.

### Problema 2. El contenedor activo utiliza Community Edition

**Síntoma:** la validación de imagen muestra `couchbase/server:community-7.6.2` o Eventing no aparece en la Web Console.

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

Si continúa apareciendo Community Edition, vuelve a la Práctica 2 y recrea el contenedor con Enterprise Edition. Después habilita Eventing durante la configuración del clúster.

### Problema 3. La Function queda en Deploying

**Causa probable:** El servicio Eventing todavía está inicializando, la metadata no es accesible o existe poca memoria disponible.

**Solución:**

Para resolver `La Function queda en Deploying`, aplica el diagnóstico en el componente indicado, interpreta la respuesta y confirma la recuperación antes de retomar la práctica.

- Verifica que `eventing-meta` exista.
- Confirma que el contenedor tenga memoria suficiente.
- Revisa los logs de Eventing.
- Espera unos segundos y actualiza la pantalla.

### Problema 4. No se crean documentos en audit-log

**Causa probable:** La Function no está desplegada, el source collection es incorrecto o el binding `auditLog` no apunta a la collection adecuada.

**Verifica:**

```text
Source: app-data.workshop.orders
Binding auditLog: app-data.workshop.audit-log
Status: Deployed
```

### Problema 5. Las consultas muestran No index available

**Solución:**

Para resolver `Las consultas muestran No index available`, aplica el diagnóstico en el componente indicado, interpreta la respuesta y confirma la recuperación antes de retomar la práctica.

```sql
SELECT name, keyspace_id, state
FROM system:indexes
WHERE bucket_id = "app-data"
  AND scope_id = "workshop"
ORDER BY keyspace_id, name;
```

Si falta algún índice primario, créalo nuevamente con `IF NOT EXISTS`.

### Problema 6. Un pedido inválido no aparece en quarantine

**Causa probable:** El valor usado sí pertenece a la lista permitida o el binding está mal configurado.

Valores permitidos:

```text
pending
processing
completed
cancelled
```

Utiliza un valor como:

```text
shipped
```

para provocar la cuarentena.

### Problema 7. El código muestra un error al guardar

**Causa probable:** Existe un error de sintaxis JavaScript.

Revisa especialmente:

- Comillas.
- Llaves.
- Paréntesis.
- Comas.
- Alias `auditLog`.
- Alias `quarantineCol`.

### Problema 8. python -m json.tool falla

**Solución:**

Para resolver `python -m json.tool falla`, aplica el diagnóstico en el componente indicado, interpreta la respuesta y confirma la recuperación antes de retomar la práctica.

```bash
python --version
python3 --version
```

Si `python3` funciona, reemplaza:

Aplica el bloque que comienza con `python -m json.tool` y revisar su efecto específico antes de continuar; conserva el bloque completo y revisa la respuesta antes de continuar.

```bash
python -m json.tool
```

por:

Aplica el bloque que comienza con `python3 -m json.tool` y revisar su efecto específico antes de continuar; conserva el bloque completo y revisa la respuesta antes de continuar.

```bash
python3 -m json.tool
```
