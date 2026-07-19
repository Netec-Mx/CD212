---
layout: lab
title: "Práctica 9: Configuración de roles y pruebas de acceso en Couchbase"
permalink: /lab9/lab9/
images_base: /labs/lab9/img
duration: "55 minutos"
objective:
  - Preparar el directorio de trabajo de la práctica 9 y validar que Couchbase Server Enterprise Edition esté disponible.
  - Reutilizar el bucket ecommerce y crear scopes, collections, documentos e índices para un escenario de seguridad.
  - Crear un usuario de solo lectura con permisos directos sobre el scope catalog.
  - Crear un usuario backend con permisos específicos a nivel de collection.
  - Crear un grupo de reporting y comprobar la herencia de permisos en un usuario sin roles directos.
  - Ejecutar pruebas positivas y negativas mediante la Query REST API para validar el principio de mínimo privilegio.
prerequisites:
  - Haber completado las prácticas 7 y 8.
  - Tener disponible el bucket ecommerce.
  - Tener Docker Desktop en ejecución.
  - Tener activo el contenedor couchbase-lab creado con la imagen couchbase/server:enterprise-7.6.2.
  - Tener acceso a la Web Console en http://localhost:8091.
  - Utilizar Visual Studio Code con Git Bash como terminal integrada.
  - Conocer la diferencia entre autenticación, autorización, usuario, grupo y rol.
  - Comprender operaciones SQL++ SELECT, UPSERT, UPDATE y DELETE.
introduction:
  - En esta práctica utilizarás Couchbase Server Enterprise Edition para implementar Control de Acceso Basado en Roles, conocido como RBAC, sobre un escenario de comercio electrónico. Crearás límites de autorización mediante los scopes catalog y sales, configurarás usuarios con permisos diferentes y comprobarás qué operaciones están permitidas o denegadas. También utilizarás un grupo para centralizar permisos de reporting y ejecutarás una matriz de pruebas desde Git Bash mediante la Query REST API.
slug: lab9
lab_number: 9
final_result: >
  Al finalizar la práctica habrás configurado usuarios y grupos con permisos directos y heredados, probado accesos permitidos y denegados, y demostrado el principio de mínimo privilegio a nivel de scope y collection. Podrás distinguir entre roles de acceso Key-Value y roles de SQL++, interpretar respuestas de autorización y validar una configuración RBAC mediante pruebas automatizadas.
notes:
  - Todos los comandos de terminal deben ejecutarse desde Git Bash dentro de Visual Studio Code.
  - Utiliza las credenciales Administrator y Password123! configuradas en las prácticas anteriores.
  - Esta práctica reutiliza el bucket ecommerce y no modifica ni elimina los scopes store ni sus datos.
  - Los scopes catalog y sales se crean específicamente como límites de autorización para esta práctica.
  - Todos los índices creados utilizan el prefijo idx_lab9_.
  - Se utiliza UPSERT e instrucciones IF NOT EXISTS para que la práctica pueda repetirse.
  - Las contraseñas utilizadas son exclusivamente educativas y no deben reutilizarse fuera del laboratorio.
  - No elimines usuarios, grupos, scopes, collections ni el bucket ecommerce al finalizar.
references: []
prev: /lab8/lab8/
next: /lab10/lab10/
---

---
<!-- Aquí comienzan las instrucciones paso a paso de la práctica -->

## 📁 Preparación del directorio de trabajo

En esta práctica conservarás el directorio raíz del curso y crearás únicamente el subdirectorio correspondiente a `lab9`.

### 🗂️ Crear y abrir el subdirectorio de la práctica

- {% include step_label.html %} Abre **Docker Desktop** y confirma que el motor esté activo, porque `couchbase-lab` depende del daemon local para ofrecer los servicios del clúster.
- {% include step_label.html %} Abre **Visual Studio Code** y espera su carga completa, ya que administrarás archivos, comandos y evidencias desde esta aplicación.
- {% include step_label.html %} Selecciona **File → Open Folder** en VS Code y abre `C:\LABS\couchbase-nosql` para mantener los archivos dentro de la estructura del curso.

  ```text
  C:\LABS\couchbase-nosql
  ```

- {% include step_label.html %} Selecciona **Terminal → New Terminal** en VS Code para abrir la consola integrada desde la cual ejecutarás las operaciones de seguridad.
- {% include step_label.html %} Comprueba en el selector del panel Terminal que **Git Bash** sea el perfil activo, porque los comandos utilizan sintaxis y rutas Bash.
- {% include step_label.html %} Crea `/c/LABS/couchbase-nosql/lab9` mediante `mkdir -p` para disponer del subdirectorio sin generar errores si ya existe.

  ```bash
  mkdir -p /c/LABS/couchbase-nosql/lab9
  ```

- {% include step_label.html %} Cambia la ubicación activa de Git Bash a `/c/LABS/couchbase-nosql/lab9` para conservar en esta carpeta los archivos de validación.

  ```bash
  cd /c/LABS/couchbase-nosql/lab9
  ```

- {% include step_label.html %} Consulta la ruta mediante `pwd` y confirma que termine en `/lab9`; corrige la ubicación antes de continuar si aparece otro directorio.

  ```bash
  pwd
  ```

