# Análisis de casos de uso NoSQL vs Relacional

## Metadatos

| Campo            | Detalle                          |
|------------------|----------------------------------|
| **Duración**     | 25 minutos                       |
| **Complejidad**  | Fácil                            |
| **Nivel Bloom**  | Aplicar (Apply)                  |
| **Modalidad**    | Análisis guiado (sin instalación)|
| **Dependencias** | Ninguna (primer laboratorio)     |

---

## Descripción General

Este laboratorio introduce los fundamentos conceptuales que justifican la existencia de las bases de datos NoSQL, partiendo del análisis de las limitaciones concretas de los sistemas relacionales frente a las demandas de las aplicaciones modernas. Los estudiantes trabajarán con fichas de escenarios reales —e-commerce, redes sociales, IoT y banca— para clasificar cada caso según el tipo de NoSQL más adecuado, aplicando el Teorema CAP como marco de evaluación. Finalmente, se analiza el patrón de **Polyglot Programming** mediante un diagrama de arquitectura de microservicios que combina múltiples tipos de bases de datos.

> **Nota:** Este laboratorio es exclusivamente de análisis y reflexión guiada. No se requiere instalación de software ni acceso a Couchbase Server. Todo el trabajo se realiza con papel, pizarra o un editor de texto.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio, el estudiante será capaz de:

- [ ] Identificar los cuellos de botella de un esquema relacional (JOINs costosos, esquemas rígidos, escalado vertical) y proponer una alternativa NoSQL justificada.
- [ ] Aplicar el Teorema CAP para clasificar sistemas de bases de datos NoSQL según sus garantías de consistencia, disponibilidad y tolerancia a particiones.
- [ ] Diferenciar los cuatro tipos principales de bases de datos NoSQL (clave-valor, documental, columnar, grafos) y seleccionar el tipo adecuado para escenarios de Big Data dados.
- [ ] Contrastar las características de Hadoop y NoSQL como estrategias complementarias para el manejo de Big Data.
- [ ] Describir el patrón de Polyglot Programming y justificar su uso en una arquitectura de microservicios.

---

## Prerrequisitos

### Conocimientos previos

| Área                         | Nivel requerido                                           |
|------------------------------|-----------------------------------------------------------|
| Bases de datos relacionales  | Básico (tablas, claves primarias/foráneas, normalización) |
| SQL                          | Básico (SELECT, JOIN, GROUP BY, WHERE)                    |
| Estructuras JSON             | Comprensión básica (objetos, arrays, anidamiento)         |
| Arquitectura de aplicaciones | Nociones generales (cliente-servidor, microservicios)     |

### Acceso y materiales

| Recurso                                       | Requerido |
|-----------------------------------------------|-----------|
| Editor de texto o papel/pizarra               | ✅ Sí     |
| Fichas de escenarios (Sección 5 de este lab)  | ✅ Sí     |
| Navegador web (para consultar referencias)    | Opcional  |
| Couchbase Server                              | ❌ No     |

---

## Entorno de Laboratorio

Este laboratorio no requiere hardware específico ni instalación de software. Se puede realizar íntegramente en papel o en cualquier editor de texto plano.

**Materiales recomendados:**
- Una copia impresa o digital de las **Fichas de Escenarios** (incluidas en el Paso 2).
- Una copia impresa o digital del **Diagrama del Teorema CAP** (incluido en el Paso 3).
- Papel cuadriculado o pizarra para dibujar el diagrama de arquitectura del Paso 5.

> **Para el instructor:** Se recomienda proyectar el diagrama del Teorema CAP y los cuatro tipos de NoSQL al inicio del laboratorio para que los estudiantes puedan referenciarlo durante los ejercicios. Si el laboratorio se realiza de forma remota, compartir este documento como PDF interactivo es suficiente.

---

## Pasos del Laboratorio

---

### Paso 1: Mapeo de Limitaciones Relacionales

**Objetivo:** Identificar los cuellos de botella concretos de un esquema relacional ante carga moderna y proponer una reestructuración NoSQL.

#### Instrucciones

1. Lee el siguiente escenario y el esquema relacional asociado.

2. Analiza el output de la consulta SQL y responde las preguntas de análisis.

3. Propón una alternativa de modelado NoSQL (tipo documental) que resuelva los problemas identificados.

---

**Escenario: Plataforma de Reservas "ViajeYA"**

ViajeYA es una plataforma de reservas de viajes que comenzó con 10 000 reservas diarias hace tres años. Tras una campaña viral, ahora procesa **2 millones de reservas diarias**. El equipo de ingeniería reporta los siguientes síntomas:

- Las búsquedas de disponibilidad tardan entre 4 y 8 segundos bajo carga pico.
- Las migraciones de esquema para añadir nuevos tipos de alojamiento requieren ventanas de mantenimiento nocturnas de 2–4 horas.
- El servidor principal opera al 95 % de CPU durante horas pico.
- Una réplica de lectura añadida recientemente introduce inconsistencias que confunden a los usuarios.

**Esquema relacional actual (simplificado):**

