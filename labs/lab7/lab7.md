---
layout: lab
title: "Práctica 7: Diseño de modelo de datos para una aplicación distribuida"
permalink: /lab7/lab7/
images_base: /labs/lab7/img
duration: "60 minutos"
objective:
  - Preparar el directorio de trabajo de la práctica 7 y validar el acceso a Couchbase Server Enterprise Edition.
  - Identificar las entidades principales de una aplicación de comercio electrónico y documentar sus patrones de acceso.
  - Transformar el modelo conceptual en documentos JSON mediante decisiones justificadas de embedding, referencing y snapshot.
  - Diseñar la estructura física del modelo mediante bucket, scope, collections, document keys e índices.
  - Crear la estructura física en Couchbase e insertar un conjunto reducido y coherente de documentos.
  - Validar los patrones de acceso mediante consultas SQL++, acceso por document key, actualización por key e índices de array.
prerequisites:
  - Haber completado la Práctica 3 sobre consultas SQL++.
  - Haber completado la Práctica 4 sobre creación y validación de índices.
  - Haber completado la Práctica 5 sobre EXPLAIN y optimización básica.
  - Tener Docker Desktop en ejecución.
  - Tener activo el contenedor couchbase-lab creado con la imagen couchbase/server:enterprise-7.6.2.
  - Tener acceso a la Web Console en http://localhost:8091.
  - Utilizar Visual Studio Code con Git Bash como terminal integrada.
  - Conocer la estructura básica de documentos JSON, objetos anidados y arreglos.
introduction:
  - En esta práctica utilizarás Couchbase Server Enterprise Edition para diseñar el modelo de datos de una aplicación de comercio electrónico aplicando tres niveles de modelado: conceptual, lógico y físico. Primero identificarás entidades y patrones de acceso; después decidirás qué información se embebe, qué información se referencia y qué datos deben conservarse como snapshots históricos. Finalmente crearás el bucket ecommerce, el scope store, cinco collections, los índices necesarios e insertarás documentos de prueba para verificar que el modelo responde correctamente a los patrones de acceso definidos.
slug: lab7
lab_number: 7
final_result: >
  Al finalizar la práctica habrás documentado un modelo conceptual y lógico para una aplicación de comercio electrónico, implementado su estructura física en Couchbase y validado cinco patrones de acceso. Podrás justificar cuándo utilizar embedding, referencing y snapshots, diseñar document keys consistentes, crear índices de array y comprobar que el modelo soporta lecturas y actualizaciones de negocio.
notes:
  - Todos los comandos de terminal deben ejecutarse desde Git Bash dentro de Visual Studio Code.
  - Utiliza las credenciales Administrator y Password123! configuradas en las prácticas anteriores.
  - Los recursos se crean con IF NOT EXISTS cuando la sintaxis lo permite para que la práctica pueda repetirse.
  - Los índices creados en esta práctica utilizan el prefijo idx_lab7_.
  - No elimines el bucket ecommerce ni las collections al finalizar; se conservarán para prácticas posteriores.
references: []
prev: /lab6/lab6/
next: /lab8/lab8/
---

---
<!-- Aquí comienzan las instrucciones paso a paso de la práctica -->

## 📁 Preparación del directorio de trabajo

En esta práctica conservarás el directorio raíz del curso y crearás únicamente el subdirectorio correspondiente a `lab7`.

### 🗂️ Crear y abrir el subdirectorio de la práctica

- {% include step_label.html %} Abre **Docker Desktop** y confirma que el motor indique estado activo, porque `couchbase-lab` depende del daemon local para ejecutar todos sus servicios.
- {% include step_label.html %} Abre **Visual Studio Code** y espera su carga completa, ya que utilizarás el Explorador y la terminal integrada durante toda la práctica.
- {% include step_label.html %} Selecciona **File → Open Folder** en VS Code y abre `C:\LABS\couchbase-nosql` para mantener esta práctica dentro de la estructura del curso.

  ```text
  C:\LABS\couchbase-nosql
  ```

- {% include step_label.html %} Selecciona **Terminal → New Terminal** en Visual Studio Code para abrir la consola integrada desde la que ejecutarás las operaciones de la práctica.
- {% include step_label.html %} Comprueba en el selector del panel Terminal que **Git Bash** sea el perfil activo, porque los comandos utilizan sintaxis y rutas compatibles con Bash.
- {% include step_label.html %} Crea el subdirectorio de la práctica desde Git Bash para crear de forma idempotente el directorio `/c/LABS/couchbase-nosql/lab7` donde se organizarán los archivos de esta práctica.

  ```bash
  mkdir -p /c/LABS/couchbase-nosql/lab7
  ```

- {% include step_label.html %} Cambia al subdirectorio desde Git Bash para cambiar la ubicación activa a `/c/LABS/couchbase-nosql/lab7` y evitar operaciones posteriores desde un directorio incorrecto.

  ```bash
  cd /c/LABS/couchbase-nosql/lab7
  ```

- {% include step_label.html %} Confirma la ruta desde Git Bash para mostrar la ruta activa y confirmar que Git Bash está ubicado en el subdirectorio asignado a esta práctica.

  ```bash
  pwd
  ```