**Salida esperada:**

Para validar `crear y abrir el subdirectorio de trabajo`, verifica la referencia siguiente y confirma que la respuesta permita mostrar la ruta activa y confirmar que Git Bash está ubicado en el subdirectorio asignado a esta práctica; detente si aparece un error.

```text
/c/LABS/couchbase-nosql/lab9
```

---

## 🛡️ Tarea 1. Preparar el entorno de seguridad

En esta tarea verificarás Couchbase Server, definirás variables de entorno y crearás los scopes, collections, datos e índices que se utilizarán para probar RBAC.

### Tarea 1.1. Verificar Couchbase Server

- {% include step_label.html %} Consulta `couchbase-lab` mediante `docker ps` y confirma que el contenedor permanezca activo antes de configurar usuarios o ejecutar consultas.

  {%raw%}
  ```bash
  docker ps --filter "name=couchbase-lab" --format "table {{.Names}}\t{{.Status}}"
  ```
  {%endraw%}

- {% include step_label.html %} Inicia `couchbase-lab` con `docker start` solamente si está detenido y espera la confirmación de Docker antes de validar los servicios.

  ```bash
  docker start couchbase-lab
  ```

- {% include step_label.html %} Solicita la Web Console mediante `curl` y confirma el código HTTP 200, que demuestra la disponibilidad administrativa en el puerto 8091.

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

- {% include step_label.html %} Inspecciona `couchbase-lab` desde Git Bash y confirma que utiliza `couchbase/server:enterprise-7.6.2` como imagen activa.

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

> **IMPORTANTE:** Si aparece `couchbase/server:community-7.6.2`, el contenedor anterior de Community Edition continúa activo. Vuelve a la Práctica 2 y recrea el entorno con Enterprise Edition antes de continuar, ya que esta práctica utiliza RBAC con usuarios, grupos y permisos granulares.
{: .lab-note .important .compact}

### Tarea 1.3. Definir variables de entorno

Ejecuta:

- {% include step_label.html %} Aplica el bloque que comienza con `export CB_HOST="localhost"` y revisa su efecto específico antes de continuar; conserva el bloque completo y revisa la respuesta antes de continuar.

```bash
export CB_HOST="localhost"
export CB_ADMIN="Administrator"
export CB_PASS="Password123!"
export CB_URL="http://${CB_HOST}:8091"
export CB_QUERY_URL="http://${CB_HOST}:8093/query/service"

export READONLY_USER="app-readonly"
export READONLY_PASS="Readonly@2026!"

export BACKEND_USER="app-backend"
export BACKEND_PASS="Backend@2026!"

export REPORTER_USER="reporter-user"
export REPORTER_PASS="Reporter@2026!"
```

- {% include step_label.html %} Muestra las variables de URL y usuario administrativo, sin imprimir contraseñas, para comprobar que las solicitudes REST usarán el destino correcto.

  ```bash
  echo "CB_URL=${CB_URL}"
  echo "CB_QUERY_URL=${CB_QUERY_URL}"
  echo "READONLY_USER=${READONLY_USER}"
  echo "BACKEND_USER=${BACKEND_USER}"
  echo "REPORTER_USER=${REPORTER_USER}"
  ```

### Tarea 1.4. Confirmar que ecommerce existe

- {% include step_label.html %} Consulta mediante REST el bucket `ecommerce` y confirma que responde correctamente antes de crear scopes, collections, usuarios o permisos.

```bash
curl -s -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_URL}/pools/default/buckets/ecommerce" \
  | python -m json.tool | grep -E '"name"|"bucketType"'
```

**Validación:**

Para validar `Confirmar que ecommerce existe`, verifica la referencia siguiente y confirma que la respuesta permita solicitar `el endpoint REST indicado` y revisar su código HTTP, estado o campos JSON antes de continuar; detente si aparece un error.

Debe aparecer:

```text
"name": "ecommerce"
```

### Tarea 1.5. Crear scopes y collections

- {% include step_label.html %} Abre `http://localhost:8091` en el navegador y espera la pantalla de autenticación para acceder a la Web Console del nodo local.
- {% include step_label.html %} Abre `http://localhost:8091` en el navegador e inicia sesión como `Administrator` para preparar scopes, collections y datos de prueba.
- {% include step_label.html %} Selecciona **Query** en la navegación lateral para abrir Query Workbench y ejecutar las sentencias SQL++ de preparación.
- {% include step_label.html %} Crea los scopes y collections indicados desde Query Workbench, conservando sus nombres para que los roles RBAC apunten al keyspace correcto.

  ```sql
  CREATE SCOPE IF NOT EXISTS ecommerce.catalog;
  CREATE SCOPE IF NOT EXISTS ecommerce.sales;
  ```

- {% include step_label.html %} Crea las collections indicadas sin duplicarlas y preparar los destinos de datos utilizados en la práctica; conserva el bloque completo y revisa la respuesta antes de continuar.

  ```sql
  CREATE COLLECTION IF NOT EXISTS ecommerce.catalog.products;
  CREATE COLLECTION IF NOT EXISTS ecommerce.catalog.categories;
  CREATE COLLECTION IF NOT EXISTS ecommerce.sales.customers;
  CREATE COLLECTION IF NOT EXISTS ecommerce.sales.purchases;
  ```

