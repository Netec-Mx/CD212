---
layout: lab
title: "Práctica 8: Modelado de documentos JSON y diseño de keys"
permalink: /lab8/lab8/
images_base: /labs/lab8/img
duration: "60 minutos"
objective:
  - Preparar el directorio de trabajo de la práctica 8 y validar la continuidad con el modelo ecommerce.store creado en Couchbase Server Enterprise Edition durante la práctica 7.
  - Implementar un producto con variantes anidadas y consultar sus subdocumentos mediante UNNEST y ANY...SATISFIES.
  - Crear y validar un índice de array para búsquedas por SKU y stock dentro de variantes.
  - Comparar cuatro estrategias de document keys y analizar legibilidad, unicidad, predecibilidad y aislamiento por tenant.
  - Implementar tres estrategias de modelado para historial de compras: embedding total, referencing total y modelo híbrido.
  - Comparar el número de operaciones, los riesgos de crecimiento y los problemas de consistencia asociados a cada estrategia.
prerequisites:
  - Haber completado la Práctica 7.
  - Tener disponible el bucket ecommerce y el scope store.
  - Tener creadas las collections customers, products y orders.
  - Tener Docker Desktop en ejecución.
  - Tener activo el contenedor couchbase-lab creado con la imagen couchbase/server:enterprise-7.6.2.
  - Tener acceso a la Web Console en http://localhost:8091.
  - Utilizar Visual Studio Code con Git Bash como terminal integrada.
  - Comprender USE KEYS, UNNEST, ANY...SATISFIES, UPSERT y EXPLAIN.
introduction:
  - En esta práctica utilizarás Couchbase Server Enterprise Edition para profundizar el modelado documental iniciado en la práctica 7. Trabajarás con estructuras que requieren mayor control de crecimiento, como variantes de producto e historiales de compras. También compararás estrategias de document keys y comprobarás cómo una decisión de key puede afectar el acceso directo, la legibilidad y el aislamiento lógico. Finalmente implementarás y compararás tres estrategias para representar compras de clientes: embedding, referencing y modelo híbrido.
slug: lab8
lab_number: 8
final_result: >
  Al finalizar la práctica habrás implementado un producto con variantes anidadas, creado un índice de array funcional, comparado cuatro estrategias de document keys y construido tres modelos alternativos para historial de compras. Podrás justificar qué estrategia utilizar según patrones de acceso, crecimiento esperado, frecuencia de escritura, consistencia y necesidad de acceso directo.
notes:
  - Todos los comandos de terminal deben ejecutarse desde Git Bash dentro de Visual Studio Code.
  - Utiliza las credenciales Administrator y Password123! configuradas en las prácticas anteriores.
  - Esta práctica reutiliza ecommerce.store y no crea un bucket adicional.
  - Todos los índices creados utilizan el prefijo idx_lab8_.
  - Se utiliza UPSERT para que la práctica pueda repetirse sin errores por documentos existentes.
  - No elimines los documentos ni índices al finalizar; se conservarán para prácticas posteriores.
references: []
prev: /lab7/lab7/
next: /lab9/lab9/
---

---
<!-- Aquí comienzan las instrucciones paso a paso de la práctica -->

## 📁 Preparación del directorio de trabajo

En esta práctica conservarás el directorio raíz del curso y crearás únicamente el subdirectorio correspondiente a `lab8`.

### 🗂️ Crear y abrir el subdirectorio de la práctica

- {% include step_label.html %} Abre **Docker Desktop** y confirma que el motor esté activo, porque `couchbase-lab` depende del daemon local para exponer todos los servicios de Couchbase.
- {% include step_label.html %} Abre **Visual Studio Code** y espera su carga completa, ya que utilizarás el Explorador y la terminal integrada para desarrollar y documentar la práctica.
- {% include step_label.html %} Selecciona **File → Open Folder** en VS Code y abre `C:\LABS\couchbase-nosql` para trabajar dentro de la estructura que contiene los laboratorios del curso.

  ```text
  C:\LABS\couchbase-nosql
  ```

- {% include step_label.html %} Selecciona **Terminal → New Terminal** en VS Code para abrir la consola integrada desde la cual administrarás el contenedor y ejecutarás las validaciones.
- {% include step_label.html %} Comprueba en el selector del panel Terminal que **Git Bash** sea el perfil activo, porque los comandos emplean sintaxis y rutas compatibles con Bash.
- {% include step_label.html %} Crea `/c/LABS/couchbase-nosql/lab8` mediante `mkdir -p` en Git Bash para disponer del subdirectorio sin provocar errores si ya existe.

  ```bash
  mkdir -p /c/LABS/couchbase-nosql/lab8
  ```

- {% include step_label.html %} Cambia la ubicación activa de Git Bash a `/c/LABS/couchbase-nosql/lab8` para mantener los archivos y evidencias dentro de esta práctica.

  ```bash
  cd /c/LABS/couchbase-nosql/lab8
  ```

- {% include step_label.html %} Consulta la ruta activa mediante `pwd` y confirma que termine en `/lab8`; corrige la ubicación antes de continuar si aparece otro directorio.

  ```bash
  pwd
  ```