**Salida esperada:**

Para validar `crear y abrir el subdirectorio de trabajo`, verifica la referencia siguiente y confirma que la respuesta permita mostrar la ruta activa y confirmar que Git Bash está ubicado en el subdirectorio asignado a esta práctica; detente si aparece un error.

```text
/c/LABS/couchbase-nosql/lab7
```

---

## 🧭 Tarea 1. Identificar entidades y patrones de acceso

En esta tarea definirás el modelo conceptual de la aplicación antes de crear cualquier bucket o documento. El objetivo es diseñar a partir de cómo se usará la información y no únicamente a partir de las entidades del negocio.

### Tarea 1.1. Verificar Couchbase Server

- {% include step_label.html %} Consulta los contenedores y comprueba que `couchbase-lab` permanezca activo antes de utilizar sus servicios; confirma el resultado correspondiente antes de continuar con la siguiente acción.

  {%raw%}
  ```bash
  docker ps --filter "name=couchbase-lab" --format "table {{.Names}}\t{{.Status}}"
  ```
  {%endraw%}

- {% include step_label.html %} Si el contenedor no está activo, inícialo desde Git Bash para iniciar `couchbase-lab` cuando esté detenido y esperar la confirmación de Docker antes de continuar.

  ```bash
  docker start couchbase-lab
  ```

- {% include step_label.html %} Verifica la Web Console desde Git Bash para solicitar la Web Console por el puerto 8091 e interpretar el código HTTP como prueba de disponibilidad.

  ```bash
  curl -s -o /dev/null -w "Web Console: HTTP %{http_code}\n" \
    http://localhost:8091/ui/index.html
  ```

**Salida esperada:**

Para validar `Verificar Couchbase Server`, verifica la referencia siguiente y confirma que la respuesta permita solicitar la Web Console por el puerto 8091 e interpretar el código HTTP como prueba de disponibilidad; detente si aparece un error.

```text
Web Console: HTTP 200
```

### Tarea 1.2. Verificar que el contenedor utiliza Enterprise Edition

- {% include step_label.html %} Consulta la configuración de `couchbase-lab` y verifica que utiliza la imagen Enterprise 7.6.2 requerida; confirma el resultado correspondiente antes de continuar con la siguiente acción.

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

### Tarea 1.3. Crear el documento de diseño

- {% include step_label.html %} Crea `lab7-data-model.md` desde el Explorador de VS Code para concentrar entidades, relaciones, esquemas, claves, índices y decisiones de diseño.

  ```text
  ecommerce-data-model.md
  ```

- {% include step_label.html %} Agrega al archivo el encabezado Markdown proporcionado para organizar las decisiones del modelo y conservar una evidencia legible de la práctica.

  ```markdown
  # Modelo de datos de la aplicación de comercio electrónico

  ## Objetivo

  Diseñar un modelo documental en Couchbase a partir de los patrones
  de acceso de una aplicación de comercio electrónico simplificada.
  ```

### Tarea 1.4. Identificar las entidades conceptuales

- {% include step_label.html %} Registra en `lab7-data-model.md` las entidades Customer, Product, Category, Order, OrderItem y Review, junto con la responsabilidad de cada una.

Registra:

```markdown
## Entidades conceptuales

| Entidad | Propósito |
|---|---|
| Customer | Representa al cliente que realiza pedidos y escribe reseñas. |
| Product | Representa un producto disponible para venta. |
| Category | Clasifica productos dentro del catálogo. |
| Order | Representa una transacción de compra. |
| OrderItem | Representa cada producto incluido dentro de un pedido. |
| Review | Representa la opinión de un cliente sobre un producto. |
```

> **IMPORTANTE:** `OrderItem` es una entidad conceptual, pero no se convertirá en una collection independiente. Se embebirá dentro del documento `Order`.
{: .lab-note .important .compact}

### Tarea 1.5. Documentar las relaciones

- {% include step_label.html %} Describe en `lab7-data-model.md` las relaciones entre clientes, pedidos, productos, categorías y reseñas, indicando sus cardinalidades principales.

Agrega:

```markdown
## Relaciones conceptuales

- Un Customer puede realizar muchos Orders.
- Un Order contiene uno o más OrderItems.
- Cada OrderItem corresponde a un Product.
- Un Product puede pertenecer a una o más Categories.
- Un Product puede recibir muchas Reviews.
- Un Customer puede escribir muchas Reviews.
```

### Tarea 1.6. Definir los patrones de acceso

- {% include step_label.html %} Anota los patrones de acceso que deberá resolver el modelo, incluidos pedidos por cliente, productos por categoría y reseñas aprobadas.

Agrega:

```markdown
## Patrones de acceso

| ID | Patrón de acceso | Frecuencia | Tipo |
|---|---|---|---|
| PA-1 | Obtener el detalle completo de un pedido por su document key. | Muy alta | Lectura |
| PA-2 | Listar los pedidos de un cliente ordenados por fecha. | Alta | Lectura |
| PA-3 | Buscar productos activos por categoría y ordenarlos por precio. | Alta | Lectura |
| PA-4 | Actualizar el estado de un pedido mediante su document key. | Media | Escritura |
| PA-5 | Listar reseñas aprobadas de un producto y calcular su promedio. | Media | Lectura |
```