```sql
-- Tabla de hoteles: ~500,000 filas
CREATE TABLE hotels (
    id          INT PRIMARY KEY,
    name        VARCHAR(200),
    city        VARCHAR(100),
    country     VARCHAR(100),
    stars       INT,
    amenities   TEXT  -- JSON serializado como texto plano (no consultable)
);

-- Tabla de habitaciones: ~3,000,000 filas
CREATE TABLE rooms (
    id          INT PRIMARY KEY,
    hotel_id    INT REFERENCES hotels(id),
    type        VARCHAR(50),
    capacity    INT,
    price       DECIMAL(10,2)
);

-- Tabla de disponibilidad: ~45,000,000 filas
CREATE TABLE availability (
    id          INT PRIMARY KEY,
    room_id     INT REFERENCES rooms(id),
    date        DATE,
    is_available BOOLEAN
);
```

**Consulta problemática bajo carga:**

```sql
-- Búsqueda de disponibilidad en Barcelona del 20 al 27 de diciembre
-- Tiempo de ejecución: ~6.8 segundos bajo carga pico

EXPLAIN ANALYZE
SELECT h.name, r.type, r.price
FROM hotels h
JOIN rooms r ON h.id = r.hotel_id
JOIN availability a ON r.id = a.room_id
WHERE a.date BETWEEN '2024-12-20' AND '2024-12-27'
  AND h.city = 'Barcelona';

/*
Output del EXPLAIN ANALYZE:
Nested Loop  (cost=0.00..98432.17 rows=1243 width=64)
  -> Seq Scan on availability  (rows=45,000,000)  <-- PROBLEMA CRÍTICO
  -> Index Scan on rooms       (rows=3,000,000)
  -> Filter on hotels.city     (rows=500,000)
Planning Time:   3.2 ms
Execution Time:  6847.4 ms   <-- Inaceptable para el usuario final
*/
```

#### Preguntas de Análisis — Paso 1

Responde las siguientes preguntas en tu cuaderno o editor de texto:

**1.1.** ¿Cuál es el principal cuello de botella identificado en el plan de ejecución? Nombra la operación específica y explica por qué es costosa.

**1.2.** El campo `amenities` está almacenado como `TEXT` (JSON serializado). ¿Qué limitaciones impone esto para las consultas? Da un ejemplo concreto de una consulta que sería imposible o muy ineficiente.

**1.3.** El equipo propone añadir un nuevo tipo de alojamiento: "glamping" (tiendas de campaña de lujo) con campos adicionales como `tent_material`, `max_wind_speed` y `eco_certification`. ¿Qué operación SQL se requeriría? ¿Cuál sería el impacto en producción?

**1.4.** Completa la siguiente tabla identificando cada problema y su causa raíz:

| Síntoma observado                          | Causa raíz en el modelo relacional | Categoría (Escala / Velocidad / Variedad) |
|--------------------------------------------|-------------------------------------|-------------------------------------------|
| Búsquedas tardan 6–8 segundos              |                                     |                                           |
| Ventanas de mantenimiento de 2–4 horas     |                                     |                                           |
| CPU al 95% en horas pico                   |                                     |                                           |
| Inconsistencias con la réplica de lectura  |                                     |                                           |

#### Propuesta de Reestructuración NoSQL — Paso 1

**1.5.** Diseña un documento JSON que represente un hotel con sus habitaciones y disponibilidad de forma desnormalizada. El documento debe:
- Incluir al menos 3 campos del hotel (nombre, ciudad, estrellas).
- Incluir un array de habitaciones con sus atributos.
- Incluir el campo `amenities` como un objeto JSON estructurado (no como texto plano).
- Incluir la disponibilidad como un array de fechas disponibles (en lugar de una tabla separada).

Usa el siguiente esqueleto como punto de partida:

```json
{
  "type": "hotel",
  "id": "hotel_001",
  "name": "...",
  "location": {
    "city": "...",
    "country": "..."
  },
  "stars": 4,
  "amenities": {
    // Completa con al menos 4 amenidades como campos booleanos o numéricos
  },
  "rooms": [
    {
      "room_id": "r001",
      "type": "...",
      "capacity": 2,
      "price_per_night": 0.0,
      "available_dates": ["2024-12-20", "2024-12-21"]
      // Añade campos adicionales si el tipo es "glamping"
    }
  ]
}
```

**Resultado esperado del Paso 1:**

- Tabla de análisis completada con 4 síntomas, causas raíz y categorías correctamente identificadas.
- Documento JSON propuesto con estructura desnormalizada que elimina la necesidad de 3 JOINs.
- Identificación correcta del `Seq Scan on availability` como el principal cuello de botella.

**Verificación del Paso 1:**

Compara tu documento JSON con el siguiente criterio de evaluación:

| Criterio                                              | Cumplido |
|-------------------------------------------------------|----------|
| El documento JSON es válido (sin errores de sintaxis) | ☐        |
| `amenities` es un objeto, no una cadena de texto      | ☐        |
| Las habitaciones están como array anidado             | ☐        |
| La disponibilidad está integrada en el documento      | ☐        |
| El tipo "glamping" tiene campos adicionales propios   | ☐        |

---

### Paso 2: Clasificación de Escenarios por Tipo de NoSQL

**Objetivo:** Aplicar el conocimiento de los cuatro tipos de bases de datos NoSQL para seleccionar la tecnología más adecuada para cada escenario de negocio.

