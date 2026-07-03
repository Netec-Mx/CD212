# Modelado de documentos JSON y diseño de keys

## Metadatos

| Atributo        | Valor                                                                 |
|-----------------|-----------------------------------------------------------------------|
| **Duración**    | 60 minutos                                                            |
| **Complejidad** | Media                                                                 |
| **Nivel Bloom** | Crear (*Create*)                                                      |
| **Laboratorio** | 08-00-01                                                              |
| **Dependencias**| Lab 07-00-01 completado; bucket `tienda` con scope `_default` activo |

---

## Descripción General

En este laboratorio aplicarás estrategias de modelado de documentos JSON en Couchbase, trabajando con tres dimensiones complementarias: **anidamiento de datos** (*data nesting*), **diseño de claves de documento** (*key design*) y **análisis de trade-offs** entre distintas estrategias de modelado. Partirás de escenarios de negocio concretos para tomar decisiones de diseño justificadas, insertarás documentos reales y validarás que las consultas SQL++ funcionan correctamente sobre las estructuras creadas. Al finalizar, habrás construido un conjunto de documentos que ilustran los patrones más comunes en aplicaciones distribuidas con Couchbase.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Implementar documentos JSON con objetos anidados y arreglos de sub-documentos para representar relaciones uno-a-uno y uno-a-muchos dentro de un único documento Couchbase.
- [ ] Diseñar y comparar cuatro estrategias de claves de documento (UUID puro, `tipo::UUID`, `tipo::campo-negocio`, `tipo::scope::ID`) evaluando su impacto en consultas `USE KEYS` y en la legibilidad del modelo.
- [ ] Evaluar los trade-offs entre *embedding total*, *referencing total* y modelo híbrido, midiendo el número de operaciones necesarias para los patrones de acceso más comunes.
- [ ] Aplicar convenciones de nomenclatura de keys en Couchbase usando prefijos de tipo, separadores `::` e IDs compuestos.
- [ ] Crear índices de arreglo y ejecutar consultas con `UNNEST` y `ANY...SATISFIES` sobre documentos con datos anidados.

---

## Prerequisitos

### Conocimiento previo

- Haber completado el **Lab 07-00-01** (diseño de modelo de datos en Couchbase).
- Comprensión de instrucciones SQL++ `INSERT`, `SELECT`, `USE KEYS` y `UPSERT`.
- Familiaridad con estructuras JSON complejas: arreglos de objetos, objetos anidados, valores nulos.

### Acceso requerido

- Couchbase Server 7.6.x en ejecución (nodo único).
- Acceso a la **Web Console** en `http://localhost:8091` con credenciales de administrador.
- Acceso a **cbq** (Couchbase Query Shell) o a la pestaña **Query** de la Web Console.
- Bucket `tienda` creado con el scope `_default` y colecciones `clientes`, `pedidos`, `productos`, `blogs` y `perfiles` disponibles (se crearán en el paso de configuración si no existen).

---

## Entorno de Laboratorio

### Hardware mínimo recomendado

| Recurso       | Mínimo        | Recomendado   |
|---------------|---------------|---------------|
| CPU           | 4 núcleos x86 | 8 núcleos     |
| RAM           | 8 GB          | 16 GB         |
| Almacenamiento| 20 GB libres  | 50 GB SSD     |
| Red           | localhost     | localhost     |

### Software requerido

| Componente          | Versión mínima | Notas                                      |
|---------------------|----------------|--------------------------------------------|
| Couchbase Server    | 7.6.x          | Community Edition o Enterprise Trial       |
| cbq / Query Console | Incluido 7.6.x | Disponible en Web Console → Query          |
| Navegador web       | Chrome/Firefox 110+ | Para Web Console                      |
| curl                | 7.x            | Opcional, para verificaciones REST         |

### Configuración inicial del entorno

Ejecuta los siguientes comandos en la pestaña **Query** de la Web Console (o en `cbq`) para preparar el bucket y las colecciones necesarias. Si el bucket `tienda` ya existe de laboratorios anteriores, verifica que las colecciones estén creadas.

```sql
-- Verificar que el bucket 'tienda' existe
SELECT name FROM system:buckets WHERE name = 'tienda';
```

Si el bucket no existe, créalo desde la Web Console en **Buckets → Add Bucket** con 256 MB de RAM mínimo, luego ejecuta:

```sql
-- Crear colecciones necesarias para este laboratorio
CREATE COLLECTION tienda._default.clientes IF NOT EXISTS;
CREATE COLLECTION tienda._default.pedidos IF NOT EXISTS;
CREATE COLLECTION tienda._default.productos IF NOT EXISTS;
CREATE COLLECTION tienda._default.blogs IF NOT EXISTS;
CREATE COLLECTION tienda._default.perfiles IF NOT EXISTS;
```

```sql
-- Crear índice primario en cada colección para permitir SELECT sin filtro durante el lab
CREATE PRIMARY INDEX IF NOT EXISTS ON tienda._default.clientes;
CREATE PRIMARY INDEX IF NOT EXISTS ON tienda._default.pedidos;
CREATE PRIMARY INDEX IF NOT EXISTS ON tienda._default.productos;
CREATE PRIMARY INDEX IF NOT EXISTS ON tienda._default.blogs;
CREATE PRIMARY INDEX IF NOT EXISTS ON tienda._default.perfiles;
```

> **Nota:** Los índices primarios son adecuados para desarrollo y laboratorio. En producción se utilizan índices secundarios específicos.

---

## Pasos del Laboratorio

---

### Parte 1 — Data Nesting: Anidamiento de Documentos JSON

---

#### Paso 1.1 — Escenario A: Cliente con dirección embebida (relación uno-a-uno)

**Objetivo:** Implementar un objeto anidado que represente una relación uno-a-uno: un cliente con su dirección de envío principal embebida en el mismo documento.

**Instrucciones:**

1. Abre la pestaña **Query** en la Web Console de Couchbase (`http://localhost:8091`).

2. Inserta el siguiente documento usando una clave explícita con el patrón `tipo::ID`:

```sql
INSERT INTO tienda._default.clientes (KEY, VALUE)
VALUES (
  "cliente::1001",
  {
    "type": "cliente",
    "id": "cliente::1001",
    "nombre": "María López",
    "email": "maria.lopez@ejemplo.com",
    "telefono": "+52 55 1234 5678",
    "direccion_principal": {
      "calle": "Av. Reforma 450",
      "colonia": "Cuauhtémoc",
      "ciudad": "Ciudad de México",
      "estado": "CDMX",
      "codigo_postal": "06600",
      "pais": "México"
    },
    "fecha_registro": "2024-01-15",
    "activo": true
  }
);
```

3. Inserta un segundo cliente para tener datos comparativos:

```sql
INSERT INTO tienda._default.clientes (KEY, VALUE)
VALUES (
  "cliente::1002",
  {
    "type": "cliente",
    "id": "cliente::1002",
    "nombre": "Carlos Mendoza",
    "email": "carlos.mendoza@ejemplo.com",
    "telefono": "+52 33 9876 5432",
    "direccion_principal": {
      "calle": "Calle Morelos 120",
      "colonia": "Centro",
      "ciudad": "Guadalajara",
      "estado": "Jalisco",
      "codigo_postal": "44100",
      "pais": "México"
    },
    "fecha_registro": "2024-02-20",
    "activo": true
  }
);
```

4. Consulta el campo anidado directamente usando notación de punto:

```sql
SELECT c.nombre,
       c.direccion_principal.ciudad,
       c.direccion_principal.estado,
       c.direccion_principal.codigo_postal
FROM tienda._default.clientes AS c
WHERE c.type = "cliente";
```

**Salida esperada:**

```json
[
  {
    "ciudad": "Ciudad de México",
    "codigo_postal": "06600",
    "estado": "CDMX",
    "nombre": "María López"
  },
  {
    "ciudad": "Guadalajara",
    "codigo_postal": "44100",
    "estado": "Jalisco",
    "nombre": "Carlos Mendoza"
  }
]
```

**Verificación:**

```sql
-- Verificar acceso directo por key y campo anidado
SELECT META().id, c.direccion_principal.ciudad
FROM tienda._default.clientes AS c
USE KEYS ["cliente::1001", "cliente::1002"];
```

Debes obtener exactamente 2 documentos con sus ciudades respectivas.

---

#### Paso 1.2 — Escenario B: Pedido con artículos como arreglo embebido (relación uno-a-muchos)

**Objetivo:** Modelar una relación uno-a-muchos usando un arreglo de sub-documentos embebidos, y consultar los datos con `UNNEST`.

**Instrucciones:**

1. Inserta un documento de pedido con artículos anidados:

```sql
INSERT INTO tienda._default.pedidos (KEY, VALUE)
VALUES (
  "order::ORD-2024-001",
  {
    "type": "pedido",
    "id": "order::ORD-2024-001",
    "numero_orden": "ORD-2024-001",
    "cliente_id": "cliente::1001",
    "fecha_creacion": "2024-03-15",
    "estado": "enviado",
    "articulos": [
      {
        "producto_id": "prod::200",
        "nombre": "Teclado mecánico",
        "sku": "TEC-MECH-001",
        "cantidad": 1,
        "precio_unitario": 1200.00,
        "subtotal": 1200.00
      },
      {
        "producto_id": "prod::315",
        "nombre": "Mouse inalámbrico",
        "sku": "MOU-WIRE-002",
        "cantidad": 2,
        "precio_unitario": 450.00,
        "subtotal": 900.00
      },
      {
        "producto_id": "prod::410",
        "nombre": "Mousepad XL",
        "sku": "PAD-XL-003",
        "cantidad": 1,
        "precio_unitario": 280.00,
        "subtotal": 280.00
      }
    ],
    "subtotal": 2380.00,
    "impuestos": 380.80,
    "total": 2760.80,
    "direccion_envio": {
      "calle": "Av. Reforma 450",
      "ciudad": "Ciudad de México",
      "codigo_postal": "06600"
    }
  }
);
```

2. Usa `UNNEST` para "aplanar" el arreglo de artículos y consultar cada ítem individualmente:

```sql
SELECT p.numero_orden,
       p.estado,
       art.nombre        AS producto,
       art.cantidad,
       art.precio_unitario,
       art.subtotal
FROM tienda._default.pedidos AS p
UNNEST p.articulos AS art
WHERE p.type = "pedido"
ORDER BY art.precio_unitario DESC;
```

3. Calcula el total de artículos y el monto total por pedido usando agregación sobre el arreglo:

```sql
SELECT p.numero_orden,
       p.cliente_id,
       ARRAY_LENGTH(p.articulos)             AS total_lineas,
       SUM(art.cantidad)                     AS total_unidades,
       p.total
FROM tienda._default.pedidos AS p
UNNEST p.articulos AS art
WHERE p.type = "pedido"
GROUP BY p.numero_orden, p.cliente_id, p.total;
```

**Salida esperada (consulta UNNEST):**

```json
[
  { "cantidad": 1, "numero_orden": "ORD-2024-001", "precio_unitario": 1200, "producto": "Teclado mecánico", "estado": "enviado", "subtotal": 1200 },
  { "cantidad": 2, "numero_orden": "ORD-2024-001", "precio_unitario": 450,  "producto": "Mouse inalámbrico", "estado": "enviado", "subtotal": 900  },
  { "cantidad": 1, "numero_orden": "ORD-2024-001", "precio_unitario": 280,  "producto": "Mousepad XL",       "estado": "enviado", "subtotal": 280  }
]
```

**Verificación:**

```sql
-- Verificar que ANY...SATISFIES funciona sobre el arreglo anidado
SELECT META().id, p.numero_orden
FROM tienda._default.pedidos AS p
WHERE ANY art IN p.articulos SATISFIES art.precio_unitario > 1000 END;
```

Debe retornar el pedido `order::ORD-2024-001` porque tiene un artículo con precio > 1000.

---

#### Paso 1.3 — Escenario C: Producto con variantes como sub-documentos

**Objetivo:** Modelar un producto con múltiples variantes (talla, color, stock) como arreglo de sub-documentos anidados.

**Instrucciones:**

1. Inserta un producto con variantes:

```sql
INSERT INTO tienda._default.productos (KEY, VALUE)
VALUES (
  "prod::500",
  {
    "type": "producto",
    "id": "prod::500",
    "nombre": "Camiseta Deportiva Pro",
    "categoria": "ropa",
    "marca": "SportMax",
    "descripcion": "Camiseta de alto rendimiento con tecnología de secado rápido",
    "precio_base": 350.00,
    "variantes": [
      { "sku": "CAM-PRO-S-BLK", "talla": "S",  "color": "Negro",  "stock": 15, "precio_adicional": 0 },
      { "sku": "CAM-PRO-M-BLK", "talla": "M",  "color": "Negro",  "stock": 22, "precio_adicional": 0 },
      { "sku": "CAM-PRO-L-BLK", "talla": "L",  "color": "Negro",  "stock": 18, "precio_adicional": 0 },
      { "sku": "CAM-PRO-XL-BLK","talla": "XL", "color": "Negro",  "stock": 8,  "precio_adicional": 20 },
      { "sku": "CAM-PRO-S-WHT", "talla": "S",  "color": "Blanco", "stock": 12, "precio_adicional": 0 },
      { "sku": "CAM-PRO-M-WHT", "talla": "M",  "color": "Blanco", "stock": 20, "precio_adicional": 0 }
    ],
    "activo": true,
    "fecha_alta": "2024-01-10"
  }
);
```

2. Consulta variantes disponibles con stock mayor a 10:

```sql
SELECT p.nombre,
       v.sku,
       v.talla,
       v.color,
       v.stock,
       (p.precio_base + v.precio_adicional) AS precio_final
FROM tienda._default.productos AS p
UNNEST p.variantes AS v
WHERE p.type = "producto"
  AND v.stock > 10
ORDER BY v.talla, v.color;
```

3. Crea un índice de arreglo para búsquedas eficientes por SKU de variante:

```sql
CREATE INDEX idx_producto_variante_sku
ON tienda._default.productos
  (ALL ARRAY v.sku FOR v IN variantes END);
```

4. Verifica la búsqueda por SKU usando el índice:

```sql
SELECT META().id AS key_documento, p.nombre, v.talla, v.color
FROM tienda._default.productos AS p
UNNEST p.variantes AS v
WHERE v.sku = "CAM-PRO-M-BLK";
```

**Salida esperada:**

```json
[
  {
    "color": "Negro",
    "key_documento": "prod::500",
    "nombre": "Camiseta Deportiva Pro",
    "talla": "M"
  }
]
```

**Verificación:**

```sql
-- Confirmar que el índice fue creado
SELECT * FROM system:indexes
WHERE keyspace_id = "productos"
  AND name = "idx_producto_variante_sku";
```

---

#### Paso 1.4 — Escenario D: Blog post con comentarios (decisión por volumen)

**Objetivo:** Analizar cuándo embeber comentarios en el documento padre es viable y cuándo conviene separarlos. Implementar el límite de 10 comentarios embebidos.

**Instrucciones:**

1. Inserta un blog post con comentarios embebidos (hasta 10):

```sql
INSERT INTO tienda._default.blogs (KEY, VALUE)
VALUES (
  "blog::post::2024-001",
  {
    "type": "blog_post",
    "id": "blog::post::2024-001",
    "titulo": "Guía completa de modelado JSON en Couchbase",
    "autor_id": "cliente::1001",
    "autor_nombre": "María López",
    "contenido": "El modelado de datos en bases de datos documentales requiere un enfoque diferente al relacional...",
    "tags": ["couchbase", "json", "nosql", "modelado"],
    "fecha_publicacion": "2024-03-20",
    "estado": "publicado",
    "comentarios_count": 3,
    "comentarios_embebidos": [
      {
        "comentario_id": "com::001",
        "autor": "Carlos Mendoza",
        "texto": "Excelente artículo, muy claro el punto sobre UNNEST.",
        "fecha": "2024-03-21",
        "likes": 5
      },
      {
        "comentario_id": "com::002",
        "autor": "Laura Jiménez",
        "texto": "¿Cuándo recomendarías usar referencing en lugar de embedding?",
        "fecha": "2024-03-21",
        "likes": 2
      },
      {
        "comentario_id": "com::003",
        "autor": "Pedro García",
        "texto": "Muy útil, lo aplicaré en mi proyecto esta semana.",
        "fecha": "2024-03-22",
        "likes": 8
      }
    ],
    "comentarios_overflow": false,
    "nota_diseno": "Cuando comentarios_count > 10, mover a coleccion separada: blog_comentarios"
  }
);
```

2. Consulta el post y sus comentarios embebidos ordenados por likes:

```sql
SELECT b.titulo,
       b.autor_nombre,
       b.comentarios_count,
       com.autor        AS comentarista,
       com.texto,
       com.likes
FROM tienda._default.blogs AS b
UNNEST b.comentarios_embebidos AS com
WHERE b.type = "blog_post"
  AND b.id = "blog::post::2024-001"
ORDER BY com.likes DESC;
```

3. Inserta un documento de comentario separado para ilustrar el patrón de *overflow* (cuando se supera el límite de 10):

```sql
INSERT INTO tienda._default.blogs (KEY, VALUE)
VALUES (
  "blog::comentario::post::2024-001::com::011",
  {
    "type": "blog_comentario",
    "post_id": "blog::post::2024-001",
    "comentario_id": "com::011",
    "autor": "Sofía Ramírez",
    "texto": "Este es el comentario número 11, almacenado como documento separado por el límite de embedding.",
    "fecha": "2024-04-01",
    "likes": 1
  }
);
```

4. Consulta los comentarios separados de un post específico:

```sql
SELECT bc.comentario_id, bc.autor, bc.texto, bc.fecha
FROM tienda._default.blogs AS bc
WHERE bc.type = "blog_comentario"
  AND bc.post_id = "blog::post::2024-001"
ORDER BY bc.fecha;
```

**Reflexión de diseño** (anota en tu cuaderno o en un comentario SQL):

```sql
-- DECISIÓN DE DISEÑO:
-- Embebido (hasta 10 comentarios):
--   ✓ 1 operación de lectura para post + comentarios
--   ✓ Consulta simple, sin JOIN
--   ✗ Actualizaciones concurrentes en el mismo documento (conflictos)
--   ✗ Documento crece con cada comentario
--
-- Separado (más de 10 comentarios):
--   ✓ Documentos de tamaño controlado
--   ✓ Sin conflictos de escritura concurrente
--   ✗ 2 operaciones de lectura (post + comentarios)
--   ✗ Requiere índice sobre post_id para eficiencia
SELECT "Análisis de trade-off documentado" AS nota;
```