### Tarea 1.6. Crear índices

- {% include step_label.html %} Crea los índices requeridos sobre las collections de seguridad para que las consultas de prueba no fallen por ausencia de acceso GSI.

```sql
CREATE PRIMARY INDEX IF NOT EXISTS idx_lab9_products_primary
ON ecommerce.catalog.products;

CREATE PRIMARY INDEX IF NOT EXISTS idx_lab9_categories_primary
ON ecommerce.catalog.categories;

CREATE PRIMARY INDEX IF NOT EXISTS idx_lab9_customers_primary
ON ecommerce.sales.customers;

CREATE PRIMARY INDEX IF NOT EXISTS idx_lab9_purchases_primary
ON ecommerce.sales.purchases;
```

- {% include step_label.html %} Crea el índice `idx_lab9_products_price` para el patrón de consulta analizado; conserva el bloque completo y revisa la respuesta antes de continuar.

```sql
CREATE INDEX IF NOT EXISTS idx_lab9_products_price
ON ecommerce.catalog.products(price, name);

CREATE INDEX IF NOT EXISTS idx_lab9_purchases_status
ON ecommerce.sales.purchases(status, customer_id, total);
```

### Tarea 1.7. Insertar datos de prueba

- {% include step_label.html %} Inserta los documentos de productos, clientes y compras utilizados para distinguir permisos de lectura, escritura, actualización y borrado.

```sql
UPSERT INTO ecommerce.catalog.products (KEY, VALUE)
VALUES
(
  "prod::RBAC-001",
  {
    "type": "product",
    "product_id": "RBAC-001",
    "name": "Laptop Pro 15",
    "price": 1299.99,
    "category": "electronics",
    "stock": 50
  }
),
(
  "prod::RBAC-002",
  {
    "type": "product",
    "product_id": "RBAC-002",
    "name": "Wireless Mouse",
    "price": 29.99,
    "category": "accessories",
    "stock": 200
  }
),
(
  "prod::RBAC-003",
  {
    "type": "product",
    "product_id": "RBAC-003",
    "name": "USB-C Hub",
    "price": 49.99,
    "category": "accessories",
    "stock": 150
  }
);
```

- {% include step_label.html %} escribir los documentos de ejemplo en `ecommerce.catalog.categories` y dejar esos datos disponibles para su validación; conserva el bloque completo y revisa la respuesta antes de continuar.

```sql
UPSERT INTO ecommerce.catalog.categories (KEY, VALUE)
VALUES
(
  "cat::RBAC-001",
  {
    "type": "category",
    "category_id": "RBAC-001",
    "name": "Electronics",
    "description": "Electronic devices and gadgets"
  }
),
(
  "cat::RBAC-002",
  {
    "type": "category",
    "category_id": "RBAC-002",
    "name": "Accessories",
    "description": "Computer and device accessories"
  }
);
```

- {% include step_label.html %} escribir los documentos de ejemplo en `ecommerce.sales.customers` y dejar esos datos disponibles para su validación; conserva el bloque completo y revisa la respuesta antes de continuar.

```sql
UPSERT INTO ecommerce.sales.customers (KEY, VALUE)
VALUES
(
  "cust::RBAC-001",
  {
    "type": "customer",
    "customer_id": "RBAC-001",
    "name": "Ana García",
    "email": "ana@example.com",
    "tier": "premium"
  }
),
(
  "cust::RBAC-002",
  {
    "type": "customer",
    "customer_id": "RBAC-002",
    "name": "Carlos López",
    "email": "carlos@example.com",
    "tier": "standard"
  }
);
```

- {% include step_label.html %} escribir los documentos de ejemplo en `ecommerce.sales.purchases` y dejar esos datos disponibles para su validación; conserva el bloque completo y revisa la respuesta antes de continuar.

```sql
UPSERT INTO ecommerce.sales.purchases (KEY, VALUE)
VALUES
(
  "ord::RBAC-001",
  {
    "type": "purchase",
    "purchase_id": "RBAC-001",
    "customer_id": "RBAC-001",
    "product_id": "RBAC-001",
    "quantity": 1,
    "total": 1299.99,
    "status": "delivered"
  }
),
(
  "ord::RBAC-002",
  {
    "type": "purchase",
    "purchase_id": "RBAC-002",
    "customer_id": "RBAC-002",
    "product_id": "RBAC-002",
    "quantity": 2,
    "total": 59.98,
    "status": "pending"
  }
);
```

### Tarea 1.8. Validar la preparación

- {% include step_label.html %} Cuenta los documentos de prueba en cada collection y confirma que el entorno contiene datos antes de evaluar los permisos RBAC.

```sql
SELECT
  (SELECT RAW COUNT(*) FROM ecommerce.catalog.products)[0]
    AS products,
  (SELECT RAW COUNT(*) FROM ecommerce.catalog.categories)[0]
    AS categories,
  (SELECT RAW COUNT(*) FROM ecommerce.sales.customers)[0]
    AS customers,
  (SELECT RAW COUNT(*) FROM ecommerce.sales.purchases)[0]
    AS purchases;
```