#### Marco de Referencia: Los Cuatro Tipos de NoSQL

Antes de analizar los escenarios, revisa la siguiente tabla de referencia:

| Tipo            | Modelo de datos              | Fortaleza principal                              | Ejemplo de tecnología     | Caso de uso típico                          |
|-----------------|------------------------------|--------------------------------------------------|---------------------------|---------------------------------------------|
| **Clave-Valor** | Par key → value (opaco)      | Lecturas/escrituras ultra-rápidas (O(1))         | Redis, DynamoDB, Couchbase| Caché de sesiones, carritos de compra       |
| **Documental**  | Documentos JSON/BSON         | Flexibilidad de esquema, consultas ricas         | Couchbase, MongoDB        | Catálogos, perfiles de usuario, contenido   |
| **Columnar**    | Familias de columnas         | Escrituras masivas, series temporales            | Apache Cassandra, HBase   | IoT, logs, métricas de telemetría           |
| **Grafos**      | Nodos, aristas, propiedades  | Traversal de relaciones complejas                | Neo4j, Amazon Neptune     | Redes sociales, detección de fraude         |

#### Instrucciones

1. Lee cada una de las siguientes **8 fichas de escenarios**.
2. Para cada ficha, selecciona el **tipo de NoSQL más adecuado** de la tabla de referencia anterior.
3. Justifica tu elección en **2–3 oraciones** mencionando al menos una fortaleza técnica del tipo elegido que resuelve el problema del escenario.
4. Indica si el escenario podría beneficiarse de un enfoque **Polyglot** (usando más de un tipo de base de datos) y por qué.

---

**Ficha A — E-Commerce: Carrito de Compras**

> Una tienda en línea con 500 000 usuarios activos simultáneos necesita almacenar el carrito de compras de cada usuario. El carrito se actualiza con cada clic (añadir/eliminar producto), tiene una vida útil de 24 horas si el usuario no completa la compra, y debe recuperarse en menos de 10 ms para no impactar la experiencia de checkout. El carrito promedio contiene 3–8 ítems.

**Ficha B — Red Social: Grafo de Amigos**

> Una red social profesional necesita calcular "personas que quizás conozcas" para sus 50 millones de usuarios. El algoritmo requiere encontrar amigos de amigos (2 saltos en el grafo), filtrar por industria y ciudad, y devolver resultados en menos de 500 ms. La red de conexiones crece en ~100 000 nuevas conexiones por día.

**Ficha C — IoT: Telemetría de Sensores**

> Una empresa de manufactura tiene 10 000 sensores industriales que envían lecturas de temperatura, presión y vibración cada 5 segundos. Los datos se almacenan durante 2 años para análisis de mantenimiento predictivo. Las escrituras son masivas y continuas; las lecturas son principalmente por rango de tiempo para un sensor específico.

**Ficha D — Banca: Perfil de Cliente 360°**

> Un banco necesita almacenar el perfil completo de sus clientes: datos personales, historial de transacciones, productos contratados (cuenta corriente, tarjetas, hipotecas), preferencias de comunicación y notas del asesor. Cada cliente tiene un perfil diferente (algunos tienen 1 producto, otros tienen 15). El perfil se consulta íntegramente en cada interacción con el call center.

**Ficha E — Streaming: Historial de Visualización**

> Una plataforma de streaming necesita registrar qué contenido ha visto cada usuario, en qué minuto pausó, su calificación y si lo completó. Con 80 millones de usuarios y un promedio de 2 horas de visualización diaria, se generan ~160 millones de eventos por día. El caso de uso principal es recuperar el historial de un usuario para mostrar "continuar viendo".

**Ficha F — E-Commerce: Catálogo de Productos**

> Una tienda multimarca vende productos de 200 categorías diferentes: electrónica, ropa, alimentos, muebles, etc. Cada categoría tiene atributos completamente distintos (un televisor tiene resolución y HDR; una camiseta tiene talla y material; un queso tiene origen y curación). Se añaden ~500 nuevos productos por día con esquemas variables. Las búsquedas incluyen filtros por atributos específicos de cada categoría.

**Ficha G — Logística: Seguimiento de Envíos**

> Una empresa de logística rastrea 2 millones de paquetes activos. Cada paquete tiene un identificador único y su estado se actualiza en cada punto de la cadena de distribución (recogida, centro de distribución, en tránsito, entregado). Los clientes consultan el estado de su paquete por ID. Las actualizaciones de estado son frecuentes (cada 15–30 minutos por paquete activo).

**Ficha H — Detección de Fraude: Análisis de Transacciones**

> Un sistema antifraude necesita detectar patrones como: "este usuario realizó 3 transacciones en 3 países diferentes en 10 minutos" o "esta tarjeta fue usada en dos comercios a 500 km de distancia simultáneamente". El análisis requiere recorrer el historial de transacciones del usuario y sus relaciones con comercios, dispositivos y ubicaciones.

#### Tabla de Respuestas — Paso 2

Completa la siguiente tabla:

| Ficha | Escenario                         | Tipo NoSQL elegido | Justificación (2–3 oraciones) | ¿Polyglot? ¿Por qué? |
|-------|-----------------------------------|--------------------|-------------------------------|----------------------|
| A     | Carrito de compras                |                    |                               |                      |
| B     | Grafo de amigos                   |                    |                               |                      |
| C     | Telemetría IoT                    |                    |                               |                      |
| D     | Perfil de cliente 360°            |                    |                               |                      |
| E     | Historial de visualización        |                    |                               |                      |
| F     | Catálogo de productos             |                    |                               |                      |
| G     | Seguimiento de envíos             |                    |                               |                      |
| H     | Detección de fraude               |                    |                               |                      |

**Resultado esperado del Paso 2:**

- Las 8 fichas clasificadas con el tipo de NoSQL correcto.
- Al menos 4 fichas identificadas como candidatas a enfoque Polyglot con justificación válida.
- Justificaciones que mencionen características técnicas específicas (no solo "es más rápido").

**Verificación del Paso 2:**

Compara tus respuestas con la siguiente guía de referencia (los instructores pueden revelarla al final):

| Ficha | Tipo primario recomendado | Tipo secundario (Polyglot)    |
|-------|---------------------------|-------------------------------|
| A     | Clave-Valor               | Documental (para persistencia)|
| B     | Grafos                    | Clave-Valor (caché)           |
| C     | Columnar                  | Documental (metadatos sensor) |
| D     | Documental                | Relacional (transacciones ACID)|
| E     | Clave-Valor / Columnar    | Documental (metadatos)        |
| F     | Documental                | Full-Text Search              |
| G     | Clave-Valor               | Columnar (historial eventos)  |
| H     | Grafos                    | Columnar (historial transacc.)|

> **Nota pedagógica:** En varios casos existe más de una respuesta válida. Lo importante es que la justificación sea coherente con las características técnicas del tipo elegido.

---

### Paso 3: Aplicación del Teorema CAP

**Objetivo:** Aplicar el Teorema CAP para evaluar y clasificar sistemas NoSQL según sus garantías, y comprender los trade-offs de diseño que implica cada elección.

#### Marco de Referencia: El Teorema CAP

El Teorema CAP (Brewer, 2000) establece que un sistema distribuido **no puede garantizar simultáneamente** las tres propiedades siguientes:

```
         Consistencia (C)
         ─────────────────
         Todos los nodos ven
         los mismos datos al
         mismo tiempo
              /\
             /  \
            /    \
           /  ¿?  \
          /        \
         ────────────────────────────────
        /                                \
Disponibilidad (A)              Tolerancia a Particiones (P)
─────────────────────           ─────────────────────────────
Cada petición recibe            El sistema continúa operando
una respuesta (éxito            aunque haya fallas de red
o fallo)                        entre nodos
```

**Los tres vértices del triángulo CAP:**

| Propiedad                      | Definición                                                                                  | Implicación práctica                                         |
|--------------------------------|---------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| **C — Consistencia**           | Todos los nodos devuelven el dato más reciente en cualquier momento                         | Escrituras pueden bloquearse hasta que todos los nodos confirmen |
| **A — Disponibilidad**         | El sistema siempre responde (aunque no sea el dato más reciente)                            | Se aceptan lecturas potencialmente desactualizadas            |
| **P — Tolerancia a particiones**| El sistema sigue funcionando aunque algunos nodos no puedan comunicarse entre sí            | Obligatorio en sistemas distribuidos reales                   |

> **Nota importante:** En la práctica, **P es casi siempre obligatorio** en sistemas distribuidos (las fallas de red son inevitables). Por ello, la elección real es entre **CP** (consistencia + tolerancia a particiones) o **AP** (disponibilidad + tolerancia a particiones).

**Clasificación CAP de sistemas conocidos:**

| Sistema           | Clasificación CAP | Razón                                                              |
|-------------------|-------------------|--------------------------------------------------------------------|
| Apache HBase      | CP                | Prioriza consistencia; puede no responder durante particiones      |
| Apache Cassandra  | AP                | Siempre responde; acepta consistencia eventual                     |
| MongoDB           | CP (configurable) | Por defecto prioriza consistencia; réplicas pueden quedar detrás   |
| Couchbase         | AP / CP           | Configurable por operación: `majority`, `none`, `linearizable`     |
| Redis             | CP (single node)  | En cluster, puede ser AP dependiendo de la configuración           |
| DynamoDB          | AP (por defecto)  | Consistencia eventual por defecto; consistencia fuerte opcional    |
| Zookeeper         | CP                | Diseñado para coordinación; prioriza consistencia                  |

#### Instrucciones

**3.1.** Para cada uno de los siguientes 5 escenarios, determina si el sistema debería priorizarse como **CP** o **AP** y justifica tu elección:

---

**Escenario CAP-1: Sistema de transferencias bancarias**

> Un banco procesa transferencias entre cuentas. Si la misma cuenta se consulta desde dos cajeros automáticos simultáneamente, ¿es aceptable que uno muestre un saldo desactualizado?

- **Tu clasificación:** CP / AP
- **Justificación:**

---

**Escenario CAP-2: Feed de noticias de una red social**

