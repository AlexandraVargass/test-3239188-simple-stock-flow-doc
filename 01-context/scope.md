# Alcance del Sistema (System Scope)

> **Documento:** `01-context/scope.md`  
> **Sistema:** Simple Stock Flow  
> **Fuente de Verdad Técnica:** [`spec/data-model.md`](../spec/data-model.md)

---

## 1. Dentro del Alcance (In Scope)

Lo que el sistema **SÍ construye, soporta y mantiene**, respaldado directamente por las tablas y casos de uso del modelo:

### Funcionalidades del Núcleo Operativo:

| # | Característica | Descripción Funcional | Respaldo en el Modelo |
| :---: | :--- | :--- | :--- |
| **1** | **Autenticación y Roles de Operador** | Inicio de sesión para roles `admin` y `seller`, normalización de usuario a minúsculas y validación mediante `password_hash` criptográfico. | §1, §2.5, §7, Q10 |
| **2** | **Aprovisionamiento Controlado** | Creación de cuentas de operadores por el administrador; el rol `admin` no se crea en ejecución, proviene del entorno (DP-04). | §9.2, §11.1 (H-3) |
| **3** | **Catálogo Esencial de Productos** | Registro, edición de precio, restock de inventario y asociación a una categoría obligatoria (FK-1 RESTRICT). Clave opaca `image_key` opcional (D-08). | §1, §2.2, §3, FK-1 |
| **4** | **Baja Lógica de Productos** | Retiro comercial de artículos marcando `deleted_at = now()`, ocultándolos de búsquedas pero preservando sus filas físicas para ventas pasadas. | §2.2, ADR-003, T-09 |
| **5** | **Búsqueda Rápida en Mostrador** | Búsqueda por coincidencia parcial de texto y categoría sobre productos activos (`deleted_at IS NULL`) con paginación (Q1). | §6.1 (Q1), §6.2 |
| **6** | **Registro Atómico de Venta en Caja** | Confirmación de ventas con al menos una línea, no repetición de productos, descuento atómico de stock y creación de líneas en una sola transacción ACID. | §2.3, §2.4, ADR-002 |
| **7** | **Inmutabilidad y Datos Congelados** | Venta inmutable tras confirmación. Congelamiento de `product_name`, `category_name` y `unit_price` en `sale_item` (*Frozen Snapshot*). | §1, §2.3, §2.4, ADR-004 |
| **8** | **Total Dinámico de Venta** | Cálculo dinámico en memoria de subtotales y total sumando importes de líneas, sin columnas físicas redundantes (Artículo VII). | §1, §2.3 |
| **9** | **Reportería Comercial Agregada** | Consulta SQL en motor agrupando por valores congelados en un rango de fechas en UTC, sin desglosar por vendedor (Q9). | §1 (D-06), §6.1 (Q9), §11.1 (H-1) |
| **10**| **Concurrencia Optimista en Stock** | Prevención de sobreventas en caja usando la columna de sistema `xmin` de PostgreSQL y la restricción física `ck_product_stock_non_negative`. | §2.2 (D-04, T-10), ADR-002, §4 |

---

## 2. Fuera del Alcance (Out of Scope Explícito)