### Tarea 1.7. Validar el modelo conceptual

- {% include step_label.html %} Revisa entidades, relaciones y patrones de acceso para confirmar que el modelo conceptual cubra las consultas requeridas antes de diseñar documentos.

Antes de continuar, confirma:

- Existen seis entidades conceptuales.
- `OrderItem` se entiende como parte del pedido.
- `Review` tiene relación con `Customer` y `Product`.
- Cada patrón de acceso representa una operación real de la aplicación.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

{% include support-prompt.html task="tarea1" %}

---

## 🧱 Tarea 2. Diseñar los documentos JSON

En esta tarea transformarás el modelo conceptual en un modelo lógico orientado a documentos. Cada decisión debe relacionarse con los patrones de acceso definidos.

### Tarea 2.1. Definir OrderItem como información embebida

- {% include step_label.html %} Justifica por qué `OrderItem` se embebe dentro de cada pedido, considerando lectura conjunta, cantidad controlada y ciclo de vida compartido.

Agrega:

```markdown
## Decisiones de modelado lógico

### Decisión 1. Embeber OrderItem dentro de Order

Los ítems se almacenarán dentro del documento Order porque siempre se
consultan junto con el pedido, tienen un tamaño controlado y no requieren
un ciclo de vida independiente.

Patrón relacionado: PA-1.

Trade-off: para realizar análisis globales de productos vendidos será
necesario consultar los documentos de pedidos o mantener un modelo
agregado adicional.
```

### Tarea 2.2. Definir snapshots de cliente y dirección

- {% include step_label.html %} Documenta los snapshots de cliente y dirección almacenados en el pedido para conservar los datos históricos aunque el perfil cambie posteriormente.

Agrega:

```markdown
### Decisión 2. Conservar snapshots dentro de Order

Cada pedido almacenará un snapshot del nombre y correo del cliente, así
como la dirección de envío utilizada al momento de la compra.

Esto evita que un cambio posterior en el perfil del cliente modifique la
información histórica del pedido.

Patrones relacionados: PA-1 y PA-4.

Trade-off: existe duplicación controlada de información.
```

### Tarea 2.3. Referenciar categorías desde productos

- {% include step_label.html %} Explica por qué Product conserva referencias de categorías y cómo esa decisión evita duplicar documentos completos dentro de cada producto.

Agrega:

```markdown
### Decisión 3. Referenciar categorías desde Product

Cada producto almacenará un array category_ids con los identificadores de
sus categorías. También conservará category_names para mostrar nombres sin
realizar un JOIN.

Patrón relacionado: PA-3.

Trade-off: si cambia el nombre de una categoría, category_names debe
actualizarse en los productos relacionados.
```

### Tarea 2.4. Mantener Review como documento independiente

- {% include step_label.html %} Registra Review como documento independiente para permitir moderación, crecimiento y consultas sin ampliar continuamente el documento Product.

Agrega:

```markdown
### Decisión 4. Mantener Review como documento independiente

Las reseñas serán documentos separados porque pueden crecer sin límite,
se consultan de forma paginada y tienen su propio ciclo de vida de
moderación.

Patrón relacionado: PA-5.

Trade-off: obtener reseñas requiere una consulta por product_id.
```

### Tarea 2.5. Registrar los esquemas lógicos

- {% include step_label.html %} Incorpora los esquemas JSON lógicos de Customer, Product, Category, Order y Review, verificando claves, tipos y estructuras anidadas.

Agrega:

```markdown
## Esquemas lógicos

### Customer
- type
- customer_id
- name
- email
- phone
- addresses[]
- created_at
- status

### Product
- type
- product_id
- sku
- name
- description
- price
- currency
- stock
- category_ids[]
- category_names[]
- attributes{}
- active

### Order
- type
- order_id
- customer_id
- customer_snapshot{}
- shipping_address{}
- items[]
- subtotal
- tax
- total
- currency
- status
- payment_method
- created_at
- updated_at

### Category
- type
- category_id
- name
- slug
- parent_id
- description

### Review
- type
- review_id
- product_id
- customer_id
- customer_name
- rating
- title
- body
- verified_purchase
- created_at
- status
```

### Tarea 2.6. Validar el modelo lógico

- {% include step_label.html %} Contrasta los esquemas lógicos con las decisiones de embedding, referencing y snapshots para detectar inconsistencias antes del diseño físico.

Confirma:

- `OrderItem` está dentro de `Order`.
- El pedido conserva snapshots históricos.
- Las categorías se referencian mediante IDs.
- Las reseñas permanecen separadas.
- Cada decisión se relaciona con al menos un patrón de acceso.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

{% include support-prompt.html task="tarea2" %}

---

## 🗃️ Tarea 3. Diseñar la estructura física, keys e índices

En esta tarea convertirás el modelo lógico en una estructura concreta de Couchbase.