> Un usuario publica una foto. Sus 500 seguidores la verán en sus feeds. ¿Es crítico que todos la vean exactamente al mismo tiempo, o es aceptable un retraso de 1–2 segundos?

- **Tu clasificación:** CP / AP
- **Justificación:**

---

**Escenario CAP-3: Inventario de productos en Black Friday**

> Una tienda tiene 5 unidades de un producto muy demandado. Durante Black Friday, miles de usuarios intentan comprarlo simultáneamente. ¿Qué es más costoso: vender más unidades de las que existen (overselling) o que algunos usuarios no puedan completar su compra?

- **Tu clasificación:** CP / AP
- **Justificación:**

---

**Escenario CAP-4: Marcador en tiempo real de un videojuego**

> Un videojuego multijugador muestra el marcador global de todos los jugadores. ¿Es crítico que el marcador sea exacto al milisegundo, o es aceptable una actualización cada 2–3 segundos?

- **Tu clasificación:** CP / AP
- **Justificación:**

---

**Escenario CAP-5: Sistema de reservas de asientos en un vuelo**

> Una aerolínea vende los últimos 3 asientos de un vuelo. Dos agentes de viaje en ciudades diferentes intentan reservar el mismo asiento simultáneamente. ¿Qué garantía debe tener el sistema?

- **Tu clasificación:** CP / AP
- **Justificación:**

---

**3.2.** Completa el siguiente diagrama clasificando los sistemas de la tabla de referencia en el triángulo CAP:

```
                    C (Consistencia)
                         △
                        /|\
                       / | \
                      /  |  \
                     / CP|CP \
                    /    |    \
                   /     |     \
                  /   [aquí]   \
                 /              \
                /________________\
         AP   /                    \  CP
             /      [aquí]          \
            /                        \
           ──────────────────────────────
          A (Disponibilidad)    P (Tolerancia)

Clasifica: HBase, Cassandra, MongoDB, Couchbase, DynamoDB, Zookeeper
```

**3.3.** Responde la siguiente pregunta de reflexión:

> Couchbase permite configurar el nivel de consistencia **por operación** (no a nivel global del clúster). ¿Qué ventaja arquitectónica ofrece esto para una aplicación como ViajeYA que tiene tanto búsquedas de disponibilidad (tolerantes a datos ligeramente desactualizados) como confirmaciones de reserva (que requieren consistencia fuerte)?

**Resultado esperado del Paso 3:**

- 5 escenarios CAP clasificados correctamente con justificaciones coherentes.
- Diagrama CAP completado con los 6 sistemas en sus posiciones correctas.
- Respuesta de reflexión que mencione el concepto de consistencia configurable por operación.

**Verificación del Paso 3:**

| Escenario CAP | Clasificación correcta | Razonamiento clave                                      |
|---------------|------------------------|---------------------------------------------------------|
| CAP-1 (Banca) | **CP**                 | El overselling o doble débito es inaceptable            |
| CAP-2 (Feed)  | **AP**                 | Consistencia eventual es aceptable; disponibilidad > exactitud |
| CAP-3 (Inventario Black Friday) | **CP** | El overselling tiene costo financiero y reputacional |
| CAP-4 (Marcador videojuego) | **AP**     | Retraso de segundos es tolerable; disponibilidad crítica|
| CAP-5 (Asientos vuelo) | **CP**          | Doble reserva del mismo asiento es inaceptable          |

---

### Paso 4: Hadoop vs NoSQL — Estrategias Complementarias

**Objetivo:** Contrastar Hadoop y NoSQL como tecnologías con propósitos diferentes pero complementarios para el manejo de Big Data.

#### Marco de Referencia

Una confusión frecuente es pensar que Hadoop y NoSQL son tecnologías competidoras. En realidad, tienen perfiles de uso muy diferentes:

| Dimensión                  | Hadoop (HDFS + MapReduce/Spark)                    | NoSQL (Couchbase, Cassandra, etc.)               |
|----------------------------|----------------------------------------------------|--------------------------------------------------|
| **Latencia de acceso**     | Alta (segundos a minutos)                          | Baja (milisegundos)                              |
| **Patrón de acceso**       | Batch (procesar grandes volúmenes)                 | Online (consultas puntuales en tiempo real)      |
| **Modelo de datos**        | Archivos (HDFS), tablas (Hive)                     | Documentos, clave-valor, columnar, grafos        |
| **Escalado**               | Horizontal (miles de nodos)                        | Horizontal (decenas a cientos de nodos)          |
| **Caso de uso principal**  | ETL, análisis histórico, machine learning          | Aplicaciones transaccionales, APIs, caché        |
| **Consistencia**           | No aplica (batch)                                  | Configurable (eventual a fuerte)                 |
| **Mutabilidad**            | Datos inmutables (append-only)                     | Datos mutables (CRUD completo)                   |

#### Instrucciones

**4.1.** Lee el siguiente escenario y determina qué tecnología (Hadoop, NoSQL, o ambas) es más adecuada para cada sub-tarea:

**Escenario: Plataforma de Análisis de Comportamiento de Usuario "InsightApp"**

InsightApp recopila eventos de comportamiento de 10 millones de usuarios: clics, páginas visitadas, tiempo en página, conversiones. Tiene los siguientes requerimientos:

| Sub-tarea | Descripción | Tecnología recomendada | Justificación |
|-----------|-------------|------------------------|---------------|
| **T1** | Mostrar al usuario sus últimas 10 actividades en tiempo real (< 50 ms) | | |
| **T2** | Calcular el funnel de conversión mensual de los últimos 12 meses (análisis histórico, puede tardar minutos) | | |
| **T3** | Almacenar el perfil de preferencias del usuario para personalización en tiempo real | | |
| **T4** | Entrenar un modelo de machine learning con 2 años de datos históricos de eventos | | |
| **T5** | Detectar en tiempo real si un usuario está en el punto de abandono del carrito (< 100 ms) | | |
| **T6** | Generar un reporte semanal de los 100 productos más vistos por categoría | | |

**4.2.** Dibuja o describe con texto un diagrama de arquitectura Lambda que combine Hadoop y NoSQL para InsightApp. El diagrama debe incluir:

```
[Fuente de eventos] → [Capa de ingesta] → ┌─ [Batch Layer: Hadoop/Spark] → [Serving Layer NoSQL]
                                           └─ [Speed Layer: NoSQL en tiempo real]
                                                        ↓
                                              [Capa de consulta unificada]
```

Describe brevemente el rol de cada componente (2–3 oraciones por capa).

**4.3.** Responde: ¿En qué situación elegiría un equipo de ingeniería **solo NoSQL** sin Hadoop? ¿Y cuándo necesitaría ambos? Da un ejemplo concreto de cada caso.

**Resultado esperado del Paso 4:**

- Tabla T1–T6 completada con tecnología correcta y justificación coherente.
- Descripción del diagrama Lambda con roles claros para cada capa.
- Respuesta clara sobre cuándo usar solo NoSQL vs. arquitectura híbrida.

**Verificación del Paso 4:**

| Sub-tarea | Respuesta esperada             |
|-----------|-------------------------------|
| T1        | NoSQL (baja latencia requerida)|
| T2        | Hadoop/Spark (batch histórico) |
| T3        | NoSQL (acceso en tiempo real)  |
| T4        | Hadoop/Spark (ML sobre big data)|
| T5        | NoSQL (tiempo real < 100 ms)   |
| T6        | Hadoop/Spark (batch semanal)   |

---

### Paso 5: Polyglot Programming — Arquitectura de Microservicios

**Objetivo:** Analizar el patrón de Polyglot Programming y justificar la selección de diferentes tipos de bases de datos para distintos microservicios en una arquitectura real.

#### Marco de Referencia: Polyglot Programming / Polyglot Persistence

El concepto de **Polyglot Persistence** (Fowler & Sadalage, 2012) establece que diferentes partes de una aplicación pueden y deben usar diferentes tipos de almacenamiento según sus necesidades específicas, en lugar de forzar todos los datos en un único sistema.

```
┌─────────────────────────────────────────────────────────────────┐
│                    APLICACIÓN E-COMMERCE                         │
├──────────────┬──────────────┬──────────────┬────────────────────┤
│  Servicio    │  Servicio    │  Servicio    │  Servicio          │
│  Catálogo    │  Sesiones    │  Búsqueda    │  Recomendaciones   │
│              │  & Carrito   │  de Texto    │                    │
├──────────────┼──────────────┼──────────────┼────────────────────┤
│  Couchbase   │   Redis      │ Elasticsearch│     Neo4j          │
│  (Documental)│ (Clave-Valor)│ (Full-Text)  │    (Grafos)        │
└──────────────┴──────────────┴──────────────┴────────────────────┘
```

#### Instrucciones

**5.1.** Analiza la siguiente arquitectura de microservicios para una plataforma de viajes y completa la tabla de selección de tecnología:

**Plataforma "TravelCloud" — Microservicios:**

| Microservicio              | Descripción del requerimiento de datos                                                                                        | Tipo de BD recomendado | Tecnología específica | Justificación |
|----------------------------|-------------------------------------------------------------------------------------------------------------------------------|------------------------|-----------------------|---------------|
| **Auth & Sesiones**        | Tokens JWT de sesión, expiran en 24h, acceso por ID de usuario, ~1M usuarios concurrentes                                    |                        |                       |               |
| **Catálogo de Hoteles**    | Documentos flexibles por tipo de alojamiento (hotel, hostel, glamping), búsqueda por ciudad/precio/amenidades                 |                        |                       |               |
| **Motor de Búsqueda**      | Búsqueda de texto libre ("hotel romántico cerca del mar en Barcelona"), autocompletado, relevancia por puntuación             |                        |                       |               |
| **Reservas & Pagos**       | Transacciones ACID, historial de pagos, integridad referencial crítica, auditoría regulatoria                                 |                        |                       |               |
| **Recomendaciones**        | "Usuarios que reservaron X también reservaron Y", análisis de grafo de comportamiento, cálculo de similitud                   |                        |                       |               |
| **Telemetría & Logs**      | Eventos de click, errores, métricas de rendimiento; 50M eventos/día; consultas por rango de tiempo                           |                        |                       |               |
| **Perfil de Usuario**      | Preferencias, historial de viajes, documentos de identidad, notificaciones configuradas; esquema variable por tipo de usuario  |                        |                       |               |