**Resultado esperado:**

Para validar `Validar la preparación`, verifica la referencia siguiente y confirma que la respuesta permita consultar `ecommerce.catalog.products` y proyectar `(SELECT RAW COUNT(*)` y comprobar las filas y el orden obtenidos; detente si aparece un error.

```json
[
  {
    "products": 3,
    "categories": 2,
    "customers": 2,
    "purchases": 2
  }
]
```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

{% include support-prompt.html task="tarea1" %}

---

## 👁️ Tarea 2. Crear y probar el usuario app-readonly

En esta tarea crearás un usuario con permisos directos de solo lectura sobre el scope `catalog`.

### Tarea 2.1. Crear app-readonly

- {% include step_label.html %} Crea `app-readonly` mediante la API de seguridad y asigna únicamente el rol de lectura definido para la collection de productos.



```bash
curl -s -X PUT \
  -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_URL}/settings/rbac/users/local/${READONLY_USER}" \
  -d "name=Application ReadOnly" \
  -d "password=${READONLY_PASS}" \
  -d "roles=data_reader[ecommerce:catalog],query_select[ecommerce:catalog]"
```

### Tarea 2.2. Verificar la configuración

- {% include step_label.html %} Consulta la definición de `app-readonly` mediante REST y confirma que sus roles y dominios coincidan con el alcance de solo lectura.

```bash
curl -s -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_URL}/settings/rbac/users/local/${READONLY_USER}" \
  | python -m json.tool
```

Busca:

```text
data_reader
query_select
ecommerce
catalog
```

### Tarea 2.3. Probar SELECT permitido

- {% include step_label.html %} Autentica la consulta como `app-readonly` y confirma que puede leer productos, demostrando que el permiso concedido funciona.

```bash
curl -s \
  -u "${READONLY_USER}:${READONLY_PASS}" \
  "${CB_QUERY_URL}" \
  --data-urlencode \
  'statement=SELECT name, price
             FROM ecommerce.catalog.products
             WHERE price < 100
             ORDER BY price;' \
  | python -m json.tool
```

**Resultado esperado:**

Para validar `Probar SELECT permitido`, verifica la referencia siguiente y confirma que la respuesta permita solicitar `el endpoint REST indicado` y revisar su código HTTP, estado o campos JSON antes de continuar; detente si aparece un error.

Deben aparecer:

```text
Wireless Mouse
USB-C Hub
```

### Tarea 2.4. Probar INSERT denegado

- {% include step_label.html %} Intenta insertar un producto como `app-readonly` y verifica que Query Service rechace la escritura por falta de privilegios.

```bash
curl -s \
  -u "${READONLY_USER}:${READONLY_PASS}" \
  "${CB_QUERY_URL}" \
  --data-urlencode \
  'statement=UPSERT INTO ecommerce.catalog.products (KEY, VALUE)
             VALUES ("prod::RBAC-DENIED", {"name":"Denied"});' \
  | python -m json.tool
```

**Validación:**

Para validar `Probar INSERT denegado`, verifica la referencia siguiente y confirma que la respuesta permita solicitar `el endpoint REST indicado` y revisar su código HTTP, estado o campos JSON antes de continuar; detente si aparece un error.

La respuesta debe contener un error de autorización o falta de credenciales.

### Tarea 2.5. Probar acceso denegado a sales

- {% include step_label.html %} Solicita datos del scope `sales` como `app-readonly` y confirma que la respuesta deniegue acceso fuera del catálogo autorizado.

```bash
curl -s \
  -u "${READONLY_USER}:${READONLY_PASS}" \
  "${CB_QUERY_URL}" \
  --data-urlencode \
  'statement=SELECT purchase_id, status
             FROM ecommerce.sales.purchases
             LIMIT 1;' \
  | python -m json.tool
```

**Validación:**

Para validar `Probar acceso denegado a sales`, verifica la referencia siguiente y confirma que la respuesta permita solicitar `el endpoint REST indicado` y revisar su código HTTP, estado o campos JSON antes de continuar; detente si aparece un error.

La consulta debe fallar porque el usuario solo tiene permisos sobre `catalog`.

### Tarea 2.6. Registrar la matriz de permisos

- {% include step_label.html %} Registra en el archivo de trabajo la matriz de permisos de `app-readonly`, diferenciando lectura permitida, escritura y scopes denegados.

Crea el archivo:

```text
rbac-test-matrix.md
```

Agrega:

```markdown
# Matriz de pruebas RBAC

## app-readonly

| Operación | Recurso | Resultado esperado |
|---|---|---|
| SELECT | ecommerce.catalog.products | Permitido |
| UPSERT | ecommerce.catalog.products | Denegado |
| SELECT | ecommerce.sales.purchases | Denegado |
```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

{% include support-prompt.html task="tarea2" %}

---

## ⚙️ Tarea 3. Crear y probar el usuario app-backend