**Salida esperada:**

Para validar `crear y abrir el subdirectorio de trabajo`, verifica la referencia siguiente y confirma que la respuesta permita mostrar la ruta activa y confirmar que Git Bash está ubicado en el subdirectorio asignado a esta práctica; detente si aparece un error.

```text
/c/LABS/couchbase-nosql/lab8
```

---

## 🧩 Tarea 1. Preparar el entorno e implementar variantes anidadas

En esta tarea validarás la continuidad con la práctica 7 e insertarás un producto con variantes anidadas. El objetivo es representar talla, color, stock y precio adicional dentro de un solo documento de producto.

### Tarea 1.1. Verificar Couchbase Server

- {% include step_label.html %} Consulta `couchbase-lab` mediante `docker ps` en Git Bash y confirma que la columna de estado indique que el contenedor continúa en ejecución.

  {%raw%}
  ```bash
  docker ps --filter "name=couchbase-lab" --format "table {{.Names}}\t{{.Status}}"
  ```
  {%endraw%}

- {% include step_label.html %} Inicia `couchbase-lab` con `docker start` solamente si está detenido y espera la confirmación de Docker antes de solicitar los servicios internos.

  ```bash
  docker start couchbase-lab
  ```

- {% include step_label.html %} Solicita la Web Console mediante `curl` en Git Bash y confirma el código HTTP 200, que demuestra la disponibilidad del servicio en el puerto 8091.

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

- {% include step_label.html %} Inspecciona `couchbase-lab` desde Git Bash y verifica que la imagen configurada sea `couchbase/server:enterprise-7.6.2`, sin sustituirla por Community.

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

### Tarea 1.3. Verificar el modelo ecommerce.store

- {% include step_label.html %} Abre `http://localhost:8091` en el navegador y espera la pantalla de autenticación para ingresar a la Web Console del nodo local.

  ```text
  http://localhost:8091
  ```

- {% include step_label.html %} Inicia sesión con `Administrator` y `Password123!` para acceder al clúster mediante las credenciales administrativas definidas en prácticas anteriores.

| Campo | Valor |
|---|---|
| Usuario | `Administrator` |
| Contraseña | `Password123!` |

- {% include step_label.html %} Selecciona **Query** en la navegación lateral para abrir Query Workbench, donde ejecutarás las sentencias SQL++ de modelado y comparación.

  ```sql
  SELECT RAW name
  FROM system:keyspaces
  WHERE `bucket` = "ecommerce"
    AND `scope` = "store"
  ORDER BY name;
  ```

**Validación:**

Para validar `Verificar el modelo ecommerce.store`, verifica la referencia siguiente y confirma que la respuesta permita consultar `system` con el filtro ``bucket` = "ecommerce" AND `scope` = "store"` y comprueba las filas y el orden obtenidos; detente si aparece un error.

Debes observar al menos:

```text
customers
orders
products
```

### Tarea 1.4. Insertar un producto con variantes

- {% include step_label.html %} Inserta en `ecommerce.store.products` el producto `PROD-500` con sus variantes anidadas y confirma que la sentencia termine correctamente.



```sql
UPSERT INTO ecommerce.store.products (KEY, VALUE)
VALUES (
  "prod::PROD-500",
  {
    "type": "product",
    "product_id": "PROD-500",
    "sku": "APP-PRO-500",
    "name": "Camiseta Deportiva Pro",
    "description": "Camiseta de alto rendimiento con tecnología de secado rápido",
    "category_ids": ["CAT-SPORT"],
    "category_names": ["Ropa deportiva"],
    "base_price": 350.00,
    "currency": "MXN",
    "variants": [
      {
        "sku": "APP-PRO-S-BLK",
        "size": "S",
        "color": "Negro",
        "stock": 15,
        "additional_price": 0
      },
      {
        "sku": "APP-PRO-M-BLK",
        "size": "M",
        "color": "Negro",
        "stock": 22,
        "additional_price": 0
      },
      {
        "sku": "APP-PRO-L-BLK",
        "size": "L",
        "color": "Negro",
        "stock": 18,
        "additional_price": 0
      },
      {
        "sku": "APP-PRO-XL-BLK",
        "size": "XL",
        "color": "Negro",
        "stock": 8,
        "additional_price": 20
      },
      {
        "sku": "APP-PRO-S-WHT",
        "size": "S",
        "color": "Blanco",
        "stock": 12,
        "additional_price": 0
      },
      {
        "sku": "APP-PRO-M-WHT",
        "size": "M",
        "color": "Blanco",
        "stock": 20,
        "additional_price": 0
      }
    ],
    "active": true,
    "created_at": "2026-07-16T09:00:00Z"
  }
);
```

### Tarea 1.5. Consultar el documento completo

- {% include step_label.html %} Recupera el documento `PROD-500` completo desde Query Workbench para reconocer los campos principales y el arreglo embebido `variants`.

```sql
SELECT META(p).id AS documentKey,
       p.product_id,
       p.name,
       p.base_price,
       ARRAY_LENGTH(p.variants) AS total_variants,
       p.variants
FROM ecommerce.store.products AS p
USE KEYS "prod::PROD-500";
```