**5.2.** Identifica **3 ventajas** y **3 desventajas** del enfoque Polyglot Persistence para TravelCloud:

| Ventajas del enfoque Polyglot                | Desventajas / Desafíos del enfoque Polyglot   |
|----------------------------------------------|-----------------------------------------------|
| 1.                                           | 1.                                            |
| 2.                                           | 2.                                            |
| 3.                                           | 3.                                            |

**5.3.** Couchbase se posiciona como una plataforma que puede cubrir múltiples roles en una arquitectura Polyglot (Documental + Clave-Valor + Full-Text Search + Analytics en un solo sistema). ¿Qué ventajas operativas ofrece esto frente a mantener 4 sistemas separados? ¿Qué trade-offs implica?

**Resultado esperado del Paso 5:**

- Tabla de 7 microservicios completada con tipo de BD, tecnología específica y justificación.
- 3 ventajas y 3 desventajas del Polyglot Persistence identificadas correctamente.
- Reflexión sobre el rol de Couchbase como plataforma multi-modelo.

**Verificación del Paso 5:**

| Microservicio         | Tipo recomendado               | Ejemplo de tecnología      |
|-----------------------|-------------------------------|----------------------------|
| Auth & Sesiones       | Clave-Valor                   | Redis, Couchbase KV        |
| Catálogo de Hoteles   | Documental                    | Couchbase, MongoDB         |
| Motor de Búsqueda     | Full-Text Search              | Elasticsearch, Couchbase FTS|
| Reservas & Pagos      | Relacional (ACID)             | PostgreSQL, MySQL           |
| Recomendaciones       | Grafos                        | Neo4j, Amazon Neptune      |
| Telemetría & Logs     | Columnar                      | Cassandra, InfluxDB         |
| Perfil de Usuario     | Documental                    | Couchbase, MongoDB         |

---

## Validación y Verificación Final

Al completar todos los pasos, verifica que hayas producido los siguientes entregables:

| Entregable                                                   | Paso   | Completado |
|--------------------------------------------------------------|--------|------------|
| Tabla de análisis de cuellos de botella (4 síntomas)         | Paso 1 | ☐          |
| Documento JSON desnormalizado para hotel ViajeYA             | Paso 1 | ☐          |
| Tabla de clasificación NoSQL para 8 fichas de escenarios     | Paso 2 | ☐          |
| 5 escenarios CAP clasificados con justificación              | Paso 3 | ☐          |
| Diagrama CAP con 6 sistemas clasificados                     | Paso 3 | ☐          |
| Tabla Hadoop vs NoSQL para 6 sub-tareas de InsightApp        | Paso 4 | ☐          |
| Tabla de microservicios TravelCloud con selección de BD      | Paso 5 | ☐          |
| Tabla de ventajas/desventajas Polyglot Persistence           | Paso 5 | ☐          |

**Criterios de evaluación global:**

| Criterio                                                                        | Puntos |
|---------------------------------------------------------------------------------|--------|
| Identificación correcta de cuellos de botella relacionales (Paso 1)             | 20     |
| Clasificación correcta de ≥6/8 fichas NoSQL con justificación técnica (Paso 2)  | 25     |
| Clasificación CAP correcta de ≥4/5 escenarios con razonamiento coherente (Paso 3)| 20    |
| Diferenciación correcta Hadoop vs NoSQL en ≥5/6 sub-tareas (Paso 4)            | 15     |
| Arquitectura Polyglot coherente con ventajas/desventajas identificadas (Paso 5) | 20     |
| **Total**                                                                       | **100**|

---

## Resolución de Problemas

### Problema 1: Dificultad para distinguir entre Clave-Valor y Documental

**Síntoma:** El estudiante clasifica todos los escenarios con acceso por ID como "Clave-Valor", sin distinguir cuándo se necesita un modelo Documental.

**Causa raíz:** Confusión entre el mecanismo de acceso (por clave) y el modelo de datos. Tanto las bases de datos Clave-Valor como las Documentales permiten acceso por ID, pero difieren en la capacidad de consulta sobre el contenido del valor.

**Solución:**
Aplicar el siguiente criterio de decisión:

```
¿Necesito consultar o filtrar por campos DENTRO del valor?
    │
    ├─ SÍ → Base de datos DOCUMENTAL
    │        (el valor es un documento con estructura consultable)
    │
    └─ NO → Base de datos CLAVE-VALOR
             (el valor es opaco; solo se accede por la clave exacta)

Ejemplo:
- "Dame la sesión del usuario ID-12345"          → Clave-Valor ✓
- "Dame todos los hoteles en Barcelona con >4★"  → Documental ✓
```

---

### Problema 2: Confusión al aplicar el Teorema CAP — "¿Por qué no elegir siempre CP?"

**Síntoma:** El estudiante clasifica todos los escenarios como CP, argumentando que "siempre es mejor tener datos consistentes".

**Causa raíz:** No se comprende el costo de la consistencia fuerte en un sistema distribuido: cuando ocurre una partición de red, un sistema CP debe **rechazar peticiones** (reducir disponibilidad) para no devolver datos potencialmente inconsistentes. En aplicaciones de alta demanda, esto puede significar que el sistema deja de responder durante la partición.