### Tarea 3.1. Definir bucket, scope y collections

- {% include step_label.html %} Registra el bucket `ecommerce`, el scope `store` y sus collections, explicando cómo separan las entidades sin crear buckets innecesarios.

Agrega:

```markdown
## Modelo físico

Bucket: ecommerce

Scope: store

Collections:
- customers
- products
- orders
- categories
- reviews
```

### Tarea 3.2. Justificar el scope

- {% include step_label.html %} Justifica el scope `store` como límite lógico del dominio comercial y documenta cómo facilita permisos, consultas y evolución del modelo.

Agrega:

```markdown
### Justificación del scope

Se utiliza un único scope llamado store porque las collections pertenecen
al mismo dominio de negocio y participan en consultas relacionadas.

El scope permite agrupar recursos, facilitar permisos y mantener un
aislamiento lógico claro.
```

### Tarea 3.3. Definir las document keys

- {% include step_label.html %} Define el patrón de document keys para cada entidad y confirma que sea estable, legible y adecuado para accesos directos mediante `USE KEYS`.

Agrega:

```markdown
## Document keys

| Collection | Patrón | Ejemplo |
|---|---|---|
| customers | cust::{customer_id} | cust::CUST-0001 |
| products | prod::{product_id} | prod::PROD-001 |
| orders | ord::{order_id} | ord::ORD-20260715-8821 |
| categories | cat::{category_id} | cat::CAT-001 |
| reviews | rev::{review_id} | rev::REV-001 |
```

Agrega:

```markdown
Las keys utilizan prefijos legibles para identificar el tipo de documento,
evitar colisiones y facilitar el acceso directo. El orden cronológico de
pedidos se resolverá mediante created_at y no mediante la document key.
```

### Tarea 3.4. Definir los índices

- {% include step_label.html %} Documenta los índices necesarios para pedidos, productos y reseñas, relacionando el orden de sus claves con cada patrón de acceso.

Agrega:

```markdown
## Índices requeridos

| Patrón | Índice |
|---|---|
| PA-1 | Acceso directo mediante USE KEYS. |
| PA-2 | idx_lab7_orders_customer sobre customer_id y created_at DESC. |
| PA-3 | idx_lab7_products_category con índice de array sobre category_ids. |
| PA-4 | Acceso directo mediante USE KEYS. |
| PA-5 | idx_lab7_reviews_product sobre product_id, status y created_at DESC. |
```

### Tarea 3.5. Validar el modelo físico

- {% include step_label.html %} Revisa bucket, scope, collections, document keys e índices para confirmar que el modelo físico implemente las decisiones lógicas anteriores.

Confirma:

- Cada patrón de acceso tiene un índice o acceso por key.
- Las keys son únicas y legibles.
- Los índices utilizan el prefijo `idx_lab7_`.
- El modelo físico tiene cinco collections, no seis.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

{% include support-prompt.html task="tarea3" %}

---

## 🛠️ Tarea 4. Crear la estructura e insertar documentos

En esta tarea materializarás el diseño en Couchbase e insertarás nueve documentos representativos.

### Tarea 4.1. Crear el bucket ecommerce

- {% include step_label.html %} Solicita `/pools/default/buckets/ecommerce` y revisa su código HTTP, estado o campos JSON antes de continuar; confirma el resultado correspondiente antes de continuar con la siguiente acción.

  ```bash
  if curl -fsS \
    -u 'Administrator:Password123!' \
    http://localhost:8091/pools/default/buckets/ecommerce \
    >/dev/null; then

    echo "El bucket ecommerce ya existe."

  else
    echo "El bucket ecommerce no existe. Creándolo..."

    curl -fsS \
      -u 'Administrator:Password123!' \
      -X POST \
      http://localhost:8091/pools/default/buckets \
      -d name=ecommerce \
      -d bucketType=couchbase \
      -d ramQuota=256 \
      -d replicaNumber=0

    echo
    echo "Bucket ecommerce creado correctamente."
  fi
  ```

- {% include step_label.html %} Espera cinco segundos después de crear el bucket para que Couchbase complete su inicialización antes de consultar el endpoint de validación.

  ```bash
  sleep 5
  ```

- {% include step_label.html %} Verifica desde Git Bash para solicitar `/pools/default/buckets/ecommerce` y revisa su código HTTP, estado o campos JSON antes de continuar.

  ```bash
  curl -s -u Administrator:Password123! http://localhost:8091/pools/default/buckets/ecommerce | python -m json.tool | grep -E '"name"|"bucketType"'
  ```

### Tarea 4.2. Crear scope y collections

- {% include step_label.html %} Abre `http://localhost:8091` en el navegador, inicia sesión como `Administrator` y espera que la Web Console cargue el clúster local.
- {% include step_label.html %} Selecciona **Query** en la navegación lateral de la Web Console para abrir Query Workbench y ejecutar las sentencias SQL++ de la práctica.
- {% include step_label.html %} Crea el scope indicado de manera idempotente y preparar el espacio lógico requerido por los ejercicios posteriores.

  ```sql
  CREATE SCOPE IF NOT EXISTS ecommerce.store;
  ```