En esta tarea crearás un usuario con permisos específicos a nivel de collection. El backend podrá leer productos y clientes, así como leer, insertar y actualizar compras.

### Tarea 3.1. Crear app-backend

- {% include step_label.html %} Crea `app-backend` mediante la API de seguridad y asigna los roles de lectura y escritura requeridos por el servicio de aplicación.

```bash
curl -s -X PUT \
  -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_URL}/settings/rbac/users/local/${BACKEND_USER}" \
  -d "name=Application Backend" \
  -d "password=${BACKEND_PASS}" \
  -d "roles=data_reader[ecommerce:catalog:products],query_select[ecommerce:catalog:products],data_reader[ecommerce:sales:customers],query_select[ecommerce:sales:customers],data_reader[ecommerce:sales:purchases],data_writer[ecommerce:sales:purchases],query_select[ecommerce:sales:purchases],query_insert[ecommerce:sales:purchases],query_update[ecommerce:sales:purchases]"
```

### Tarea 3.2. Verificar los roles

- {% include step_label.html %} Consulta la definición de `app-backend` y confirma que incluya acceso de lectura al catálogo y mutación limitada sobre compras.

```bash
curl -s -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_URL}/settings/rbac/users/local/${BACKEND_USER}" \
  | python -m json.tool
```

Confirma que existan roles asociados específicamente a:

```text
catalog:products
sales:customers
sales:purchases
```

### Tarea 3.3. Probar lectura de productos

- {% include step_label.html %} Autentica como `app-backend` una consulta de productos y confirma que el rol asignado permita leer la collection del catálogo.

```bash
curl -s \
  -u "${BACKEND_USER}:${BACKEND_PASS}" \
  "${CB_QUERY_URL}" \
  --data-urlencode \
  'statement=SELECT product_id, name, price
             FROM ecommerce.catalog.products
             ORDER BY price;' \
  | python -m json.tool
```

### Tarea 3.4. Probar lectura de clientes

- {% include step_label.html %} Consulta clientes como `app-backend` y verifica que el acceso de lectura al scope de ventas funcione con las credenciales nuevas.

```bash
curl -s \
  -u "${BACKEND_USER}:${BACKEND_PASS}" \
  "${CB_QUERY_URL}" \
  --data-urlencode \
  'statement=SELECT customer_id, name, tier
             FROM ecommerce.sales.customers
             ORDER BY customer_id;' \
  | python -m json.tool
```

### Tarea 3.5. Probar inserción de una compra

- {% include step_label.html %} Inserta una compra como `app-backend` y confirma que el permiso de escritura permita crear el documento en `sales.purchases`.

```bash
curl -s \
  -u "${BACKEND_USER}:${BACKEND_PASS}" \
  "${CB_QUERY_URL}" \
  --data-urlencode \
  'statement=UPSERT INTO ecommerce.sales.purchases (KEY, VALUE)
             VALUES (
               "ord::RBAC-003",
               {
                 "type":"purchase",
                 "purchase_id":"RBAC-003",
                 "customer_id":"RBAC-001",
                 "product_id":"RBAC-002",
                 "quantity":3,
                 "total":89.97,
                 "status":"processing"
               }
             );' \
  | python -m json.tool
```

**Validación:**

Para validar `Probar inserción de una compra`, verifica la referencia siguiente y confirma que la respuesta permita solicitar `el endpoint REST indicado` y revisar su código HTTP, estado o campos JSON antes de continuar; detente si aparece un error.

La respuesta debe mostrar:

```text
status = success
mutationCount = 1
```

### Tarea 3.6. Probar actualización de una compra

- {% include step_label.html %} Actualiza la compra de prueba como `app-backend` y confirma que el rol permita modificar campos del documento existente.

```bash
curl -s \
  -u "${BACKEND_USER}:${BACKEND_PASS}" \
  "${CB_QUERY_URL}" \
  --data-urlencode \
  'statement=UPDATE ecommerce.sales.purchases
             USE KEYS "ord::RBAC-003"
             SET status = "confirmed"
             RETURNING purchase_id, status;' \
  | python -m json.tool
```

### Tarea 3.7. Probar inserción denegada en products

- {% include step_label.html %} Intenta insertar un producto como `app-backend` y confirma que RBAC impida escribir en la collection de catálogo.

```bash
curl -s \
  -u "${BACKEND_USER}:${BACKEND_PASS}" \
  "${CB_QUERY_URL}" \
  --data-urlencode \
  'statement=UPSERT INTO ecommerce.catalog.products (KEY, VALUE)
             VALUES ("prod::RBAC-BLOCKED", {"name":"Blocked"});' \
  | python -m json.tool
```

### Tarea 3.8. Probar DELETE denegado

- {% include step_label.html %} Intenta eliminar la compra como `app-backend` y verifica que el permiso de actualización no conceda automáticamente privilegio de borrado.

```bash
curl -s \
  -u "${BACKEND_USER}:${BACKEND_PASS}" \
  "${CB_QUERY_URL}" \
  --data-urlencode \
  'statement=DELETE FROM ecommerce.sales.purchases
             USE KEYS "ord::RBAC-003";' \
  | python -m json.tool
```