**Solución:**
Replantear la pregunta con el siguiente razonamiento:

```
En caso de partición de red, ¿qué prefiere el negocio?

CP → "Prefiero NO responder antes que responder con datos incorrectos"
     Aplicaciones: banca, reservas, inventario crítico

AP → "Prefiero responder aunque el dato sea ligeramente antiguo"
     Aplicaciones: feeds sociales, marcadores, catálogos de productos

Pregunta clave: ¿Cuál es el COSTO de la inconsistencia temporal?
- Costo alto (fraude, overselling, doble reserva) → CP
- Costo bajo (ver una foto con 2 segundos de retraso) → AP
```

---

## Limpieza

Este laboratorio no genera artefactos digitales persistentes. Sin embargo, se recomienda:

1. **Guardar los entregables** en un archivo de texto o documento (`.md`, `.txt`, `.docx`) para revisión posterior del instructor.
2. **Conservar el documento JSON** del Paso 1 (modelado de hotel ViajeYA), ya que servirá como referencia conceptual en los laboratorios de modelado de datos de los Labs 02 y 03.
3. **Anotar las clasificaciones CAP** de los sistemas estudiados, ya que estos conceptos son fundamentales para los Labs 07 y 08 (replicación y alta disponibilidad).

```
# Estructura de archivo sugerida para guardar los entregables:

lab-01-00-01-respuestas.md
├── ## Paso 1: Análisis ViajeYA
│   ├── Tabla de cuellos de botella
│   └── Documento JSON propuesto
├── ## Paso 2: Clasificación NoSQL
│   └── Tabla de 8 fichas
├── ## Paso 3: Teorema CAP
│   ├── 5 escenarios clasificados
│   └── Diagrama CAP
├── ## Paso 4: Hadoop vs NoSQL
│   └── Tabla InsightApp + Diagrama Lambda
└── ## Paso 5: Polyglot Programming
    ├── Tabla TravelCloud
    └── Ventajas/desventajas
```

---

## Resumen

En este laboratorio aplicaste los conceptos fundamentales que justifican la existencia y el diseño de las bases de datos NoSQL:

### Puntos Clave

- **Las limitaciones del modelo relacional** son concretas y medibles: JOINs de millones de filas que tardan 7 segundos, migraciones que bloquean tablas durante horas, campos `TEXT` que almacenan JSON no consultable. Estos no son problemas teóricos, sino cuellos de botella reales que afectan directamente la experiencia del usuario.

- **Los cuatro tipos de NoSQL** tienen perfiles de uso bien diferenciados: Clave-Valor para acceso ultra-rápido por ID (sesiones, caché), Documental para datos flexibles con consultas ricas (catálogos, perfiles), Columnar para escrituras masivas y series temporales (IoT, logs), y Grafos para traversal de relaciones complejas (redes sociales, fraude).

- **El Teorema CAP** no es una limitación a evitar, sino una herramienta de diseño: elegir CP o AP es una decisión de negocio que depende del costo de la inconsistencia temporal. Los sistemas modernos como Couchbase permiten configurar este trade-off por operación, ofreciendo flexibilidad sin sacrificar coherencia en operaciones críticas.

- **Hadoop y NoSQL son complementarios**, no competidores. Hadoop es óptimo para procesamiento batch de grandes volúmenes históricos; NoSQL es óptimo para acceso en tiempo real con baja latencia. La arquitectura Lambda combina ambos para cubrir todos los casos de uso.

- **Polyglot Persistence** es el patrón natural en arquitecturas de microservicios maduras: cada servicio usa el tipo de almacenamiento más adecuado para sus datos. Couchbase reduce la complejidad operativa al unificar múltiples modelos (KV, Documental, FTS, Analytics) en una sola plataforma.

### Conexión con los Siguientes Laboratorios

| Concepto de este lab              | Se profundiza en...                                          |
|-----------------------------------|--------------------------------------------------------------|
| Modelado de documentos JSON       | Lab 02: Diseño de modelos de datos JSON en Couchbase         |
| Full-Text Search                  | Lab 04 y 05: FTS con mappings avanzados y SQL++              |
| Consultas sobre documentos        | Lab 03: SQL++ para documentos JSON                           |
| Configuración CAP en Couchbase    | Lab 07 y 08: Replicación y consistencia                      |

### Recursos Adicionales

| Recurso                                                                                       | Tipo       |
|-----------------------------------------------------------------------------------------------|------------|
| [Couchbase: Why NoSQL?](https://www.couchbase.com/resources/why-nosql/)                       | Artículo   |
| [NoSQL Distilled — Fowler & Sadalage](https://martinfowler.com/books/nosql.html)              | Libro      |
| [Brewer's CAP Theorem — Eric Brewer (2000)](https://people.eecs.berkeley.edu/~brewer/cs262b-2004/PODC-keynote.pdf) | Paper |
| [Amazon Dynamo Paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) | Paper      |
| [Google Bigtable Paper](https://research.google/pubs/pub27898/)                              | Paper      |
| [Polyglot Persistence — Martin Fowler](https://martinfowler.com/bliki/PolyglotPersistence.html) | Artículo |

---
