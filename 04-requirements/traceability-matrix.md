# Matriz de Trazabilidad: Requerimientos vs. Modelo Físico

> **Documento:** `04-requirements/traceability-matrix.md`  
> **Sistema:** Simple Stock Flow  
> **Fuente de Verdad:** Mapeo formal entre Requerimientos de Negocio y las 22 columnas / 12 índices de [`spec/data-model.md`](../spec/data-model.md).

---

## 1. Matriz de Requerimientos vs. Modelo de Datos y Patrones de Acceso

Esta matriz demuestra que **cada requerimiento funcional e historia de usuario tiene un sustento directo en las tablas, columnas y consultas** documentadas en el modelo físico de PostgreSQL:

| Historia de Usuario (HU) | Requisito No Funcional (NFR) | Patrón de Acceso [§6.1] | Tablas y Columnas Físicas Involucradas [§3] | Restricciones e Índices en Motor [§4, §10] | Cita en el Modelo |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **HU-IAM-01** (Login operador) | NFR-03 (Seguridad credenciales) | **Q10** (Usuario por nombre exacto) | `sales.user` (`id`, `username`, `password_hash`, `role`) | `IX_user_username` (Único), `PK_user` | §2.5, §6.1 Q10, §7 |
| **HU-IAM-02** (Aprovisionar operadores) | NFR-03, NFR-04 (Minimización) | — | `sales.user` (`id`, `username`, `password_hash`, `role`) | `IX_user_username`, `PK_user` | §2.5, §9.2, DP-04 |
| **HU-CAT-01** (Alta de producto) | NFR-05 (Precisión monetaria) | **Q4** (Listar categorías), **Q5** (Cat. por ID) | `sales.product` (`id`, `name`, `price`, `stock`, `category_id`, `image_key`), `sales.category` (`id`, `name`) | `PK_product`, `FK_product_category_category_id` (`RESTRICT`), `ck_product_stock_non_negative` | §2.1, §2.2, §3, FK-1 |
| **HU-CAT-02** (Modificar precio) | NFR-05, NFR-07 (Inmutabilidad) | — | `sales.product` (`price`, `xmin`) | `PK_product`, propiedad sombra `xmin` | §2.2, D-04, §3 |
| **HU-CAT-03** (Restock inventario) | NFR-01 (Concurrencia stock) | **Q2** (Producto por ID) | `sales.product` (`stock`, `xmin`) | `PK_product`, `ck_product_stock_non_negative`, propiedad sombra `xmin` | §2.2, Q2, D-04 |
| **HU-CAT-04** (Baja lógica producto) | NFR-07 (Integridad histórica) | — | `sales.product` (`deleted_at`) | Filtro global EF Core `deleted_at IS NULL`, `FK_sale_item_product_product_id` (`RESTRICT`) | §2.2, ADR-003, T-09, FK-3 |
| **HU-CAT-05** (Búsqueda mostrador) | NFR-02 (Latencia P95 < 200ms) | **Q1** (Buscar por texto, cat, activos) | `sales.product` (`id`, `name`, `price`, `stock`, `category_id`, `deleted_at`) | `IX_product_category_id`, `IX_product_name`, índice compuesto parcial sobre activos (T-13) | §6.1 Q1, §6.2 |
| **HU-SALE-01** (Registro venta mostrador) | NFR-01 (Stock no negativo), NFR-05, NFR-06 (UTC), NFR-07 | **Q3** (Lote productos activos), **Q6** (Guardar venta) | `sales.sale` (`id`, `sold_at`, `sold_by`), `sales.sale_item` (`id`, `sale_id`, `product_id`, `product_name`, `category_name`, `quantity`, `unit_price`), `sales.product` (`stock`, `xmin`) | `PK_sale`, `PK_sale_item`, `FK_sale_item_sale_sale_id` (`CASCADE`), `IX_sale_item_sale_id_product_id` (`UNIQUE`), `ck_product_stock_non_negative` | §2.3, §2.4, §3, §4, §5 |
| **HU-SALE-02** (Detalle venta histórica) | NFR-07 (Snapshot congelado) | **Q6** (Venta con sus líneas) | `sales.sale` (`id`, `sold_at`, `sold_by`), `sales.sale_item` (todas las columnas congeladas) | `PK_sale`, `FK_sale_item_sale_sale_id` | §1, §2.4, §6.1 Q6 |
| **HU-SALE-03** (Listar ventas por rango)| NFR-02, NFR-06 (UTC) | **Q7** (Ventas por fecha desc) | `sales.sale` (`id`, `sold_at`, `sold_by`) | `IX_sale_sold_at` (Ascendente/recorrido B-tree reversible) | §6.1 Q7, §6.2 |
| **HU-REP-01** (Reporte agregado ventas) | NFR-02 (Index-only scan), NFR-04, NFR-07 | **Q9** (Reporte agregado por producto) | `sales.sale` (`sold_at`), `sales.sale_item` (`product_id`, `product_name`, `category_name`, `quantity`, `unit_price`) | `IX_sale_sold_at`, `IX_sale_item_product_id`, índice único compuesto con `INCLUDE (quantity, unit_price)` | §1 D-06, §6.1 Q9, §11.1 H-1 |