**Validación:**

Para validar `Consultar el documento completo`, verifica la referencia siguiente y confirma que la respuesta permita consultar `ecommerce.store.products` y comprueba las filas y el orden obtenidos; detente si aparece un error.

Debes obtener:

```text
documentKey = prod::PROD-500
total_variants = 6
```

### Tarea 1.6. Explicar la decisión de embedding

- {% include step_label.html %} Analiza la decisión de mantener las variantes dentro del producto y registra cómo la lectura conjunta evita consultar documentos relacionados.

Registra en un archivo llamado:

```text
lab8-design-notes.md
```

el siguiente contenido:

```markdown
## Producto con variantes

Las variantes se embeben dentro del producto porque:

- Siempre se consultan en el contexto del producto.
- El número de variantes es controlado.
- Comparten el ciclo de vida del producto.
- No requieren una collection independiente.
- El acceso por document key devuelve producto y variantes en una sola lectura.

Riesgo aceptado:

Si el número de variantes creciera sin límite o se actualizaran de forma
muy concurrente, sería necesario separar las variantes en documentos
independientes.
```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

{% include support-prompt.html task="tarea1" %}

---

## 🔍 Tarea 2. Consultar variantes y crear un índice de array

En esta tarea utilizarás UNNEST y ANY...SATISFIES para consultar variantes embebidas. Después crearás un índice de array y confirmarás su uso mediante EXPLAIN.

### Tarea 2.1. Expandir variantes con UNNEST

- {% include step_label.html %} Descompón `variants` mediante `UNNEST` para producir una fila por variante y conservar los datos generales del producto en cada resultado.



```sql
SELECT p.product_id,
       p.name,
       v.sku,
       v.size,
       v.color,
       v.stock,
       p.base_price + v.additional_price AS final_price
FROM ecommerce.store.products AS p
UNNEST p.variants AS v
WHERE p.product_id = "PROD-500"
ORDER BY v.size, v.color;
```

**Interpretación:**

`UNNEST` convierte cada elemento del array `variants` en una fila independiente sin modificar el documento original.

### Tarea 2.2. Consultar variantes con stock mayor a 10

- {% include step_label.html %} Filtra las variantes de `PROD-500` cuyo stock sea mayor que diez y comprueba que ninguna fila devuelta incumpla la condición numérica.

```sql
SELECT p.name,
       v.sku,
       v.size,
       v.color,
       v.stock
FROM ecommerce.store.products AS p
UNNEST p.variants AS v
WHERE p.product_id = "PROD-500"
  AND v.stock > 10
ORDER BY v.stock DESC;
```

### Tarea 2.3. Identificar productos con stock bajo

- {% include step_label.html %} Localiza productos con alguna variante por debajo de diez unidades mediante `ANY ... SATISFIES` y revisa que el arreglo cumpla el predicado.

```sql
SELECT META(p).id AS documentKey,
       p.name
FROM ecommerce.store.products AS p
WHERE ANY v IN p.variants
      SATISFIES v.stock < 10
      END;
```

**Resultado esperado:**

Para validar `Identificar productos con stock bajo`, verifica la referencia siguiente y confirma que la respuesta permita consultar `ecommerce.store.products` con el filtro `ANY v IN p.variants SATISFIES v.stock < 10 END` y comprueba las filas y el orden obtenidos; detente si aparece un error.

Debe aparecer:

```text
prod::PROD-500
```

porque la variante `APP-PRO-XL-BLK` tiene stock igual a 8.

### Tarea 2.4. Crear el índice de array


- {% include step_label.html %} Crea el índice `idx_lab8_product_variant_sku` para el patrón de consulta analizado; conserva la sentencia completa y revisa su respuesta antes de continuar.

```sql
CREATE INDEX IF NOT EXISTS idx_lab8_product_variant_sku
ON ecommerce.store.products(
  ALL ARRAY v.sku FOR v IN variants END
)
WHERE variants IS NOT MISSING;
```

- {% include step_label.html %} Consulta `system:indexes` en Query Workbench y confirma que el índice de arreglo exista, tenga las claves previstas y permanezca en estado `online`.

  ```sql
  SELECT name,
         keyspace_id,
         state,
         `index_key`,
         condition
  FROM system:indexes
  WHERE bucket_id = "ecommerce"
    AND scope_id = "store"
    AND name = "idx_lab8_product_variant_sku";
  ```

**Resultado esperado:**

Para validar `Crear el índice de array`, verifica la referencia siguiente y confirma que la respuesta permita inventariar en `system:indexes` los índices del bucket, scope y collections especificados por los filtros; detente si aparece un error.

```text
state = online
```

### Tarea 2.5. Buscar una variante por SKU

- {% include step_label.html %} Busca la variante con SKU `APP-PRO-M-BLK` mediante `UNNEST` para comprobar que el índice de arreglo respalda una búsqueda por elemento.

