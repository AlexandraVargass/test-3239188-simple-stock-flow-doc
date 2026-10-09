# Requisitos No Funcionales (NFR) — Simple Stock Flow

> **Documento:** `04-requirements/non-functional.md`  
> **Sistema:** Simple Stock Flow  
> **Fuente de Verdad:** Deducido cuantitativamente de las restricciones físicas, índices y políticas de [`spec/data-model.md`](../spec/data-model.md).

---

## 1. Principio de Medibilidad de Requisitos No Funcionales

Siguiendo el estándar riguroso de ingeniería de software, un requisito no funcional solo es válido si posee una **métrica cuantitativa verificable** y una **condición de prueba reproducible**. No se aceptan declaraciones subjetivas como "el sistema debe ser rápido".

---

## 2. Catálogo de Requisitos No Funcionales

### NFR-01: Integridad Transaccional y Concurrencia de Inventario

| Dimensión | Especificación Cuantitativa | Cita en el Modelo |
| :--- | :--- | :--- |
| **Atributo** | Concurrencia Optimista e Integridad Absoluta de Stock | §2.2 (D-04, T-10), ADR-002, §4 |
| **Métrica** | **0 ocurrencias de stock negativo (`stock < 0`)** y **0 inconsistencias por sobreescritura perdida (*lost updates*)** bajo 50 operaciones concurrentes simultáneas de venta sobre el mismo producto. | ADR-002, §2.2 |
| **Mecanismo Técnico** | Uso de la columna interna de sistema `xmin` de PostgreSQL como propiedad sombra de concurrencia en Entity Framework Core. Respaldo definitivo en el motor mediante la restricción `CHECK ((stock >= 0))` (`ck_product_stock_non_negative`). | §2.2, §3, §10.2 |
| **Condición de Prueba** | Ejecutar una prueba de carga simulando dos cajas intentando vender la última unidad disponible (`stock = 1`) en el mismo milisegundo. La primera debe confirmar y la segunda debe arrojar `DbUpdateConcurrencyException`, manteniendo el stock final en 0. | Q3, §6.1 |

---

### NFR-02: Rendimiento y Latencia de Consultas Críticas

| Dimensión | Especificación Cuantitativa | Cita en el Modelo |
| :--- | :--- | :--- |
| **Atributo** | Tiempo de Respuesta y Throughput en Punto de Venta | §6.1 (Q1, Q3, Q6, Q9, Q10), §6.2 |
| **Métricas Objetivo** | 1. **P95 < 200 ms** en búsqueda de productos en mostrador (Q1) sobre catálogo de 10,000 ítems.<br/>2. **P95 < 150 ms** en registro y confirmación atómica de venta de hasta 10 ítems (Q3 + Q6).<br/>3. **P95 < 500 ms** en generación del reporte comercial agregado (Q9) sobre un histórico de 100,000 líneas de venta. | §6.1, §6.2 |
| **Mecanismo Técnico** | 1. Para Q1: Índice compuesto `product(category_id, name)` parcial sobre activos (`WHERE deleted_at IS NULL`) [T-13, §6.2].<br/>2. Para Q9: Índice compuesto `sale_item(sale_id, product_id)` con columnas incluidas `INCLUDE (quantity, unit_price)` para lograr **recorrido exclusivo por índice (*Index-Only Scan*)**, calculando la agregación sin tocar las páginas de tabla en disco [T-13, §6.2].<br/>3. Para Q10: Índice único `IX_user_username` para autenticación instantánea. | §6.2, §10.3 |
| **Condición de Prueba** | Carga sintética con k6 ejecutando 100 peticiones por segundo en el entorno de pruebas local. | [Supuesto] |

---

### NFR-03: Seguridad y Confidencialidad de Credenciales

| Dimensión | Especificación Cuantitativa | Cita en el Modelo |
| :--- | :--- | :--- |
| **Atributo** | Protección Criptográfica y Aislamiento de Secretos | §1 (D-09), §2.5, §7, §9.2 |
| **Métricas Objetivo** | 1. **0 contraseñas en texto plano** visibles en memoria de dominio, base de datos, trazas de depuración o logs del servidor.<br/>2. **0 ocurrencias del hash de contraseña expuesto** en respuestas JSON de la API.<br/>3. **0 índices creados sobre la columna `password_hash`** (clasificación confidencial). | §2.5, §7, §10.3 |
| **Mecanismo Técnico** | La transformación de la contraseña la ejecuta exclusivamente el puerto secundario `IPasswordHasher` (D-09). La base de datos almacena únicamente `password_hash` (`varchar(512)`). El aprovisionamiento del administrador inicial se realiza en el arranque desde variables de entorno, nunca sembrando hashes literales en SQL [§9.2]. | §2.5, §7, §9.2 |
| **Condición de Prueba** | Inspección automatizada de logs y respuestas de los endpoints `/api/users` y `/api/sales` verificando ausencia total del campo `password_hash`. | §7 |