**Verificación:**

```sql
SELECT COUNT(*) AS total_documentos_blog
FROM tienda._default.blogs;
-- Debe retornar 2 (el post y el comentario separado)
```

---

#### Paso 1.5 — Escenario E: Perfil de usuario con historial de actividad (separación por volumen)

**Objetivo:** Demostrar la separación de historial de actividad en documentos independientes para evitar documentos de tamaño ilimitado.

**Instrucciones:**

1. Inserta el perfil de usuario con historial reciente embebido (últimas 5 acciones):

```sql
INSERT INTO tienda._default.perfiles (KEY, VALUE)
VALUES (
  "perfil::usuario::1001",
  {
    "type": "perfil_usuario",
    "usuario_id": "cliente::1001",
    "nombre_usuario": "mlopez",
    "nivel": "premium",
    "puntos_fidelidad": 1250,
    "preferencias": {
      "idioma": "es",
      "moneda": "MXN",
      "notificaciones_email": true,
      "notificaciones_push": false,
      "tema": "oscuro"
    },
    "historial_reciente": [
      { "accion": "login",          "timestamp": "2024-03-22T10:15:00Z", "ip": "192.168.1.10" },
      { "accion": "ver_producto",   "timestamp": "2024-03-22T10:16:30Z", "producto_id": "prod::500" },
      { "accion": "agregar_carrito","timestamp": "2024-03-22T10:17:45Z", "producto_id": "prod::500" },
      { "accion": "checkout",       "timestamp": "2024-03-22T10:20:00Z", "orden_id": "order::ORD-2024-001" },
      { "accion": "logout",         "timestamp": "2024-03-22T10:45:00Z", "ip": "192.168.1.10" }
    ],
    "politica_historial": "Solo se embeben las últimas 5 acciones. El historial completo está en la colección historial_actividad con key: historial::usuario::1001::YYYY-MM",
    "ultima_actualizacion": "2024-03-22T10:45:00Z"
  }
);
```

2. Inserta el documento de historial completo separado (patrón de partición por mes):

```sql
INSERT INTO tienda._default.perfiles (KEY, VALUE)
VALUES (
  "historial::usuario::1001::2024-03",
  {
    "type": "historial_actividad",
    "usuario_id": "cliente::1001",
    "periodo": "2024-03",
    "total_acciones": 47,
    "acciones": [
      { "accion": "login",        "timestamp": "2024-03-01T09:00:00Z" },
      { "accion": "ver_producto", "timestamp": "2024-03-01T09:02:00Z", "producto_id": "prod::200" },
      { "accion": "login",        "timestamp": "2024-03-22T10:15:00Z", "ip": "192.168.1.10" },
      { "accion": "logout",       "timestamp": "2024-03-22T10:45:00Z" }
    ],
    "nota": "Documento truncado para el laboratorio. En producción contendría las 47 acciones del mes."
  }
);
```

3. Consulta el perfil y su historial reciente:

```sql
SELECT p.nombre_usuario,
       p.nivel,
       p.puntos_fidelidad,
       act.accion,
       act.timestamp
FROM tienda._default.perfiles AS p
UNNEST p.historial_reciente AS act
WHERE p.type = "perfil_usuario"
  AND p.usuario_id = "cliente::1001"
ORDER BY act.timestamp DESC;
```

**Verificación:**

```sql
SELECT META().id, type, usuario_id
FROM tienda._default.perfiles
WHERE type IN ["perfil_usuario", "historial_actividad"];
```

Debe retornar 2 documentos: el perfil y el historial mensual.

---

### Parte 2 — Key Design: Estrategias de Diseño de Claves

---

#### Paso 2.1 — Comparación de las 4 estrategias de keys

**Objetivo:** Implementar y comparar cuatro estrategias de claves de documento, evaluando su impacto en consultas `USE KEYS` y en la legibilidad del modelo.

**Instrucciones:**

1. **Estrategia A — UUID puro:** Genera una clave completamente aleatoria usando `UUID()`:

```sql
-- Estrategia A: UUID puro (opaco, sin semántica)
INSERT INTO tienda._default.clientes (KEY, VALUE)
VALUES (
  UUID(),
  {
    "type": "cliente",
    "nombre": "Ana Torres",
    "email": "ana.torres@ejemplo.com",
    "estrategia_key": "A_UUID_puro",
    "nota": "La key es un UUID generado automáticamente. No es legible ni predecible."
  }
);

-- Recuperar el documento recién insertado (necesitamos buscar por campo, no por key)
SELECT META().id AS key_generada, c.nombre, c.estrategia_key
FROM tienda._default.clientes AS c
WHERE c.estrategia_key = "A_UUID_puro";
```

2. **Estrategia B — tipo::UUID:** Prefijo de tipo seguido de UUID:

```sql
-- Estrategia B: tipo::UUID (legible por tipo, único garantizado)
INSERT INTO tienda._default.clientes (KEY, VALUE)
VALUES (
  CONCAT("cliente::", UUID()),
  {
    "type": "cliente",
    "nombre": "Roberto Silva",
    "email": "roberto.silva@ejemplo.com",
    "estrategia_key": "B_tipo_UUID"
  }
);

SELECT META().id AS key_generada, c.nombre, c.estrategia_key
FROM tienda._default.clientes AS c
WHERE c.estrategia_key = "B_tipo_UUID";
```

3. **Estrategia C — tipo::campo-negocio:** Clave derivada de un campo de negocio único:

```sql
-- Estrategia C: tipo::campo-negocio (legible, predecible, acceso directo por key)
INSERT INTO tienda._default.clientes (KEY, VALUE)
VALUES (
  "cliente::email::lucia.herrera@ejemplo.com",
  {
    "type": "cliente",
    "nombre": "Lucía Herrera",
    "email": "lucia.herrera@ejemplo.com",
    "estrategia_key": "C_tipo_campo_negocio",
    "nota": "La key incluye el email. Permite GET directo si se conoce el email."
  }
);

-- Acceso directo por key (sin índice, O(1))
SELECT META().id, c.nombre, c.email
FROM tienda._default.clientes AS c
USE KEYS ["cliente::email::lucia.herrera@ejemplo.com"];
```

4. **Estrategia D — tipo::scope::ID (multi-tenant):** Clave compuesta con identificador de tenant:

```sql
-- Estrategia D: tipo::tenant::ID (multi-tenant, aislamiento lógico)
INSERT INTO tienda._default.clientes (KEY, VALUE)
VALUES (
  "cliente::tenant::mx::10050",
  {
    "type": "cliente",
    "tenant": "mx",
    "id_local": "10050",
    "nombre": "Fernando Castillo",
    "email": "f.castillo@empresa-mx.com",
    "estrategia_key": "D_multi_tenant",
    "nota": "La key incluye el tenant (mx). Permite aislar datos por organización."
  }
);

INSERT INTO tienda._default.clientes (KEY, VALUE)
VALUES (
  "cliente::tenant::us::10050",
  {
    "type": "cliente",
    "tenant": "us",
    "id_local": "10050",
    "nombre": "Fernando Castillo (US Branch)",
    "email": "f.castillo@empresa-us.com",
    "estrategia_key": "D_multi_tenant",
    "nota": "Mismo id_local=10050 pero diferente tenant. No hay colisión de keys."
  }
);

-- Acceso directo a ambos tenants
SELECT META().id AS key, c.nombre, c.tenant
FROM tienda._default.clientes AS c
USE KEYS ["cliente::tenant::mx::10050", "cliente::tenant::us::10050"];
```

**Salida esperada (Estrategia D):**

```json
[
  { "key": "cliente::tenant::mx::10050", "nombre": "Fernando Castillo",          "tenant": "mx" },
  { "key": "cliente::tenant::us::10050", "nombre": "Fernando Castillo (US Branch)", "tenant": "us" }
]
```

**Verificación — Tabla comparativa de estrategias:**

```sql
-- Resumen comparativo de todas las estrategias insertadas
SELECT META().id AS key_documento,
       c.nombre,
       c.estrategia_key,
       LENGTH(META().id) AS longitud_key
FROM tienda._default.clientes AS c
WHERE c.estrategia_key IS NOT MISSING
ORDER BY c.estrategia_key;
```

Anota los resultados en la siguiente tabla de análisis:

| Estrategia | Longitud típica | Legibilidad | Predecibilidad | Acceso USE KEYS directo | Riesgo de colisión |
|------------|-----------------|-------------|----------------|-------------------------|--------------------|
| A: UUID puro | 36 chars | ❌ Ninguna | ❌ No | ❌ No (debe buscarse) | ✅ Mínimo |
| B: tipo::UUID | ~43 chars | ✅ Por tipo | ❌ No | ❌ No | ✅ Mínimo |
| C: tipo::campo | Variable | ✅ Alta | ✅ Sí | ✅ Sí | ⚠️ Si el campo cambia |
| D: tipo::tenant::ID | Variable | ✅ Alta | ✅ Sí | ✅ Sí | ✅ Mínimo por tenant |

---

#### Paso 2.2 — Impacto de las keys en consultas USE KEYS

**Objetivo:** Medir la diferencia de rendimiento entre acceso por `USE KEYS` (lookup O(1)) y acceso por índice secundario.

**Instrucciones:**

1. Inserta un conjunto de pedidos con keys predecibles basadas en el número de orden:

```sql
INSERT INTO tienda._default.pedidos (KEY, VALUE)
VALUES ("order::ORD-2024-002", {
  "type": "pedido", "numero_orden": "ORD-2024-002",
  "cliente_id": "cliente::1002", "estado": "pendiente", "total": 890.00,
  "articulos": [{"producto_id": "prod::315", "nombre": "Mouse inalámbrico", "cantidad": 1, "precio_unitario": 890.00, "subtotal": 890.00}]
});

INSERT INTO tienda._default.pedidos (KEY, VALUE)
VALUES ("order::ORD-2024-003", {
  "type": "pedido", "numero_orden": "ORD-2024-003",
  "cliente_id": "cliente::1001", "estado": "entregado", "total": 560.00,
  "articulos": [{"producto_id": "prod::410", "nombre": "Mousepad XL", "cantidad": 2, "precio_unitario": 280.00, "subtotal": 560.00}]
});
```

2. Compara acceso por `USE KEYS` vs. acceso por campo indexado:

```sql
-- Método 1: USE KEYS (acceso directo al Key-Value store, sin índice)
SELECT META().id, p.numero_orden, p.estado, p.total
FROM tienda._default.pedidos AS p
USE KEYS ["order::ORD-2024-001", "order::ORD-2024-002", "order::ORD-2024-003"];
```

```sql
-- Método 2: Filtro por campo (requiere índice o full scan)
SELECT META().id, p.numero_orden, p.estado, p.total
FROM tienda._default.pedidos AS p
WHERE p.numero_orden IN ["ORD-2024-001", "ORD-2024-002", "ORD-2024-003"];
```

3. Verifica el plan de ejecución de cada consulta con `EXPLAIN`:

```sql
-- Plan para USE KEYS (debe mostrar "fetch" directo, sin IndexScan)
EXPLAIN
SELECT META().id, p.numero_orden, p.estado
FROM tienda._default.pedidos AS p
USE KEYS ["order::ORD-2024-001", "order::ORD-2024-002"];
```

```sql
-- Plan para filtro por campo (puede mostrar PrimaryScan si no hay índice secundario)
EXPLAIN
SELECT META().id, p.numero_orden, p.estado
FROM tienda._default.pedidos AS p
WHERE p.numero_orden IN ["ORD-2024-001", "ORD-2024-002"];
```

**Observación esperada:** El plan con `USE KEYS` mostrará un operador `Fetch` directo sin `IndexScan`, lo que confirma el acceso O(1) al Key-Value store. El filtro por campo mostrará `PrimaryScan` o `IndexScan` dependiendo de los índices disponibles.

**Verificación:**

```sql
SELECT COUNT(*) AS total_pedidos
FROM tienda._default.pedidos
WHERE type = "pedido";
-- Debe retornar 3
```

---

### Parte 3 — Trade-offs: Embedding vs. Referencing vs. Híbrido

---

#### Paso 3.1 — Implementar el mismo modelo con tres estrategias

**Objetivo:** Implementar el modelo de "cliente con historial de compras" usando tres estrategias distintas y medir el número de operaciones necesarias para los patrones de acceso más comunes.

**Instrucciones:**

1. **Estrategia 1 — Embedding total:** Todo en un solo documento:

```sql
INSERT INTO tienda._default.clientes (KEY, VALUE)
VALUES (
  "cliente::embed::2001",
  {
    "type": "cliente_embedding_total",
    "id": "cliente::embed::2001",
    "nombre": "Isabel Vargas",
    "email": "isabel.vargas@ejemplo.com",
    "direccion": {
      "calle": "Blvd. Kukulcán km 12",
      "ciudad": "Cancún",
      "estado": "Quintana Roo",
      "codigo_postal": "77500"
    },
    "historial_compras": [
      {
        "orden_id": "ORD-2023-100",
        "fecha": "2023-11-01",
        "total": 1500.00,
        "articulos": [
          { "nombre": "Teclado mecánico", "cantidad": 1, "precio": 1500.00 }
        ]
      },
      {
        "orden_id": "ORD-2024-050",
        "fecha": "2024-02-14",
        "total": 900.00,
        "articulos": [
          { "nombre": "Mouse inalámbrico", "cantidad": 2, "precio": 450.00 }
        ]
      }
    ],
    "estrategia": "embedding_total",
    "operaciones_lectura_perfil_completo": 1,
    "operaciones_nueva_orden": "1 (actualizar documento cliente)",
    "riesgo": "Documento crece indefinidamente con el historial"
  }
);
```

2. **Estrategia 2 — Referencing total:** Documentos separados con referencias:

```sql
-- Documento cliente (sin historial embebido)
INSERT INTO tienda._default.clientes (KEY, VALUE)
VALUES (
  "cliente::ref::2002",
  {
    "type": "cliente_referencing_total",
    "id": "cliente::ref::2002",
    "nombre": "Miguel Ángel Reyes",
    "email": "ma.reyes@ejemplo.com",
    "direccion": {
      "calle": "Calle 5 de Mayo 88",
      "ciudad": "Monterrey",
      "estado": "Nuevo León",
      "codigo_postal": "64000"
    },
    "estrategia": "referencing_total",
    "nota": "El historial de compras está en documentos separados con key: order::cliente::ref::2002::*"
  }
);

-- Documentos de órdenes separados con referencia al cliente
INSERT INTO tienda._default.pedidos (KEY, VALUE)
VALUES (
  "order::cliente::ref::2002::ORD-2023-200",
  {
    "type": "pedido_referencing",
    "orden_id": "ORD-2023-200",
    "cliente_id": "cliente::ref::2002",
    "fecha": "2023-12-10",
    "total": 2200.00,
    "articulos": [
      { "nombre": "Monitor 24 pulgadas", "cantidad": 1, "precio": 2200.00 }
    ]
  }
);

INSERT INTO tienda._default.pedidos (KEY, VALUE)
VALUES (
  "order::cliente::ref::2002::ORD-2024-080",
  {
    "type": "pedido_referencing",
    "orden_id": "ORD-2024-080",
    "cliente_id": "cliente::ref::2002",
    "fecha": "2024-03-05",
    "total": 650.00,
    "articulos": [
      { "nombre": "Auriculares Bluetooth", "cantidad": 1, "precio": 650.00 }
    ]
  }
);
```

3. **Estrategia 3 — Modelo híbrido:** Cliente con resumen embebido y detalle en documentos separados:

```sql
-- Documento cliente con resumen de últimas 3 órdenes embebido
INSERT INTO tienda._default.clientes (KEY, VALUE)
VALUES (
  "cliente::hybrid::2003",
  {
    "type": "cliente_hibrido",
    "id": "cliente::hybrid::2003",
    "nombre": "Valentina Cruz",
    "email": "v.cruz@ejemplo.com",
    "direccion": {
      "calle": "Paseo de la Reforma 250",
      "ciudad": "Ciudad de México",
      "estado": "CDMX",
      "codigo_postal": "06500"
    },
    "resumen_compras": {
      "total_ordenes": 5,
      "monto_total_historico": 8750.00,
      "ultima_compra": "2024-03-18",
      "ultimas_3_ordenes": [
        { "orden_id": "ORD-2024-090", "fecha": "2024-03-18", "total": 1800.00 },
        { "orden_id": "ORD-2024-055", "fecha": "2024-02-28", "total": 3200.00 },
        { "orden_id": "ORD-2023-310", "fecha": "2023-12-20", "total": 950.00 }
      ]
    },
    "estrategia": "hibrido",
    "nota": "Resumen embebido para vista rápida. Detalle completo en documentos separados."
  }
);
```

---

#### Paso 3.2 — Medir operaciones por patrón de acceso

**Objetivo:** Contar cuántas operaciones (lecturas de documento) requiere cada estrategia para los patrones de acceso más comunes.

**Instrucciones:**

1. **Patrón A — Ver perfil completo del cliente con historial:**

```sql
-- Estrategia Embedding: 1 operación
SELECT c.nombre, c.email, c.direccion, c.historial_compras
FROM tienda._default.clientes AS c
USE KEYS ["cliente::embed::2001"];

-- Estrategia Referencing: 2+ operaciones (cliente + órdenes)
SELECT c.nombre, c.email, c.direccion
FROM tienda._default.clientes AS c
USE KEYS ["cliente::ref::2002"];

SELECT p.orden_id, p.fecha, p.total
FROM tienda._default.pedidos AS p
WHERE p.cliente_id = "cliente::ref::2002"
ORDER BY p.fecha DESC;

-- Estrategia Híbrida: 1 operación para resumen, 2 para detalle completo
SELECT c.nombre, c.email, c.resumen_compras
FROM tienda._default.clientes AS c
USE KEYS ["cliente::hybrid::2003"];
```

2. **Patrón B — Actualizar solo la dirección del cliente:**

```sql
-- Las 3 estrategias requieren 1 operación de escritura
-- Embedding: actualizar subdocumento (sub-document API en producción)
UPDATE tienda._default.clientes
SET direccion.colonia = "Juárez"
WHERE META().id = "cliente::embed::2001";

-- Referencing: igual, 1 operación
UPDATE tienda._default.clientes
SET direccion.colonia = "Centro Histórico"
WHERE META().id = "cliente::ref::2002";

-- Híbrido: igual, 1 operación
UPDATE tienda._default.clientes
SET direccion.colonia = "Cuauhtémoc"
WHERE META().id = "cliente::hybrid::2003";
```

3. **Patrón C — Agregar una nueva orden:**