- {% include step_label.html %} Crea las collections indicadas sin duplicarlas y preparar los destinos de datos utilizados en la práctica; conserva el bloque completo y revisa la respuesta antes de continuar.

  ```sql
  CREATE COLLECTION IF NOT EXISTS ecommerce.store.customers;
  CREATE COLLECTION IF NOT EXISTS ecommerce.store.products;
  CREATE COLLECTION IF NOT EXISTS ecommerce.store.orders;
  CREATE COLLECTION IF NOT EXISTS ecommerce.store.categories;
  CREATE COLLECTION IF NOT EXISTS ecommerce.store.reviews;
  ```

- {% include step_label.html %} Verifica desde Query Workbench para consultar `system` con el filtro ``bucket` = "ecommerce" AND `scope` = "store"` y comprueba las filas y el orden obtenidos.

  ```sql
  SELECT RAW name
  FROM system:keyspaces
  WHERE `bucket` = "ecommerce"
    AND `scope` = "store"
  ORDER BY name;
  ```

### Tarea 4.3. Crear índices primarios de desarrollo

- {% include step_label.html %} Crea el índice `idx_lab7_customers_primary` para el patrón de consulta analizado; confirma el resultado correspondiente antes de continuar con la siguiente acción.

```sql
CREATE PRIMARY INDEX IF NOT EXISTS idx_lab7_customers_primary
ON ecommerce.store.customers;

CREATE PRIMARY INDEX IF NOT EXISTS idx_lab7_products_primary
ON ecommerce.store.products;

CREATE PRIMARY INDEX IF NOT EXISTS idx_lab7_orders_primary
ON ecommerce.store.orders;

CREATE PRIMARY INDEX IF NOT EXISTS idx_lab7_categories_primary
ON ecommerce.store.categories;

CREATE PRIMARY INDEX IF NOT EXISTS idx_lab7_reviews_primary
ON ecommerce.store.reviews;
```

### Tarea 4.4. Crear índices secundarios

- {% include step_label.html %} Crea el índice `idx_lab7_orders_customer` para el patrón de consulta analizado; confirma el resultado correspondiente antes de continuar con la siguiente acción.

```sql
CREATE INDEX IF NOT EXISTS idx_lab7_orders_customer
ON ecommerce.store.orders(customer_id, created_at DESC);
```

- {% include step_label.html %} Crea el índice `idx_lab7_products_category` para el patrón de consulta analizado; conserva el bloque completo y revisa la respuesta antes de continuar.

```sql
CREATE INDEX IF NOT EXISTS idx_lab7_products_category
ON ecommerce.store.products(
  ALL ARRAY cat FOR cat IN category_ids END,
  price,
  stock
)
WHERE active = true;
```

- {% include step_label.html %} Crea el índice `idx_lab7_products_id` para el patrón de consulta analizado; conserva el bloque completo y revisa la respuesta antes de continuar.

```sql
CREATE INDEX IF NOT EXISTS idx_lab7_products_id
ON ecommerce.store.products(product_id);
```

- {% include step_label.html %} Crea el índice `idx_lab7_reviews_product` para el patrón de consulta analizado; conserva el bloque completo y revisa la respuesta antes de continuar.

```sql
CREATE INDEX IF NOT EXISTS idx_lab7_reviews_product
ON ecommerce.store.reviews(
  product_id,
  status,
  created_at DESC,
  rating
);
```

### Tarea 4.5. Insertar categorías

- {% include step_label.html %} Inserta los documentos de ejemplo en `ecommerce.store.categories` y deja esos datos disponibles para su validación.

```sql
UPSERT INTO ecommerce.store.categories (KEY, VALUE)
VALUES
("cat::CAT-001", {
  "type": "category",
  "category_id": "CAT-001",
  "name": "Electrónica",
  "slug": "electronica",
  "parent_id": null,
  "description": "Dispositivos electrónicos y accesorios"
}),
("cat::CAT-003", {
  "type": "category",
  "category_id": "CAT-003",
  "name": "Periféricos",
  "slug": "perifericos",
  "parent_id": "CAT-001",
  "description": "Teclados, ratones y otros periféricos"
});
```

### Tarea 4.6. Insertar productos

- {% include step_label.html %} Inserta los documentos de ejemplo en `ecommerce.store.products` y deja esos datos disponibles para su validación.

```sql
UPSERT INTO ecommerce.store.products (KEY, VALUE)
VALUES
("prod::PROD-001", {
  "type": "product",
  "product_id": "PROD-001",
  "sku": "TEC-MEC-001",
  "name": "Teclado Mecánico RGB",
  "description": "Teclado mecánico con retroiluminación RGB",
  "price": 1200.00,
  "currency": "MXN",
  "stock": 45,
  "category_ids": ["CAT-001", "CAT-003"],
  "category_names": ["Electrónica", "Periféricos"],
  "attributes": {
    "brand": "KeyMaster",
    "connectivity": "USB-C"
  },
  "created_at": "2026-07-10T08:00:00Z",
  "active": true
}),
("prod::PROD-002", {
  "type": "product",
  "product_id": "PROD-002",
  "sku": "MOU-INL-002",
  "name": "Mouse Inalámbrico Ergonómico",
  "description": "Mouse inalámbrico con diseño ergonómico",
  "price": 350.00,
  "currency": "MXN",
  "stock": 120,
  "category_ids": ["CAT-001", "CAT-003"],
  "category_names": ["Electrónica", "Periféricos"],
  "attributes": {
    "brand": "ErgoTech",
    "connectivity": "USB-A receiver"
  },
  "created_at": "2026-07-11T09:00:00Z",
  "active": true
});
```