---

## 2. Cobertura de las 22 Columnas Físicas del Esquema `sales`

Comprobación cruzada que asegura que ninguna columna del modelo físico carece de caso de uso o razón de ser:

| Tabla | Columna | Tipo | Requerimiento / HU que la utiliza | Cita en el Modelo |
| :--- | :--- | :--- | :--- | :--- |
| `category` | `id` | `uuid` | HU-CAT-01 (Clave foránea FK-1 de productos) | §2.1, §3, §9.1 |
| `category` | `name` | `varchar(120)` | HU-CAT-01, HU-CAT-05 (Clasificación en catálogo) | §2.1, §3, §10.1 |
| `product` | `id` | `uuid` | HU-CAT-01..05, HU-SALE-01..02 (Identificador primario) | §2.2, §3, §10.1 |
| `product` | `name` | `varchar(200)` | HU-CAT-01, HU-CAT-05 (Búsqueda en catálogo) | §2.2, §3, §10.1 |
| `product` | `price` | `numeric(18,2)` | HU-CAT-01, HU-CAT-02 (Precio vigente en mostrador) | §2.2, §3, §10.1 |
| `product` | `stock` | `integer` | HU-CAT-01, HU-CAT-03, HU-SALE-01 (Inventario físico disponible) | §2.2, §3, ADR-002 |
| `product` | `category_id` | `uuid` | HU-CAT-01, HU-CAT-05 (Relación con categoría) | §2.2, §3, FK-1 |
| `product` | `image_key` | `varchar(512)` | HU-CAT-01 (Clave opaca de almacenamiento externo) | §1, §2.2, D-08, §3 |
| `product` | `deleted_at` | `timestamptz` | HU-CAT-04 (Baja lógica, propiedad sombra) | §2.2, ADR-003, T-09 |
| `product` | `xmin` | `xid` | HU-CAT-02, HU-CAT-03, HU-SALE-01 (Concurrencia optimista) | §2.2, D-04, T-10 |
| `sale` | `id` | `uuid` | HU-SALE-01, HU-SALE-02, HU-SALE-03 (Identificador de venta) | §2.3, §3, §10.1 |
| `sale` | `sold_at` | `timestamptz` | HU-SALE-01, HU-SALE-03, HU-REP-01 (Fecha y hora de venta UTC) | §2.3, §3, §8 |
| `sale` | `sold_by` | `varchar(120)` | HU-SALE-01, HU-SALE-02 (Nombre del operador que registró) | §2.3, §3, §7 |
| `sale` | `sold_by_user_id`| `uuid` | HU-SALE-01, HU-SALE-02 (Autoría por ID de operador - T-12) | §3, FK-4, §13 |
| `sale_item` | `id` | `uuid` | HU-SALE-01, HU-SALE-02 (Identificador de línea) | §2.4, §3, §10.1 |
| `sale_item` | `sale_id` | `uuid` | HU-SALE-01, HU-SALE-02 (Vínculo en cascada con la venta) | §2.4, §3, FK-2 |
| `sale_item` | `product_id` | `uuid` | HU-SALE-01, HU-REP-01 (Vínculo restrictivo con producto) | §2.4, §3, FK-3 |
| `sale_item` | `product_name` | `varchar(200)` | HU-SALE-01, HU-SALE-02, HU-REP-01 (Nombre congelado en la venta) | §1, §2.4, D-06 |
| `sale_item` | `category_name`| `varchar(120)` | HU-SALE-01, HU-SALE-02, HU-REP-01 (Categoría congelada en la venta)| §1, §2.4, D-06, T-11 |
| `sale_item` | `quantity` | `integer` | HU-SALE-01, HU-SALE-02, HU-REP-01 (Unidades vendidas) | §1, §2.4, §3 |
| `sale_item` | `unit_price` | `numeric(18,2)` | HU-SALE-01, HU-SALE-02, HU-REP-01 (Precio congelado en la venta) | §1, §2.4, D-06 |
| `user` | `id` | `uuid` | HU-IAM-01, HU-IAM-02 (Identificador de cuenta) | §2.5, §3, §10.1 |
| `user` | `username` | `varchar(120)` | HU-IAM-01, HU-IAM-02 (Nombre de usuario único) | §2.5, §3, §4 |
| `user` | `password_hash`| `varchar(512)` | HU-IAM-01, HU-IAM-02 (Hash seguro de autenticación) | §2.5, §3, §7 |
| `user` | `role` | `varchar(40)` | HU-IAM-01, HU-IAM-02 (Atribución 'admin' o 'seller') | §2.5, §3, §4 |