```sql
-- Estrategia Embedding: 1 operación (ARRAY_APPEND)
UPDATE tienda._default.clientes
SET historial_compras = ARRAY_APPEND(historial_compras, {
  "orden_id": "ORD-2024-NEW",
  "fecha": "2024-04-01",
  "total": 750.00,
  "articulos": [{"nombre": "Webcam HD", "cantidad": 1, "precio": 750.00}]
})
WHERE META().id = "cliente::embed::2001";

-- Estrategia Referencing: 1 operación (INSERT de nuevo documento pedido)
INSERT INTO tienda._default.pedidos (KEY, VALUE)
VALUES (
  "order::cliente::ref::2002::ORD-2024-NEW",
  {
    "type": "pedido_referencing",
    "orden_id": "ORD-2024-NEW",
    "cliente_id": "cliente::ref::2002",
    "fecha": "2024-04-01",
    "total": 750.00,
    "articulos": [{"nombre": "Webcam HD", "cantidad": 1, "precio": 750.00}]
  }
);

-- Estrategia Híbrida: 2 operaciones (INSERT pedido + UPDATE resumen en cliente)
INSERT INTO tienda._default.pedidos (KEY, VALUE)
VALUES (
  "order::cliente::hybrid::2003::ORD-2024-NEW",
  {
    "type": "pedido_hibrido",
    "orden_id": "ORD-2024-NEW",
    "cliente_id": "cliente::hybrid::2003",
    "fecha": "2024-04-01",
    "total": 750.00,
    "articulos": [{"nombre": "Webcam HD", "cantidad": 1, "precio": 750.00}]
  }
);

UPDATE tienda._default.clientes
SET resumen_compras.total_ordenes = resumen_compras.total_ordenes + 1,
    resumen_compras.monto_total_historico = resumen_compras.monto_total_historico + 750.00,
    resumen_compras.ultima_compra = "2024-04-01"
WHERE META().id = "cliente::hybrid::2003";
```

4. Documenta los resultados en la tabla de trade-offs:

```sql
-- Tabla resumen de trade-offs (ejecutar como referencia)
SELECT "Tabla de Trade-offs" AS titulo,
       [
         {"estrategia": "Embedding Total",   "lectura_perfil": 1, "actualizar_dir": 1, "nueva_orden": 1, "riesgo_principal": "Documento crece sin límite"},
         {"estrategia": "Referencing Total", "lectura_perfil": 2, "actualizar_dir": 1, "nueva_orden": 1, "riesgo_principal": "Múltiples lecturas para vista completa"},
         {"estrategia": "Híbrido",           "lectura_perfil": 1, "actualizar_dir": 1, "nueva_orden": 2, "riesgo_principal": "Consistencia entre resumen y detalle"}
       ] AS comparacion;
```

---

## Validación y Pruebas

Ejecuta las siguientes consultas de validación para confirmar que todos los documentos del laboratorio fueron creados correctamente:

```sql
-- 1. Contar documentos por tipo en cada colección
SELECT "clientes" AS coleccion, COUNT(*) AS total
FROM tienda._default.clientes
UNION ALL
SELECT "pedidos" AS coleccion, COUNT(*) AS total
FROM tienda._default.pedidos
UNION ALL
SELECT "productos" AS coleccion, COUNT(*) AS total
FROM tienda._default.productos
UNION ALL
SELECT "blogs" AS coleccion, COUNT(*) AS total
FROM tienda._default.blogs
UNION ALL
SELECT "perfiles" AS coleccion, COUNT(*) AS total
FROM tienda._default.perfiles;
```

**Resultado esperado mínimo:**

| coleccion  | total |
|------------|-------|
| clientes   | ≥ 8   |
| pedidos    | ≥ 6   |
| productos  | ≥ 1   |
| blogs      | ≥ 2   |
| perfiles   | ≥ 2   |

```sql
-- 2. Verificar que los índices de arreglo existen
SELECT name, keyspace_id, state
FROM system:indexes
WHERE keyspace_id IN ["clientes", "pedidos", "productos", "blogs", "perfiles"]
ORDER BY keyspace_id, name;
```

```sql
-- 3. Prueba de acceso USE KEYS con las 4 estrategias de key
SELECT META().id AS key,
       c.nombre,
       c.estrategia_key
FROM tienda._default.clientes AS c
USE KEYS [
  "cliente::1001",
  "cliente::1002",
  "cliente::email::lucia.herrera@ejemplo.com",
  "cliente::tenant::mx::10050",
  "cliente::tenant::us::10050"
];
-- Debe retornar exactamente 5 documentos
```

```sql
-- 4. Prueba de UNNEST sobre arreglo de artículos
SELECT COUNT(*) AS total_lineas_pedido
FROM tienda._default.pedidos AS p
UNNEST p.articulos AS art
WHERE p.type = "pedido";
-- Debe retornar al menos 5 (3 de ORD-2024-001 + 1 de ORD-2024-002 + 1 de ORD-2024-003)
```

```sql
-- 5. Prueba de ANY...SATISFIES sobre arreglo de variantes
SELECT META().id, p.nombre
FROM tienda._default.productos AS p
WHERE p.type = "producto"
  AND ANY v IN p.variantes SATISFIES v.stock < 10 END;
-- Debe retornar prod::500 (tiene variante XL con stock=8)
```

```sql
-- 6. Verificar modelo híbrido: resumen embebido + detalle separado
SELECT c.nombre,
       c.resumen_compras.total_ordenes,
       c.resumen_compras.ultima_compra
FROM tienda._default.clientes AS c
WHERE c.type = "cliente_hibrido";
```

---

## Resolución de Problemas

### Problema 1: Error "Document Not Found" al ejecutar USE KEYS

**Síntomas:** La consulta `SELECT ... USE KEYS ["cliente::1001"]` retorna un resultado vacío o el error `"Document not found"` aunque el documento fue insertado.

**Causa:** El documento fue insertado en una colección diferente a la que se está consultando, o el nombre del bucket/scope/colección en la consulta no coincide exactamente con donde se realizó el INSERT. Couchbase 7.x requiere especificar el path completo `bucket.scope.collection`.

**Solución:**