### Tarea 4.7. Insertar cliente

- {% include step_label.html %} Inserta los documentos de ejemplo en `ecommerce.store.customers` y deja esos datos disponibles para su validación.

```sql
UPSERT INTO ecommerce.store.customers (KEY, VALUE)
VALUES
("cust::CUST-0001", {
  "type": "customer",
  "customer_id": "CUST-0001",
  "name": "María González",
  "email": "maria.gonzalez@ejemplo.com",
  "phone": "+52-55-1234-5678",
  "addresses": [{
    "label": "casa",
    "street": "Av. Insurgentes Sur 1500",
    "city": "Ciudad de México",
    "state": "CDMX",
    "zip": "03810",
    "country": "MX"
  }],
  "created_at": "2026-07-12T09:00:00Z",
  "status": "active"
});
```

### Tarea 4.8. Insertar pedidos

- {% include step_label.html %} Inserta los documentos de ejemplo en `ecommerce.store.orders` y deja esos datos disponibles para su validación.

```sql
UPSERT INTO ecommerce.store.orders (KEY, VALUE)
VALUES
("ord::ORD-20260715-8821", {
  "type": "order",
  "order_id": "ORD-20260715-8821",
  "customer_id": "CUST-0001",
  "customer_snapshot": {
    "name": "María González",
    "email": "maria.gonzalez@ejemplo.com"
  },
  "shipping_address": {
    "street": "Av. Insurgentes Sur 1500",
    "city": "Ciudad de México",
    "state": "CDMX",
    "zip": "03810",
    "country": "MX"
  },
  "items": [
    {
      "product_id": "PROD-001",
      "sku": "TEC-MEC-001",
      "name": "Teclado Mecánico RGB",
      "quantity": 1,
      "unit_price": 1200.00,
      "subtotal": 1200.00
    },
    {
      "product_id": "PROD-002",
      "sku": "MOU-INL-002",
      "name": "Mouse Inalámbrico Ergonómico",
      "quantity": 2,
      "unit_price": 350.00,
      "subtotal": 700.00
    }
  ],
  "subtotal": 1900.00,
  "tax": 304.00,
  "total": 2204.00,
  "currency": "MXN",
  "status": "shipped",
  "payment_method": "credit_card",
  "created_at": "2026-07-15T10:22:00Z",
  "updated_at": "2026-07-15T14:35:00Z"
}),
("ord::ORD-20260716-9102", {
  "type": "order",
  "order_id": "ORD-20260716-9102",
  "customer_id": "CUST-0001",
  "customer_snapshot": {
    "name": "María González",
    "email": "maria.gonzalez@ejemplo.com"
  },
  "shipping_address": {
    "street": "Av. Insurgentes Sur 1500",
    "city": "Ciudad de México",
    "state": "CDMX",
    "zip": "03810",
    "country": "MX"
  },
  "items": [{
    "product_id": "PROD-002",
    "sku": "MOU-INL-002",
    "name": "Mouse Inalámbrico Ergonómico",
    "quantity": 1,
    "unit_price": 350.00,
    "subtotal": 350.00
  }],
  "subtotal": 350.00,
  "tax": 56.00,
  "total": 406.00,
  "currency": "MXN",
  "status": "pending",
  "payment_method": "bank_transfer",
  "created_at": "2026-07-16T11:10:00Z",
  "updated_at": "2026-07-16T11:10:00Z"
});
```

### Tarea 4.9. Insertar reseñas

- {% include step_label.html %} Inserta los documentos de ejemplo en `ecommerce.store.reviews` y deja esos datos disponibles para su validación.

```sql
UPSERT INTO ecommerce.store.reviews (KEY, VALUE)
VALUES
("rev::REV-001", {
  "type": "review",
  "review_id": "REV-001",
  "product_id": "PROD-001",
  "customer_id": "CUST-0001",
  "customer_name": "María González",
  "rating": 5,
  "title": "Excelente teclado",
  "body": "Los switches son precisos y la iluminación es clara.",
  "verified_purchase": true,
  "created_at": "2026-07-15T16:00:00Z",
  "status": "approved"
}),
("rev::REV-002", {
  "type": "review",
  "review_id": "REV-002",
  "product_id": "PROD-001",
  "customer_id": "CUST-0001",
  "customer_name": "María González",
  "rating": 4,
  "title": "Buen producto",
  "body": "Funciona correctamente y la construcción es sólida.",
  "verified_purchase": true,
  "created_at": "2026-07-16T10:30:00Z",
  "status": "approved"
});
```