---

### NFR-04: Privacidad y Minimización de Datos Personales

| Dimensión | Especificación Cuantitativa | Cita en el Modelo |
| :--- | :--- | :--- |
| **Atributo** | Cumplimiento de Principio de Minimización y Ámbito de Privacidad | §1, §7, §7.1, DP-02 |
| **Métricas Objetivo** | **0 datos personales de clientes finales almacenados en el sistema.** | §1, §7 |
| **Justificación del Modelo** | El sistema no posee entidad comprador ni cliente final (§1). Los únicos datos personales son `user.username` y `sale.sold_by_username` (operadores internos de la empresa). El reporte comercial Q9 agrega exclusivamente por producto, prohibiendo explícitamente el desglose por operador o vendedor (DP-02, §7, §7.1). | §1, §7, DP-02 |
| **Política de Retención** | Ventas y autoría de operadores poseen retención indefinida por obligación contable (§7.1). No existe borrado de ventas. | §7.1 |

---

### NFR-05: Precisión Aritmética y Consistencia Monetaria

| Dimensión | Especificación Cuantitativa | Cita en el Modelo |
| :--- | :--- | :--- |
| **Atributo** | Exactitud en Cálculos Financieros sin Pérdida de Fracciones | §1 (D-05), §2.2, §3 |
| **Métricas Objetivo** | **0 discrepancias por redondeo en totales de venta y reportes.** Exactitud garantizada a 2 decimales. | §2.2 |
| **Mecanismo Técnico** | 1. Tipo de dato `numeric(18,2)` en columnas `product.price` y `sale_item.unit_price` [§3].<br/>2. Redondeo estricto en el Value Object `Money` mediante `MidpointRounding.AwayFromZero` antes de cualquier asignación [§2.2].<br/>3. Sistema monomoneda estricto por diseño (D-05): prohibición total de mezclar divisas o conversiones en caliente [§1, §3]. | §1, §2.2, §3 |
| **Condición de Prueba** | Suite de pruebas unitarias sobre `Sale.Total` sumando 10,000 líneas con fracciones centesimales, validando paridad exacta con la suma calculada por PostgreSQL. | [Supuesto] |

---

### NFR-06: Manejo Temporal y Estandarización de Zona Horaria

| Dimensión | Especificación Cuantitativa | Cita en el Modelo |
| :--- | :--- | :--- |
| **Atributo** | Sincronización Temporal Global | §3, §8 |
| **Métricas Objetivo** | **100% de los sellos temporales gestionados y persistidos en UTC.** | §3 |
| **Mecanismo Técnico** | Todas las columnas temporales del esquema `sales` son obligatoriamente de tipo `timestamptz` (`timestamp with time zone`) [§3]. El servidor corre en UTC. Esto aplica al único instante de negocio (`sale.sold_at`) y a la marca de baja lógica (`product.deleted_at`) [§3, §8]. | §3, §8, §10.1 |
| **Condición de Prueba** | Inserción de venta desde un cliente configurado en zona horaria GMT-5 (Colombia); verificación en `information_schema` y `pg_catalog` de que el registro se almacena en UTC exacto. | §10.1 |

---

### NFR-07: Inmutabilidad e Integridad Referencial Histórica

| Dimensión | Especificación Cuantitativa | Cita en el Modelo |
| :--- | :--- | :--- |
| **Atributo** | Preservación Inmutable de Períodos Contables Cerrados | §1, §2.4, §5, §7.1, §11.1 |
| **Métricas Objetivo** | **0 modificaciones a reportes de ventas de períodos cerrados tras cambios en el catálogo.** | §11.1 |
| **Mecanismo Técnico** | 1. Patrón *Frozen Snapshot*: `sale_item` almacena copias inmutables de `product_name`, `category_name` y `unit_price` en el momento de la venta (D-06, ADR-004) [§1, §2.4].<br/>2. `category_name` en `sale_item` carece deliberadamente de clave foránea hacia la tabla `category` para evitar que renombrar una categoría reescriba el pasado comercial [§2.4, §3].<br/>3. Restricción `FK_sale_item_product_product_id` con `ON DELETE RESTRICT` (FK-3, T-20) [§5, §10.2]. | §2.4, §3, §5, §11.1 |
| **Condición de Prueba** | Registrar una venta de un producto; modificar posteriormente su nombre, precio y categoría en el catálogo; verificar que la venta histórica y el reporte del período previo conservan idénticos valores a los originales. | §11.1 |