La operación debe fallar porque no se asignó `query_delete`.

### Tarea 3.9. Actualizar la matriz

- {% include step_label.html %} Actualiza la matriz con los permisos efectivos de `app-backend`, incluidos lectura, inserción, actualización y eliminación denegada.

Agrega a `rbac-test-matrix.md`:

```markdown
## app-backend

| Operación | Recurso | Resultado esperado |
|---|---|---|
| SELECT | ecommerce.catalog.products | Permitido |
| SELECT | ecommerce.sales.customers | Permitido |
| SELECT | ecommerce.sales.purchases | Permitido |
| UPSERT | ecommerce.sales.purchases | Permitido |
| UPDATE | ecommerce.sales.purchases | Permitido |
| DELETE | ecommerce.sales.purchases | Denegado |
| UPSERT | ecommerce.catalog.products | Denegado |
```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

{% include support-prompt.html task="tarea3" %}

---

## 👥 Tarea 4. Crear un grupo y comprobar la herencia de permisos

En esta tarea crearás el grupo `reporting-team`, le asignarás permisos de lectura y crearás un usuario que recibirá los permisos exclusivamente por herencia.

### Tarea 4.1. Crear el grupo reporting-team

- {% include step_label.html %} Crea `reporting-team` mediante la API de seguridad y asigna al grupo los roles de lectura necesarios para generar reportes.

```bash
curl -s -X PUT \
  -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_URL}/settings/rbac/groups/reporting-team" \
  -d "description=Reporting team with read-only catalog access" \
  -d "roles=data_reader[ecommerce:catalog],query_select[ecommerce:catalog]"
```

### Tarea 4.2. Verificar el grupo

- {% include step_label.html %} Consulta `reporting-team` mediante REST y confirma que los roles definidos pertenezcan al grupo antes de agregar usuarios.

```bash
curl -s -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_URL}/settings/rbac/groups/reporting-team" \
  | python -m json.tool
```

Confirma:

```text
reporting-team
data_reader
query_select
catalog
```

### Tarea 4.3. Crear reporter-user

- {% include step_label.html %} Crea `reporter-user` sin roles directos y asígnalo a `reporting-team` para probar la herencia de permisos desde el grupo.

```bash
curl -s -X PUT \
  -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_URL}/settings/rbac/users/local/${REPORTER_USER}" \
  -d "name=Reporter User" \
  -d "password=${REPORTER_PASS}" \
  -d "groups=reporting-team"
```

### Tarea 4.4. Verificar la pertenencia al grupo

- {% include step_label.html %} Consulta la definición de `reporter-user` y confirma que muestre pertenencia a `reporting-team` sin duplicar roles directos.

```bash
curl -s -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_URL}/settings/rbac/users/local/${REPORTER_USER}" \
  | python -m json.tool
```

Busca:

```text
reporting-team
```

No es necesario que el usuario tenga roles directos; sus permisos deben provenir del grupo.

### Tarea 4.5. Probar acceso heredado permitido

- {% include step_label.html %} Autentica como `reporter-user` una consulta autorizada y confirma que el acceso provenga de los roles heredados del grupo.

```bash
curl -s \
  -u "${REPORTER_USER}:${REPORTER_PASS}" \
  "${CB_QUERY_URL}" \
  --data-urlencode \
  'statement=SELECT category_id, name
             FROM ecommerce.catalog.categories
             ORDER BY category_id;' \
  | python -m json.tool
```

### Tarea 4.6. Probar acceso heredado denegado

- {% include step_label.html %} Consulta una collection no autorizada como `reporter-user` y verifica que la herencia no amplíe permisos fuera del grupo.

```bash
curl -s \
  -u "${REPORTER_USER}:${REPORTER_PASS}" \
  "${CB_QUERY_URL}" \
  --data-urlencode \
  'statement=SELECT customer_id, name
             FROM ecommerce.sales.customers
             LIMIT 1;' \
  | python -m json.tool
```

La consulta debe fallar porque el grupo solo tiene acceso a `catalog`.

### Tarea 4.7. Comparar roles directos y heredados

- {% include step_label.html %} Compara en la matriz los roles directos con los heredados mediante `reporting-team`, indicando su origen y alcance sobre cada keyspace.

Agrega:

```markdown
## reporter-user

| Asignación | Valor |
|---|---|
| Roles directos | Ninguno |
| Grupo | reporting-team |
| Permiso heredado | SELECT en ecommerce.catalog |
| Acceso a ecommerce.sales | Denegado |
```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

{% include support-prompt.html task="tarea4" %}

---

## ✅ Tarea 5. Inspeccionar RBAC y ejecutar validación automatizada

En esta tarea revisarás los usuarios desde la Web Console y crearás un script resumido para comprobar la matriz de permisos.

### Tarea 5.1. Inspeccionar usuarios y grupos en Web Console

- {% include step_label.html %} Abre `http://localhost:8091` en el navegador para inspeccionar visualmente las cuentas, grupos y roles creados durante las pruebas RBAC.

  ```text
  http://localhost:8091
  ```