### Tarea 4.10. Validar conteos

- {% include step_label.html %} Consulta `ecommerce.store.customers` y proyectar `(SELECT RAW COUNT(*)` y comprueba las filas y el orden obtenidos.

```sql
SELECT
  (SELECT RAW COUNT(*) FROM ecommerce.store.customers)[0] AS total_customers,
  (SELECT RAW COUNT(*) FROM ecommerce.store.products)[0] AS total_products,
  (SELECT RAW COUNT(*) FROM ecommerce.store.orders)[0] AS total_orders,
  (SELECT RAW COUNT(*) FROM ecommerce.store.categories)[0] AS total_categories,
  (SELECT RAW COUNT(*) FROM ecommerce.store.reviews)[0] AS total_reviews;
```

**Resultado esperado:**

Para validar `Validar conteos`, verifica la referencia siguiente y confirma que la respuesta permita consultar `ecommerce.store.customers` y proyectar `(SELECT RAW COUNT(*)` y comprobar las filas y el orden obtenidos; detente si aparece un error.

```json
[
  {
    "total_customers": 1,
    "total_products": 2,
    "total_orders": 2,
    "total_categories": 2,
    "total_reviews": 2
  }
]
```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

{% include support-prompt.html task="tarea4" %}

---

## ✅ Tarea 5. Validar los patrones de acceso

En esta tarea ejecutarás una consulta por cada patrón de acceso y comprobarás que el modelo responde como fue diseñado.

### Tarea 5.1. PA-1: obtener pedido por document key

- {% include step_label.html %} Consulta `ecommerce.store.orders` y proyectar `META(o).id AS documentKey, o.*` y comprueba las filas y el orden obtenidos.

```sql
SELECT META(o).id AS documentKey,
       o.*
FROM ecommerce.store.orders AS o
USE KEYS "ord::ORD-20260715-8821";
```

**Validación:**

Para validar `PA-1: obtener pedido por document key`, verifica la referencia siguiente y confirma que la respuesta permita consultar `ecommerce.store.orders` y proyectar `META(o).id AS documentKey, o.*` y comprobar las filas y el orden obtenidos; detente si aparece un error.

Debes obtener en una sola lectura los datos del pedido, el snapshot del cliente, la dirección de envío y los dos ítems embebidos.

### Tarea 5.2. PA-2: listar pedidos por cliente

- {% include step_label.html %} Consulta `ecommerce.store.orders` con el filtro `customer_id = "CUST-0001"` y comprueba las filas y el orden obtenidos.

```sql
SELECT order_id,
       status,
       total,
       currency,
       created_at,
       ARRAY_LENGTH(items) AS item_count
FROM ecommerce.store.orders
WHERE customer_id = "CUST-0001"
ORDER BY created_at DESC;
```

**Validación:**

Para validar `PA-2: listar pedidos por cliente`, verifica la referencia siguiente y confirma que la respuesta permita consultar `ecommerce.store.orders` con el filtro `customer_id = "CUST-0001"` y comprobar las filas y el orden obtenidos; detente si aparece un error.

Debes obtener dos pedidos y el más reciente debe aparecer primero.

### Tarea 5.3. PA-3: buscar productos por categoría

- {% include step_label.html %} Consulta `ecommerce.store.products` y proyectar `product_id, name, price, stock, category_names` y comprueba las filas y el orden obtenidos.

```sql
SELECT product_id,
       name,
       price,
       stock,
       category_names
FROM ecommerce.store.products
WHERE ANY cat IN category_ids
      SATISFIES cat = "CAT-003"
      END
  AND active = true
ORDER BY price ASC;
```

**Validación:**

Para validar `PA-3: buscar productos por categoría`, verifica la referencia siguiente y confirma que la respuesta permita consultar `ecommerce.store.products` y proyectar `product_id, name, price, stock, category_names` y comprobar las filas y el orden obtenidos; detente si aparece un error.

Debes obtener `PROD-002` primero y `PROD-001` después.

### Tarea 5.4. Verificar el índice de array

- {% include step_label.html %} Obtén el plan de ejecución e identifica el índice, los spans y los operadores seleccionados por Query; confirma el resultado correspondiente antes de continuar con la siguiente acción.

```sql
EXPLAIN
SELECT product_id,
       name,
       price,
       stock
FROM ecommerce.store.products
WHERE ANY cat IN category_ids
      SATISFIES cat = "CAT-003"
      END
  AND active = true
ORDER BY price ASC;
```

Busca:

```text
idx_lab7_products_category
```

### Tarea 5.5. PA-4: actualizar el estado por key

- {% include step_label.html %} Actualiza los documentos de `ecommerce.store.orders` seleccionados por la clave o condición y verifica los campos modificados.

```sql
UPDATE ecommerce.store.orders
USE KEYS "ord::ORD-20260716-9102"
SET status = "processing",
    updated_at = NOW_STR()
RETURNING order_id, status, updated_at;
```

### Tarea 5.6. PA-5: listar reseñas aprobadas