Lo que el sistema **NO construye**, con la justificación técnica de por qué fue formalmente excluido:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                          TABLA FORMAL DE EXCLUSIÓN DE ALCANCE                          │
├──────────────────────────┬─────────────────────────────────────────────────────────────┤
│ Lo que NO se construye   │ Justificación y Decisión Cerrada en el Modelo               │
├──────────────────────────┼─────────────────────────────────────────────────────────────┤
│ 1. Cuentas de Compradores│ No existe entidad cliente ni comprador (§1). La venta       │
│    o Clientes Finales    │ registra únicamente al operador interno (§7).               │
├──────────────────────────┼─────────────────────────────────────────────────────────────┤
│ 2. Pagos en Línea y      │ No hay tarjetas de crédito, pasarelas de pago ni tokens     │
│    Pasarelas Bancarias   │ financieros (§7). El cobro se realiza presencialmente.     │
├──────────────────────────┼─────────────────────────────────────────────────────────────┤
│ 3. Multimoneda o         │ El sistema es monomoneda estricto por diseño (D-05, §1, §3).│
│    Conversión de Divisas │ Prohibición total de columnas de moneda o tipos de cambio.  │
├──────────────────────────┼─────────────────────────────────────────────────────────────┤
│ 4. CRUD de Categorías    │ Las categorías son fijas (5) sembradas en la migración      │
│                          │ inicial (D-10, §2.1, §9.1). Repositorio de solo lectura.    │
├──────────────────────────┼─────────────────────────────────────────────────────────────┤
│ 5. Columnas de Auditoría │ Decisión formalmente cerrada en §8: el sistema NO lleva     │
│    created_at / updated_at│ columnas forenses ni disparadores ocultos en base de datos.│
├──────────────────────────┼─────────────────────────────────────────────────────────────┤
│ 6. Edición o Anulación   │ Las ventas son hechos contables consumados e inmutables.    │
│    de Ventas Registradas │ No existen operaciones de modificación o borrado (§1, §7.1).│
├──────────────────────────┼─────────────────────────────────────────────────────────────┤
│ 7. Desglose de Reportes  │ Prohibido por política de privacidad laboral (DP-02, §7):    │
│    por Vendedor          │ el reporte comercial agrega exclusivamente por producto.    │
├──────────────────────────┼─────────────────────────────────────────────────────────────┤
│ 8. Binarios de Imagen en │ La base solo guarda una clave opaca de 512 caracteres       │
│    Base de Datos (Blobs) │ (D-08, §1, §7.1). Los binarios viven en storage externo.    │
├──────────────────────────┼─────────────────────────────────────────────────────────────┤
│ 9. Campos Complejos de   │ Principio de Catálogo Esencial (DP-03, §1, §12): no hay     │
│    Producto (SKU, etc.)  │ descripciones, marcas, variantes de talla/color ni tags.    │
└──────────────────────────┴─────────────────────────────────────────────────────────────┘
```

---

## 3. Supuestos del Alcance

Los siguientes supuestos se consideran verdaderos para el diseño e implantación del sistema:

1. **Entorno de Red Local o WAN Confiable `[Supuesto]`:** Los terminales de punto de venta (navegadores en mostrador) tienen conectividad TCP directa hacia el servidor de la API (`simple-stock-flow-api`).
2. **Volumen de Punto de Venta Individual (§6.1, §10.4):** El sistema atiende el flujo de un comercio o sucursal con un catálogo de miles de artículos y hasta decenas de terminales concurrentes (no requiere sharding distribuido ni clusters multiregión).
3. **Gestión de Stock Físico Centralizada:** Todo el stock del producto reside en la misma tienda o mostrador (no se gestionan múltiples almacenes o bodegas intermedias).
4. **Moneda Local Homogénea:** El negocio factura exclusivamente en la moneda local de curso legal sin requerir discriminación de divisas (§1, D-05).

---

## 4. Restricciones Tecnológicas Innegociables

| Restricción | Descripción |
| :--- | :--- |
| **Motor de Base de Datos** | Debe ser **PostgreSQL versión 16+** para soportar el esquema `sales`, tipos `timestamptz` y la columna interna `xmin` [§3, §10]. |
| **Zona Horaria del Servidor** | El servidor de base de datos y la aplicación **deben correr en UTC estricto**. No se admiten marcas temporales locales en base de datos [§3]. |
| **Redondeo Numérico** | La aritmética decimal de moneda debe utilizar el redondeo bancario **`MidpointRounding.AwayFromZero`** a 2 decimales en el tipo `numeric(18,2)` [§2.2, §3]. |
| **Control de Esquema** | El DDL de la base de datos es gobernado **exclusivamente por las migraciones de Entity Framework Core** [ADR-001, §3.2]. No se admiten modificaciones manuales de esquema por fuera de las migraciones. |