```sql
SELECT META(p).id AS documentKey,
       p.name,
       v.sku,
       v.size,
       v.color,
       v.stock
FROM ecommerce.store.products AS p
UNNEST p.variants AS v
WHERE v.sku = "APP-PRO-M-BLK";
```

### Tarea 2.6. Validar el uso del índice

- {% include step_label.html %} Analiza el plan con `EXPLAIN` y verifica que aparezcan el índice de variantes y los operadores esperados, sin asumir su uso por el nombre solamente.

```sql
EXPLAIN
SELECT META(p).id AS documentKey,
       p.name,
       v.sku,
       v.size,
       v.color
FROM ecommerce.store.products AS p
UNNEST p.variants AS v
WHERE v.sku = "APP-PRO-M-BLK";
```

Busca en el plan:

```text
idx_lab8_product_variant_sku
```

> **IMPORTANTE:** Confirmar que el índice existe no es suficiente. EXPLAIN permite verificar si el optimizador lo utiliza realmente.
{: .lab-note .important .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

{% include support-prompt.html task="tarea2" %}

---

## 🔑 Tarea 3. Comparar estrategias de document keys

En esta tarea crearás cuatro documentos de cliente utilizando estrategias diferentes de key. No se utilizarán correos electrónicos ni otros datos personales dentro de la document key.

### Tarea 3.1. Estrategia A: UUID puro

- {% include step_label.html %} Inserta el documento de estrategia UUID puro en `ecommerce.store.customers` para representar una clave opaca sin información de negocio.



```sql
UPSERT INTO ecommerce.store.customers (KEY, VALUE)
VALUES (
  "550e8400-e29b-41d4-a716-446655440000",
  {
    "type": "customer_key_strategy",
    "customer_id": "CUST-UUID-001",
    "name": "Ana Torres",
    "key_strategy": "A_UUID_ONLY",
    "description": "Clave opaca sin información semántica."
  }
);
```

### Tarea 3.2. Estrategia B: tipo más UUID

- {% include step_label.html %} Registra una clave formada por tipo y UUID para distinguir la entidad visualmente sin incorporar atributos de negocio susceptibles a cambios.

```sql
UPSERT INTO ecommerce.store.customers (KEY, VALUE)
VALUES (
  "cust::550e8400-e29b-41d4-a716-446655440001",
  {
    "type": "customer_key_strategy",
    "customer_id": "CUST-UUID-002",
    "name": "Roberto Silva",
    "key_strategy": "B_TYPE_UUID",
    "description": "Clave única con prefijo que identifica el tipo."
  }
);
```

### Tarea 3.3. Estrategia C: tipo y campo de negocio estable

- {% include step_label.html %} Crea una clave con tipo y campo de negocio estable para evaluar su legibilidad y el riesgo de depender de un valor funcional.

```sql
UPSERT INTO ecommerce.store.customers (KEY, VALUE)
VALUES (
  "cust::external::CRM-10050",
  {
    "type": "customer_key_strategy",
    "customer_id": "CUST-CRM-10050",
    "external_id": "CRM-10050",
    "name": "Lucía Herrera",
    "key_strategy": "C_TYPE_BUSINESS_KEY",
    "description": "Clave predecible basada en un identificador externo estable."
  }
);
```

### Tarea 3.4. Estrategia D: tipo, tenant e ID local

- {% include step_label.html %} Inserta la variante con tipo, tenant e identificador local para demostrar cómo una clave puede delimitar datos pertenecientes a varios clientes.

```sql
UPSERT INTO ecommerce.store.customers (KEY, VALUE)
VALUES
(
  "cust::tenant::mx::10050",
  {
    "type": "customer_key_strategy",
    "customer_id": "CUST-MX-10050",
    "tenant": "mx",
    "local_id": "10050",
    "name": "Fernando Castillo",
    "key_strategy": "D_MULTI_TENANT"
  }
),
(
  "cust::tenant::us::10050",
  {
    "type": "customer_key_strategy",
    "customer_id": "CUST-US-10050",
    "tenant": "us",
    "local_id": "10050",
    "name": "Fernando Castillo US",
    "key_strategy": "D_MULTI_TENANT"
  }
);
```

### Tarea 3.5. Recuperar documentos por key

- {% include step_label.html %} Recupera los cuatro documentos mediante `USE KEYS` y confirma que el acceso directo funciona sin necesitar un índice secundario.

```sql
SELECT META(c).id AS documentKey,
       c.name,
       c.key_strategy,
       c.tenant
FROM ecommerce.store.customers AS c
USE KEYS [
  "550e8400-e29b-41d4-a716-446655440000",
  "cust::550e8400-e29b-41d4-a716-446655440001",
  "cust::external::CRM-10050",
  "cust::tenant::mx::10050",
  "cust::tenant::us::10050"
];
```

### Tarea 3.6. Comparar propiedades de las keys

- {% include step_label.html %} Compara longitud, legibilidad, estabilidad y contexto de las cuatro claves para identificar las ventajas y limitaciones de cada estrategia.



```sql
SELECT META(c).id AS documentKey,
       c.name,
       c.key_strategy,
       LENGTH(META(c).id) AS key_length
FROM ecommerce.store.customers AS c
WHERE c.type = "customer_key_strategy"
ORDER BY c.key_strategy, META(c).id;
```

Registra en `lab8-design-notes.md`:

```markdown
## Comparación de estrategias de keys

| Estrategia | Legibilidad | Predecible | Unicidad | Acceso directo | Riesgo principal |
|---|---|---|---|---|---|
| UUID puro | Baja | No | Alta | Solo si se conoce la key | Difícil de depurar |
| tipo::UUID | Media | No | Alta | Solo si se conoce la key | Key extensa |
| tipo::external-id | Alta | Sí | Depende del sistema externo | Sí | El identificador externo puede cambiar |
| tipo::tenant::ID | Alta | Sí | Alta por tenant | Sí | Requiere convención consistente |
```

> **IMPORTANTE:** Evita incluir correos, teléfonos, nombres u otros datos personales directamente en document keys.
{: .lab-note .important .compact}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

{% include support-prompt.html task="tarea3" %}

---

## ⚖️ Tarea 4. Implementar embedding, referencing e híbrido

En esta tarea representarás el historial de compras de un cliente mediante tres estrategias diferentes. El objetivo es comparar operaciones, crecimiento y consistencia.

### Tarea 4.1. Crear el modelo con embedding total

- {% include step_label.html %} Inserta el ejemplo con historial completamente embebido y observa que cliente y compras pueden recuperarse mediante una sola document key.



```sql
UPSERT INTO ecommerce.store.customers (KEY, VALUE)
VALUES (
  "cust::embed::2001",
  {
    "type": "customer_embedding_total",
    "customer_id": "CUST-EMBED-2001",
    "name": "Isabel Vargas",
    "address": {
      "street": "Blvd. Kukulcán km 12",
      "city": "Cancún",
      "state": "Quintana Roo",
      "zip": "77500"
    },
    "purchase_history": [
      {
        "order_id": "ORD-2026-100",
        "created_at": "2026-06-01T09:00:00Z",
        "total": 1500.00
      },
      {
        "order_id": "ORD-2026-150",
        "created_at": "2026-07-01T09:00:00Z",
        "total": 900.00
      }
    ],
    "strategy": "embedding_total"
  }
);
```

**Interpretación:**

El perfil completo y el historial se obtienen en una sola lectura, pero el documento crece con cada compra.

### Tarea 4.2. Crear el modelo con referencing total

- {% include step_label.html %} Crea el ejemplo con referencias independientes para separar cliente y pedidos, permitiendo que cada documento cambie sin reescribir el otro.

```sql
UPSERT INTO ecommerce.store.customers (KEY, VALUE)
VALUES (
  "cust::ref::2002",
  {
    "type": "customer_referencing_total",
    "customer_id": "CUST-REF-2002",
    "name": "Miguel Ángel Reyes",
    "address": {
      "street": "Calle 5 de Mayo 88",
      "city": "Monterrey",
      "state": "Nuevo León",
      "zip": "64000"
    },
    "strategy": "referencing_total"
  }
);
```

- {% include step_label.html %} Inserta el pedido del modelo referenciado en `ecommerce.store.orders` y confirma que pueda consultarse sin reescribir el documento del cliente.

```sql
UPSERT INTO ecommerce.store.orders (KEY, VALUE)
VALUES
(
  "ord::ref::2002::ORD-2026-200",
  {
    "type": "order_referencing",
    "order_id": "ORD-2026-200",
    "customer_id": "CUST-REF-2002",
    "customer_key": "cust::ref::2002",
    "created_at": "2026-06-10T10:00:00Z",
    "total": 2200.00
  }
),
(
  "ord::ref::2002::ORD-2026-280",
  {
    "type": "order_referencing",
    "order_id": "ORD-2026-280",
    "customer_id": "CUST-REF-2002",
    "customer_key": "cust::ref::2002",
    "created_at": "2026-07-05T10:00:00Z",
    "total": 650.00
  }
);
```

### Tarea 4.3. Crear el modelo híbrido

- {% include step_label.html %} Registra el modelo híbrido con un resumen embebido y pedidos independientes para combinar lectura rápida con historial detallado consultable.

```sql
UPSERT INTO ecommerce.store.customers (KEY, VALUE)
VALUES (
  "cust::hybrid::2003",
  {
    "type": "customer_hybrid",
    "customer_id": "CUST-HYBRID-2003",
    "name": "Valentina Cruz",
    "address": {
      "street": "Paseo de la Reforma 250",
      "city": "Ciudad de México",
      "state": "CDMX",
      "zip": "06500"
    },
    "purchase_summary": {
      "total_orders": 2,
      "historical_amount": 4150.00,
      "last_purchase": "2026-07-08T11:00:00Z",
      "recent_orders": [
        {
          "order_id": "ORD-2026-310",
          "created_at": "2026-07-08T11:00:00Z",
          "total": 3200.00
        },
        {
          "order_id": "ORD-2026-250",
          "created_at": "2026-06-20T11:00:00Z",
          "total": 950.00
        }
      ]
    },
    "strategy": "hybrid"
  }
);
```

- {% include step_label.html %} Inserta el pedido detallado del modelo híbrido en `ecommerce.store.orders` y conserva en el cliente solamente el resumen de compra requerido.

```sql
UPSERT INTO ecommerce.store.orders (KEY, VALUE)
VALUES
(
  "ord::hybrid::2003::ORD-2026-310",
  {
    "type": "order_hybrid",
    "order_id": "ORD-2026-310",
    "customer_id": "CUST-HYBRID-2003",
    "customer_key": "cust::hybrid::2003",
    "created_at": "2026-07-08T11:00:00Z",
    "total": 3200.00
  }
),
(
  "ord::hybrid::2003::ORD-2026-250",
  {
    "type": "order_hybrid",
    "order_id": "ORD-2026-250",
    "customer_id": "CUST-HYBRID-2003",
    "customer_key": "cust::hybrid::2003",
    "created_at": "2026-06-20T11:00:00Z",
    "total": 950.00
  }
);
```

### Tarea 4.4. Crear un índice para pedidos referenciados

- {% include step_label.html %} Crea `idx_lab8_orders_customer_key` sobre los pedidos referenciados y confirma que el índice permita localizar documentos por cliente.

```sql
CREATE INDEX IF NOT EXISTS idx_lab8_orders_customer_key
ON ecommerce.store.orders(
  customer_key,
  created_at DESC,
  total
)
WHERE customer_key IS NOT MISSING;
```

### Tarea 4.5. Validar los tres modelos

- {% include step_label.html %} Consulta los ejemplos de embedding, referencing e híbrido y contrasta la forma, cantidad de documentos y datos devueltos por cada modelo.

**Embedding:**


```sql
SELECT c.name,
       c.purchase_history
FROM ecommerce.store.customers AS c
USE KEYS "cust::embed::2001";
```

**Referencing:**

- {% include step_label.html %} Consulta `ecommerce.store.customers` y proyecta `c.name, c.address` y comprueba las filas y el orden obtenidos; conserva la sentencia completa y revisa su respuesta antes de continuar.

```sql
SELECT c.name,
       c.address
FROM ecommerce.store.customers AS c
USE KEYS "cust::ref::2002";
```

- {% include step_label.html %} Consulta `ecommerce.store.orders` con el filtro `customer_key = "cust::ref::2002"` y comprueba las filas y el orden obtenidos; conserva la sentencia completa y revisa su respuesta antes de continuar.

```sql
SELECT order_id,
       created_at,
       total
FROM ecommerce.store.orders
WHERE customer_key = "cust::ref::2002"
ORDER BY created_at DESC;
```

**Híbrido:**

- {% include step_label.html %} Consulta `ecommerce.store.customers` y proyecta `c.name, c.purchase_summary` y comprueba las filas y el orden obtenidos; conserva la sentencia completa y revisa su respuesta antes de continuar.

```sql
SELECT c.name,
       c.purchase_summary
FROM ecommerce.store.customers AS c
USE KEYS "cust::hybrid::2003";
```

- {% include step_label.html %} Consulta `ecommerce.store.orders` con el filtro `customer_key = "cust::hybrid::2003"` y comprueba las filas y el orden obtenidos; conserva la sentencia completa y revisa su respuesta antes de continuar.

```sql
SELECT order_id,
       created_at,
       total
FROM ecommerce.store.orders
WHERE customer_key = "cust::hybrid::2003"
ORDER BY created_at DESC;
```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

{% include support-prompt.html task="tarea4" %}

---

## 📊 Tarea 5. Comparar operaciones y documentar trade-offs

En esta tarea comprobarás cómo cambia el número de operaciones según la estrategia y analizarás los riesgos de crecimiento y consistencia.

### Tarea 5.1. Agregar una compra al modelo embebido

- {% include step_label.html %} Agrega una compra al arreglo embebido mediante `UPDATE` y comprueba que la modificación reescriba el documento del cliente seleccionado.

```sql
UPDATE ecommerce.store.customers
USE KEYS "cust::embed::2001"
SET purchase_history = ARRAY_APPEND(
  purchase_history,
  {
    "order_id": "ORD-2026-NEW-EMBED",
    "created_at": NOW_STR(),
    "total": 750.00
  }
);
```

**Interpretación:**

Se realiza una escritura, pero el documento completo del cliente continúa creciendo.

### Tarea 5.2. Agregar una compra al modelo referenciado

- {% include step_label.html %} Inserta un pedido independiente para el modelo referenciado y confirma que la operación no requiera modificar el documento del cliente.

```sql
UPSERT INTO ecommerce.store.orders (KEY, VALUE)
VALUES (
  "ord::ref::2002::ORD-2026-NEW",
  {
    "type": "order_referencing",
    "order_id": "ORD-2026-NEW",
    "customer_id": "CUST-REF-2002",
    "customer_key": "cust::ref::2002",
    "created_at": NOW_STR(),
    "total": 750.00
  }
);
```

**Interpretación:**

Se crea un documento independiente sin modificar el cliente.

### Tarea 5.3. Agregar una compra al modelo híbrido

- {% include step_label.html %} Inserta el pedido detallado del modelo híbrido en `ecommerce.store.orders` para mantener el historial completo como documento independiente.

  ```sql
  UPSERT INTO ecommerce.store.orders (KEY, VALUE)
  VALUES (
    "ord::hybrid::2003::ORD-2026-NEW",
    {
      "type": "order_hybrid",
      "order_id": "ORD-2026-NEW",
      "customer_id": "CUST-HYBRID-2003",
      "customer_key": "cust::hybrid::2003",
      "created_at": NOW_STR(),
      "total": 750.00
    }
  );
  ```

- {% include step_label.html %} Actualiza el resumen embebido del cliente híbrido y verifica que refleje la compra nueva sin duplicar todos los detalles del pedido.

  ```sql
  UPDATE ecommerce.store.customers
  USE KEYS "cust::hybrid::2003"
  SET purchase_summary.total_orders =
        purchase_summary.total_orders + 1,
      purchase_summary.historical_amount =
        purchase_summary.historical_amount + 750.00,
      purchase_summary.last_purchase = NOW_STR(),
      purchase_summary.recent_orders =
        ARRAY_PREPEND(
          {
            "order_id": "ORD-2026-NEW",
            "created_at": NOW_STR(),
            "total": 750.00
          },
          purchase_summary.recent_orders
        )[0:3];
  ```

> **IMPORTANTE:** El modelo híbrido requiere dos escrituras. Si una se completa y la otra falla, el resumen puede quedar inconsistente. En producción se puede utilizar una transacción, Eventing o un proceso de reconciliación.
{: .lab-note .important .compact}

### Tarea 5.4. Comparar planes de acceso directo y consulta indexada

- {% include step_label.html %} Compara con `EXPLAIN` el acceso por document key y la consulta mediante índice, identificando operadores y recursos implicados en cada alternativa.

**Acceso por key conocida:**


```sql
EXPLAIN
SELECT META(o).id,
       o.order_id,
       o.total
FROM ecommerce.store.orders AS o
USE KEYS [
  "ord::ref::2002::ORD-2026-200",
  "ord::ref::2002::ORD-2026-280"
];
```

**Consulta por atributo indexado:**

- {% include step_label.html %} Obtén el plan de ejecución e identifica el índice, los spans y los operadores seleccionados por Query; contrasta la respuesta nueva con la obtenida previamente.

```sql
EXPLAIN
SELECT META(o).id,
       o.order_id,
       o.total
FROM ecommerce.store.orders AS o
WHERE o.customer_key = "cust::ref::2002"
ORDER BY o.created_at DESC;
```

**Validación:**

Para validar `Comparar planes de acceso directo y consulta indexada`, verifica la referencia siguiente y confirma que la respuesta permita obtener el plan de ejecución e identifica el índice, los spans y los operadores seleccionados por Query; detente si aparece un error.

- En `USE KEYS`, confirma que no se utiliza un `PrimaryScan` para localizar los documentos.
- En la consulta por `customer_key`, busca `idx_lab8_orders_customer_key`.

### Tarea 5.5. Crear la matriz de trade-offs

- {% include step_label.html %} Completa la matriz de trade-offs con costo de lectura, escritura, consistencia y crecimiento para justificar cuándo usar cada modelo documental.

Agrega a `lab8-design-notes.md`:

```markdown
## Matriz de trade-offs

| Criterio | Embedding total | Referencing total | Híbrido |
|---|---|---|---|
| Leer perfil y resumen | 1 lectura | 2 lecturas | 1 lectura |
| Leer historial completo | 1 lectura | 2 lecturas | 2 lecturas |
| Crear una compra | 1 actualización | 1 inserción | 1 inserción + 1 actualización |
| Crecimiento del cliente | Alto | Bajo | Controlado |
| Escrituras concurrentes | Riesgo alto | Riesgo bajo | Riesgo medio |
| Consistencia | Dentro de un documento | Referencias externas | Resumen y detalle pueden divergir |
| Uso recomendado | Historial pequeño y acotado | Historial grande | Resumen rápido con detalle separado |
```

### Tarea 5.6. Ejecutar validaciones finales

- {% include step_label.html %} Ejecuta las consultas finales de productos, clientes y pedidos, y confirma que los documentos creados conservan relaciones, claves y estructuras esperadas.

```sql
SELECT META(p).id AS documentKey,
       p.product_id,
       ARRAY_LENGTH(p.variants) AS total_variants
FROM ecommerce.store.products AS p
USE KEYS "prod::PROD-500";
```

- {% include step_label.html %} Consulta `ecommerce.store.customers` con el filtro `type = "customer_key_strategy"` y comprueba las filas y el orden obtenidos; conserva la sentencia completa y revisa su respuesta antes de continuar.

```sql
SELECT COUNT(*) AS key_strategy_documents
FROM ecommerce.store.customers
WHERE type = "customer_key_strategy";
```

- {% include step_label.html %} Consulta `ecommerce.store.customers` y proyecta `type, COUNT(*) AS total` y comprueba las filas y el orden obtenidos; conserva la sentencia completa y revisa su respuesta antes de continuar.

```sql
SELECT type,
       COUNT(*) AS total
FROM ecommerce.store.customers
WHERE type IN [
  "customer_embedding_total",
  "customer_referencing_total",
  "customer_hybrid"
]
GROUP BY type
ORDER BY type;
```

- {% include step_label.html %} Consulta `system:indexes` y revisa los índices cuyo nombre coincide con `idx_lab8_%`; conserva la sentencia completa y revisa su respuesta antes de continuar.

```sql
SELECT name,
       keyspace_id,
       state
FROM system:indexes
WHERE bucket_id = "ecommerce"
  AND scope_id = "store"
  AND name LIKE "idx_lab8_%"
ORDER BY keyspace_id, name;
```

**Resultado esperado:**

Para validar `Ejecutar validaciones finales`, verifica la referencia siguiente y confirma que la respuesta permita consultar `system:indexes` y revisa los índices cuyo nombre coincide con `idx_lab8_%`; detente si aparece un error.

Todos los índices deben estar `online`.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

{% include support-prompt.html task="tarea5" %}

---

## 🛠️ Resolución de problemas

### Problema 1. ecommerce.store no existe

Verifica:

Consulta `system` con el filtro ``bucket` = "ecommerce" AND `scope` = "store"` y comprueba las filas y el orden obtenidos; conserva la sentencia completa y revisa su respuesta antes de continuar.

```sql
SELECT RAW name
FROM system:keyspaces
WHERE `bucket` = "ecommerce"
  AND `scope` = "store";
```

Si no aparecen las collections, revisa la práctica 7 antes de continuar.

### Problema 2. UNNEST no devuelve resultados

Verifica que el documento tenga el campo `variants`:

Consulta `ecommerce.store.products` y proyecta `META(p).id, ARRAY_LENGTH(p.variants) AS total_variants` y comprueba las filas y el orden obtenidos; conserva la sentencia completa y revisa su respuesta antes de continuar.

```sql
SELECT META(p).id,
       ARRAY_LENGTH(p.variants) AS total_variants
FROM ecommerce.store.products AS p
USE KEYS "prod::PROD-500";
```

Asegúrate de utilizar el mismo alias en `UNNEST` y en `SELECT`.

### Problema 3. El índice de array no se utiliza

Verifica:

Consulta `system:indexes` y confirmar la definición y el estado del índice `idx_lab8_product_variant_sku`; conserva la sentencia completa y revisa su respuesta antes de continuar.

```sql
SELECT name, state, `index_key`
FROM system:indexes
WHERE name = "idx_lab8_product_variant_sku";
```

Después ejecuta nuevamente `EXPLAIN` y confirma que la consulta utiliza `v.sku`.

### Problema 4. USE KEYS devuelve 0 resultados

Revisa que la key coincida exactamente con la utilizada en el `UPSERT`.

Consulta:

Vuelve a consultar los documentos con `type = "customer_key_strategy"` y confirma en la validación final que las cuatro estrategias continúen disponibles.

```sql
SELECT META().id
FROM ecommerce.store.customers
WHERE type = "customer_key_strategy";
```

### Problema 5. La consulta de pedidos usa un índice primario

Confirma que existe:

Consulta `system:indexes` y confirmar la definición y el estado del índice `idx_lab8_orders_customer_key`; conserva la sentencia completa y revisa su respuesta antes de continuar.

```sql
SELECT name, state
FROM system:indexes
WHERE name = "idx_lab8_orders_customer_key";
```

La consulta debe incluir:

Aplica el bloque que comienza con `WHERE customer_key = "cust::ref::2002"` y revisa su efecto específico antes de continuar; conserva la sentencia completa y revisa su respuesta antes de continuar.

```sql
WHERE customer_key = "cust::ref::2002"
```

### Problema 6. ARRAY_PREPEND o el slice generan error

Verifica que `purchase_summary.recent_orders` exista y sea un array:

Consulta `ecommerce.store.customers` y proyecta `purchase_summary.recent_orders` y comprueba las filas y el orden obtenidos; conserva la sentencia completa y revisa su respuesta antes de continuar.

```sql
SELECT purchase_summary.recent_orders
FROM ecommerce.store.customers
USE KEYS "cust::hybrid::2003";
```

### Problema 7. El contenedor activo utiliza Community Edition

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

### Problema 8. python -m json.tool falla

Ejecuta:

Aplica el bloque que comienza con `python --version` y revisa su efecto específico antes de continuar; conserva la sentencia completa y revisa su respuesta antes de continuar.

```bash
python --version
python3 --version
```

Si `python3` funciona, reemplaza:

Aplica el bloque que comienza con `python -m json.tool` y revisa su efecto específico antes de continuar; conserva la sentencia completa y revisa su respuesta antes de continuar.

```bash
python -m json.tool
```

por:

Aplica el bloque que comienza con `python3 -m json.tool` y revisa su efecto específico antes de continuar; conserva la sentencia completa y revisa su respuesta antes de continuar.

```bash
python3 -m json.tool
```