```sql
-- Verificar en qué colección existe el documento
SELECT META().id, c.nombre
FROM tienda._default.clientes AS c
USE KEYS ["cliente::1001"];

-- Si retorna vacío, buscar en todas las colecciones del bucket
SELECT META().id, c.type
FROM tienda._default.clientes AS c
WHERE META().id = "cliente::1001";

-- Verificar que la colección existe y tiene el documento
SELECT COUNT(*) FROM tienda._default.clientes;

-- Si el bucket se llama diferente, ajustar el path:
-- SELECT * FROM mi_bucket._default.clientes USE KEYS ["cliente::1001"];
```

Asegúrate de que el nombre del bucket en todas las consultas sea exactamente `tienda` (minúsculas). Verifica en **Web Console → Buckets** el nombre exacto del bucket.

---

### Problema 2: La consulta con UNNEST retorna 0 resultados aunque el documento tiene el arreglo

**Síntomas:** La consulta `SELECT ... FROM coleccion UNNEST campo_arreglo AS item WHERE ...` retorna 0 filas, pero al hacer `SELECT *` el documento aparece con el arreglo correctamente.

**Causa:** El nombre del campo del arreglo en la cláusula `UNNEST` no coincide exactamente con el nombre en el documento JSON (diferencia de mayúsculas/minúsculas o typo), o el campo del arreglo está vacío (`[]`) en algunos documentos, lo que hace que `UNNEST` (inner join por defecto) excluya esos documentos.

**Solución:**

```sql
-- Paso 1: Verificar el nombre exacto del campo
SELECT RAW OBJECT_NAMES(p)
FROM tienda._default.pedidos AS p
WHERE META().id = "order::ORD-2024-001";
-- Debe mostrar todos los campos del documento, incluyendo "articulos"

-- Paso 2: Verificar que el arreglo no está vacío
SELECT META().id, ARRAY_LENGTH(p.articulos) AS num_articulos
FROM tienda._default.pedidos AS p
WHERE p.type = "pedido";

-- Paso 3: Usar LEFT UNNEST para incluir documentos con arreglo vacío
SELECT p.numero_orden, art
FROM tienda._default.pedidos AS p
LEFT UNNEST p.articulos AS art
WHERE p.type = "pedido";

-- Paso 4: Asegurarse de que el alias en UNNEST coincide con el usado en SELECT
-- CORRECTO:
SELECT p.numero_orden, art.nombre
FROM tienda._default.pedidos AS p
UNNEST p.articulos AS art
WHERE p.type = "pedido";
-- INCORRECTO (alias diferente):
-- SELECT p.numero_orden, item.nombre  ← 'item' no está definido
-- FROM tienda._default.pedidos AS p
-- UNNEST p.articulos AS art           ← alias es 'art', no 'item'
```

---

## Limpieza del Entorno

> **Nota:** Ejecuta la limpieza solo si el instructor lo indica o si deseas reiniciar el laboratorio desde cero. Los documentos creados en este lab son necesarios para los laboratorios posteriores (Lab 09 y Lab 10).

```sql
-- Eliminar documentos de prueba de estrategias de key (opcionales)
DELETE FROM tienda._default.clientes
WHERE estrategia_key IS NOT MISSING;

-- Eliminar documentos de comparación de trade-offs
DELETE FROM tienda._default.clientes
WHERE type IN ["cliente_embedding_total", "cliente_referencing_total", "cliente_hibrido"];

DELETE FROM tienda._default.pedidos
WHERE type IN ["pedido_referencing", "pedido_hibrido"];

-- Eliminar índice de arreglo si se desea recrear
DROP INDEX tienda._default.productos.idx_producto_variante_sku IF EXISTS;
```

Si deseas eliminar **todos** los documentos del laboratorio y comenzar de nuevo:

```sql
-- PRECAUCIÓN: Elimina TODOS los documentos de las colecciones del lab
DELETE FROM tienda._default.clientes WHERE type LIKE "cliente%";
DELETE FROM tienda._default.pedidos   WHERE type LIKE "pedido%";
DELETE FROM tienda._default.productos WHERE type LIKE "producto%";
DELETE FROM tienda._default.blogs     WHERE type IN ["blog_post", "blog_comentario"];
DELETE FROM tienda._default.perfiles  WHERE type IN ["perfil_usuario", "historial_actividad"];
```

---

## Resumen

En este laboratorio implementaste los tres pilares del modelado de documentos JSON en Couchbase:

### Lo que aprendiste

| Área | Conceptos aplicados |
|------|---------------------|
| **Data Nesting** | Objetos anidados (uno-a-uno), arreglos de sub-documentos (uno-a-muchos), límite de embedding por volumen, separación de historial por partición temporal |
| **Key Design** | UUID puro, `tipo::UUID`, `tipo::campo-negocio`, `tipo::tenant::ID`; impacto en `USE KEYS` vs. índice secundario |
| **Trade-offs** | Embedding total vs. referencing total vs. híbrido; conteo de operaciones por patrón de acceso |
| **SQL++** | `UNNEST`, `ANY...SATISFIES`, `ARRAY_APPEND`, `ARRAY_LENGTH`, `USE KEYS`, `EXPLAIN`, índices de arreglo |

### Reglas de diseño clave

1. **Embebe cuando** los datos relacionados siempre se leen juntos, el sub-documento tiene tamaño acotado y las actualizaciones concurrentes son poco frecuentes.
2. **Separa cuando** el arreglo puede crecer sin límite, los sub-documentos se actualizan independientemente o necesitas consultar los sub-documentos sin el padre.
3. **Usa keys predecibles** (`tipo::campo-negocio`) cuando necesitas acceso directo O(1) por un campo de negocio conocido.
4. **Usa keys con UUID** cuando la unicidad es la prioridad y el acceso siempre ocurre a través de índices secundarios.
5. **El modelo híbrido** es el más común en producción: resumen embebido para lecturas rápidas, detalle en documentos separados para escrituras independientes.

### Recursos adicionales

- [Couchbase Data Modeling Best Practices](https://docs.couchbase.com/server/current/learn/data/data-modeling-best-practices.html)
- [SQL++ UNNEST Clause Reference](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/from.html#unnest)
- [Array Indexing in Couchbase](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/indexing-arrays.html)
- [Sub-Document API Overview](https://docs.couchbase.com/server/current/sdk/overview.html)
- [Key-Value Operations in Couchbase](https://docs.couchbase.com/server/current/learn/data/data-model.html)

---