- {% include step_label.html %} Inicia sesión en la Web Console como `Administrator` para inspeccionar usuarios y grupos con permisos administrativos.
- {% include step_label.html %} Selecciona **Security** en la navegación lateral para abrir la administración de usuarios, grupos y asignaciones RBAC.
- {% include step_label.html %} Revisa **Security → Users** y confirma que la lista contenga las tres cuentas creadas antes de abrir sus detalles.
- {% include step_label.html %} En **Security → Users**, confirma que aparezcan `app-readonly`, `app-backend` y `reporter-user` antes de inspeccionar sus asignaciones.

  ```text
  app-readonly
  app-backend
  reporter-user
  ```

- {% include step_label.html %} En **Security → Groups**, localiza `reporting-team` y confirma que sus roles coincidan con los permisos heredados por `reporter-user`.

  ```text
  reporting-team
  ```

- {% include step_label.html %} Abre cada usuario en **Security → Users** y contrasta sus roles directos o grupo con la matriz de permisos documentada.
- {% include step_label.html %} Cierra el detalle de cada usuario sin guardar cambios, porque esta inspección visual no debe alterar las asignaciones RBAC validadas.

### Tarea 5.2. Crear el script validate_rbac.sh

- {% include step_label.html %} Crea `validate_rbac.sh` en el subdirectorio de la práctica y copia el script completo para automatizar las pruebas de permisos.

En VS Code, crea:

```text
validate_rbac.sh
```



```bash
#!/usr/bin/env bash

set -u

CB_QUERY_URL="http://localhost:8093/query/service"

test_query() {
  local user="$1"
  local pass="$2"
  local statement="$3"
  local expected="$4"
  local description="$5"

  local response
  local actual

  response=$(curl -s \
    -u "${user}:${pass}" \
    "${CB_QUERY_URL}" \
    --data-urlencode "statement=${statement}")

  if echo "${response}" | grep -Eq '"status"[[:space:]]*:[[:space:]]*"success"'; then
    actual="success"
  else
    actual="error"
  fi

  if [ "${actual}" = "${expected}" ]; then
    echo "PASS: ${description}"
  else
    echo "FAIL: ${description}"
    echo "Esperado: ${expected}"
    echo "Obtenido: ${actual}"
    echo "Respuesta:"
    echo "${response}"
  fi
}

echo "=============================================="
echo "VALIDACIÓN RBAC - PRÁCTICA 9"
echo "=============================================="

test_query \
  "app-readonly" \
  "Readonly@2026!" \
  'SELECT name FROM ecommerce.catalog.products LIMIT 1;' \
  "success" \
  "app-readonly puede consultar productos"

test_query \
  "app-readonly" \
  "Readonly@2026!" \
  'SELECT purchase_id FROM ecommerce.sales.purchases LIMIT 1;' \
  "error" \
  "app-readonly no puede consultar compras"

test_query \
  "app-backend" \
  "Backend@2026!" \
  'SELECT name FROM ecommerce.catalog.products LIMIT 1;' \
  "success" \
  "app-backend puede consultar productos"

test_query \
  "app-backend" \
  "Backend@2026!" \
  'UPDATE ecommerce.sales.purchases
   USE KEYS "ord::RBAC-003"
   SET status = "validated";' \
  "success" \
  "app-backend puede actualizar compras"

test_query \
  "app-backend" \
  "Backend@2026!" \
  'DELETE FROM ecommerce.sales.purchases
   USE KEYS "ord::RBAC-003";' \
  "error" \
  "app-backend no puede eliminar compras"

test_query \
  "reporter-user" \
  "Reporter@2026!" \
  'SELECT name FROM ecommerce.catalog.categories LIMIT 1;' \
  "success" \
  "reporter-user hereda lectura del catálogo"

test_query \
  "reporter-user" \
  "Reporter@2026!" \
  'SELECT name FROM ecommerce.sales.customers LIMIT 1;' \
  "error" \
  "reporter-user no puede consultar clientes"

echo "=============================================="
echo "VALIDACIÓN FINALIZADA"
echo "=============================================="
```

### Tarea 5.3. Dar permisos y ejecutar

- {% include step_label.html %} Asigna permiso de ejecución a `validate_rbac.sh` y ejecútalo desde Git Bash para resumir las pruebas permitidas y denegadas.

```bash
chmod +x validate_rbac.sh
./validate_rbac.sh
```

**Salida esperada:**

Para validar `Dar permisos y ejecutar`, verifica la referencia siguiente y confirma que la respuesta permita aplicar el bloque que comienza con `chmod +x validate_rbac.sh` y revisar su efecto específico antes de continuar; detente si aparece un error.

```text
PASS: app-readonly puede consultar productos
PASS: app-readonly no puede consultar compras
PASS: app-backend puede consultar productos
PASS: app-backend puede actualizar compras
PASS: app-backend no puede eliminar compras
PASS: reporter-user hereda lectura del catálogo
PASS: reporter-user no puede consultar clientes
```

### Tarea 5.4. Validar la configuración final

- {% include step_label.html %} Consulta mediante REST usuarios y grupos al finalizar, y confirma que las asignaciones coincidan con la matriz RBAC documentada.

Ejecuta como administrador:


```bash
curl -s -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_URL}/settings/rbac/users/local" \
  | python -m json.tool \
  | grep -E '"id"|"groups"'
```

Y:

- {% include step_label.html %} Consulta los grupos RBAC mediante REST y confirma que `reporting-team` aparezca con su identificador y descripción configurados.

```bash
curl -s -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_URL}/settings/rbac/groups" \
  | python -m json.tool \
  | grep -E '"id"|"description"'
```

### Tarea 5.5. Documentar la conclusión

- {% include step_label.html %} Redacta la conclusión de la práctica con las diferencias entre roles directos, grupos, herencia y principio de mínimo privilegio.

Agrega al final de `rbac-test-matrix.md`:

```markdown
## Conclusión

La configuración aplica mínimo privilegio porque:

- app-readonly solo puede consultar catalog.
- app-backend solo puede modificar purchases.
- app-backend no puede modificar products ni eliminar purchases.
- reporter-user no tiene roles directos.
- reporter-user hereda permisos desde reporting-team.
- ningún usuario de aplicación administra el clúster o el bucket.
```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

{% include support-prompt.html task="tarea5" %}

---

## 🛠️ Resolución de problemas

### Problema 1. Error 401 o Unauthenticated

Verifica las credenciales:

Aplica el bloque que comienza con `echo "${READONLY_USER}"` y revisa su efecto específico antes de continuar; conserva el bloque completo y revisa la respuesta antes de continuar.

```bash
echo "${READONLY_USER}"
echo "${BACKEND_USER}"
echo "${REPORTER_USER}"
```

Consulta los usuarios como administrador:

Consulta la lista de usuarios locales como administrador para confirmar que las cuentas existen y distinguir un error de credenciales de una cuenta ausente.

```bash
curl -s -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_URL}/settings/rbac/users/local" \
  | python -m json.tool
```

Si necesitas recrear un usuario, ejecuta nuevamente su comando `PUT` completo.

### Problema 2. Error por rol inválido o alcance incorrecto

Lista los roles disponibles:

Consulta el catálogo de roles disponibles y verifica el nombre y formato de alcance antes de recrear una asignación rechazada por Couchbase.

```bash
curl -s -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_URL}/settings/rbac/roles" \
  | python -m json.tool \
  | grep '"role"' \
  | sort -u
```

Verifica que el alcance use:

```text
bucket:scope
bucket:scope:collection
```

Ejemplo:

```text
query_select[ecommerce:catalog:products]
```

### Problema 3. La consulta falla por falta de índice

Consulta los índices:

Consulta `system:indexes` para verificar que los índices de `catalog` y `sales` existan y permanezcan `online` antes de repetir la operación autorizada.

```sql
SELECT name,
       bucket_id,
       scope_id,
       keyspace_id,
       state
FROM system:indexes
WHERE bucket_id = "ecommerce"
  AND scope_id IN ["catalog", "sales"]
ORDER BY scope_id, keyspace_id, name;
```

Todos los índices `idx_lab9_` deben estar `online`.

### Problema 4. Una operación que debía fallar aparece como success

Revisa los roles efectivos del usuario:

Consulta los roles efectivos de `app-backend` para detectar privilegios adicionales que expliquen por qué una operación prevista como denegada tuvo éxito.

```bash
curl -s -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_URL}/settings/rbac/users/local/${BACKEND_USER}" \
  | python -m json.tool
```

Confirma que no tenga roles adicionales asignados accidentalmente.

### Problema 5. reporter-user no hereda permisos

Verifica el grupo:

Consulta `reporting-team` mediante REST y confirma que el grupo conserve los roles de lectura que debe heredar `reporter-user`.

```bash
curl -s -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_URL}/settings/rbac/groups/reporting-team" \
  | python -m json.tool
```

Verifica el usuario:

Consulta `reporter-user` mediante REST y confirma que la propiedad `groups` incluya `reporting-team` antes de repetir la prueba heredada.

```bash
curl -s -u "${CB_ADMIN}:${CB_PASS}" \
  "${CB_URL}/settings/rbac/users/local/${REPORTER_USER}" \
  | python -m json.tool
```

El usuario debe pertenecer a:

```text
reporting-team
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

### Problema 7. python -m json.tool no funciona

Ejecuta:

Aplica el bloque que comienza con `python --version` y revisa su efecto específico antes de continuar; conserva el bloque completo y revisa la respuesta antes de continuar.

```bash
python --version
python3 --version
```

Si `python3` funciona, reemplaza:

Aplica el bloque que comienza con `python -m json.tool` y revisa su efecto específico antes de continuar; conserva el bloque completo y revisa la respuesta antes de continuar.

```bash
python -m json.tool
```

por:

Aplica el bloque que comienza con `python3 -m json.tool` y revisa su efecto específico antes de continuar; conserva el bloque completo y revisa la respuesta antes de continuar.

```bash
python3 -m json.tool
```

### Problema 8. La Web Console no muestra Query para un usuario restringido

La navegación visible puede variar según la versión y los roles. Realiza las pruebas mediante:

```text
http://localhost:8093/query/service
```

usando los comandos `curl` incluidos en la práctica.