- {% include step_label.html %} Consulta las reseñas aprobadas de `PROD-001` y verifica que cada fila incluya autor, calificación, comentario y estado correspondiente.

```sql
SELECT review_id,
       customer_name,
       rating,
       title,
       body,
       created_at
FROM ecommerce.store.reviews
WHERE product_id = "PROD-001"
  AND status = "approved"
ORDER BY created_at DESC;
```

### Tarea 5.7. Calcular el promedio de reseñas

- {% include step_label.html %} Calcula el promedio y la cantidad de reseñas aprobadas para `PROD-001`, comprobando que la agregación devuelva una sola fila.

```sql
SELECT COUNT(*) AS total_reviews,
       AVG(rating) AS avg_rating
FROM ecommerce.store.reviews
WHERE product_id = "PROD-001"
  AND status = "approved";
```

**Resultado esperado:**

Para validar `Calcular el promedio de reseñas`, verifica la referencia siguiente y confirma que la respuesta permita consultar `ecommerce.store.reviews` con el filtro `product_id = "PROD-001" AND status = "approved"` y comprobar las filas y el orden obtenidos; detente si aparece un error.

```json
[
  {
    "total_reviews": 2,
    "avg_rating": 4.5
  }
]
```

### Tarea 5.8. Validar referencias con UNNEST y JOIN

- {% include step_label.html %} Consulta `ecommerce.store.orders` y comprueba las filas y el orden obtenidos; confirma el resultado correspondiente antes de continuar con la siguiente acción.

```sql
SELECT o.order_id,
       o.customer_snapshot.name AS customer,
       item.product_id,
       item.name AS ordered_product,
       p.stock AS current_stock
FROM ecommerce.store.orders AS o
UNNEST o.items AS item
JOIN ecommerce.store.products AS p
  ON p.product_id = item.product_id
ORDER BY o.created_at DESC;
```

### Tarea 5.9. Verificar índices

- {% include step_label.html %} Consulta `system:indexes` y revisa los índices cuyo nombre coincide con `idx_lab7_%`; confirma el resultado correspondiente antes de continuar con la siguiente acción.

```sql
SELECT name,
       bucket_id,
       scope_id,
       keyspace_id,
       state
FROM system:indexes
WHERE bucket_id = "ecommerce"
  AND scope_id = "store"
  AND name LIKE "idx_lab7_%"
ORDER BY keyspace_id, name;
```

**Resultado esperado:**

Para validar `Verificar índices`, verifica la referencia siguiente y confirma que la respuesta permita consultar `system:indexes` y revisar los índices cuyo nombre coincide con `idx_lab7_%`; detente si aparece un error.

Todos los índices deben aparecer con:

```text
state = online
```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

{% include support-prompt.html task="tarea5" %}

---

## 🛠️ Resolución de problemas

### Problema 1. El bucket ecommerce ya existe

Esto no representa un error. La práctica utiliza `UPSERT` e instrucciones idempotentes para permitir la repetición.

```bash
curl -s -u Administrator:Password123!   http://localhost:8091/pools/default/buckets/ecommerce   | python -m json.tool | grep '"name"'
```

### Problema 2. CREATE SCOPE o CREATE COLLECTION muestra que el recurso existe

Verifica que usaste `IF NOT EXISTS`. Después consulta:

Consulta `system` con el filtro ``bucket` = "ecommerce" AND `scope` = "store"` y comprueba las filas y el orden obtenidos; conserva el bloque completo y revisa la respuesta antes de continuar.

```sql
SELECT RAW name
FROM system:keyspaces
WHERE `bucket` = "ecommerce"
  AND `scope` = "store"
ORDER BY name;
```

### Problema 3. Una consulta muestra No index available

inventariar en `system:indexes` los índices del bucket, scope y collections especificados por los filtros; conserva el bloque completo y revisa la respuesta antes de continuar.

```sql
SELECT name, keyspace_id, state
FROM system:indexes
WHERE bucket_id = "ecommerce"
  AND scope_id = "store"
ORDER BY keyspace_id, name;
```

Todos los índices deben estar `online`.

### Problema 4. El índice de array no se utiliza

Asegúrate de que la consulta incluya tanto `ANY ... SATISFIES` como `active = true`, ya que el índice es parcial.

### Problema 5. El JOIN no devuelve resultados

Verifica que `item.product_id` coincida con `products.product_id`:

Consulta `ecommerce.store.products` y proyectar `META().id, product_id` y comprueba las filas y el orden obtenidos; conserva el bloque completo y revisa la respuesta antes de continuar.

```sql
SELECT META().id, product_id
FROM ecommerce.store.products;
```

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

Si continúa apareciendo Community Edition, vuelve a la Práctica 2 y recrea el contenedor con Enterprise Edition antes de ejecutar esta práctica.

### Problema 7. python -m json.tool falla

aplicar el bloque que comienza con `python --version` y revisar su efecto específico antes de continuar; conserva el bloque completo y revisa la respuesta antes de continuar.

```bash
python --version
python3 --version
```

Si `python3` funciona, reemplaza `python -m json.tool` por `python3 -m json.tool`.
