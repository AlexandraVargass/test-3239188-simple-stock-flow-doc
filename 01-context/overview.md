# Descripción General del Sistema (System Overview)

> **Documento:** `01-context/overview.md`  
> **Sistema:** Simple Stock Flow  
> **Fuente de Verdad Técnica:** [`spec/data-model.md`](../spec/data-model.md)

---

## 1. ¿Qué es Simple Stock Flow?

**Simple Stock Flow** es un sistema de información web para **punto de venta (POS) y control de flujo de inventario**, diseñado para la gestión operativa en mostrador de comercios de venta directa al público (como ferreterías, depósitos y almacenes técnicos, según lo demuestran las categorías semilla: *General, Herramientas, Electricidad, Fontanería y Pinturas* [§9.1]).

El sistema permite a los operadores registrar ventas de forma rápida y segura, garantizando el descuento atómico de existencias en tiempo real, congelando los precios históricos para proteger la contabilidad de períodos cerrados y ofreciendo reportería comercial agregada sin sobrecargar la infraestructura [§1, §2.3, §6.1 Q9].

---

## 2. El Problema que Resuelve

### Antes del Sistema:
En el mostrador tradicional, los negocios gestionan su catálogo y ventas mediante libretas, hojas de cálculo o sistemas ERP genéricos sobrecargados. Esto ocasiona:
1. **Sobreventas en Mostrador:** Dos vendedores en cajas distintas venden simultáneamente el mismo artículo escaso, provocando stock negativo e incumplimiento a los clientes.
2. **Corrupción de Balances Pasados:** Al actualizar el precio o reclasificar un producto en el catálogo hoy, los reportes de ventas de meses anteriores se alteran retrospectivamente, destruyendo la trazabilidad contable.
3. **Lentitud Operativa:** Sistemas ERP pesados que exigen configurar múltiples divisas, pasarelas de pago y datos extensos de clientes para una venta presencial rápida en caja.

### Con Simple Stock Flow:
1. **Consistencia Transaccional:** Cada venta descuenta el inventario de manera atómica bajo control de concurrencia optimista (`xmin` de PostgreSQL) y la salvaguarda física de `ck_product_stock_non_negative` (`stock >= 0`), haciendo imposible las sobreventas [ADR-002, §2.2, §2.3].
2. **Inmutabilidad Contable (*Frozen Snapshot*):** Cada línea de venta congela el nombre del producto, la categoría y el precio en el instante de la transacción. Los reportes de períodos contables cerrados jamás se alteran, incluso si el catálogo cambia después (D-06, ADR-004 [§1, §11.1]).
3. **Agilidad Monomoneda:** Sin conversiones de divisas ni fricciones operativas innecesarias (D-05 [§1, §3]).

---

## 3. Usuarios Principales del Sistema

El sistema restringe su acceso exclusivamente a **operadores internos de la empresa** autenticados bajo dos roles cerrados [§1, §2.5]:

| Rol | Descripción Funcional | Actividades en el Sistema | Cita en el Modelo |
| :--- | :--- | :--- | :--- |
| **Administrador (`admin`)** | Dueño, gerente o jefe de almacén del negocio | Administra el catálogo de productos (altas, bajas lógicas, cambio de precios, restock), gestiona cuentas de vendedores y consulta reportes comerciales consolidados (Q9). | §1, §2.2, §2.5, §6.1 Q9 |
| **Vendedor (`seller`)** | Operador de mostrador o cajero en el punto de venta | Inicia sesión en caja, busca productos activos en catálogo (Q1), consulta existencias en tiempo real y registra ventas atómicas presenciales. | §1, §2.3, §2.5, §6.1 Q1, Q6 |

> **Aviso de Dominio Innegociable:** **No existe el rol de cliente, comprador ni usuario público.** La venta se efectúa presencialmente en mostrador y el sistema registra al operador interno que cobra (`sold_by`/`sold_by_username`), preservando la privacidad del comprador [§1, §7].

---

## 4. Stack Tecnológico Confirmado

| Capa / Componente | Tecnología | Justificación y Evidencia en el Modelo |
| :--- | :--- | :--- |
| **Lenguaje de Dominio** | **C# / .NET 8+** | Clases de dominio en singular, Value Objects en registros (`record`), colecciones en plural (`DbSet<Product>`), redondeo `MidpointRounding.AwayFromZero` [§0, §2.2, §12]. |
| **Acceso a Datos** | **Entity Framework Core (EF Core)** | Migraciones automáticas de esquema, propiedades sombra (`deleted_at`, `xmin`), `HasForeignKey` y mapeo a esquema `sales` [§0, §3, §3.1, §3.2]. |
| **Base de Datos** | **PostgreSQL 16.14** | Servidor en UTC, esquema `sales`, restricciones físicas (`CHECK (stock >= 0)`), 12 índices y tipos `timestamptz`/`numeric(18,2)` [§3, §10]. |
| **Almacenamiento de Binarios** | **Almacenamiento Externo Desacoplado** `[Supuesto: S3 / Blob]` | La base de datos solo almacena claves opacas `image_key` (`varchar(512)`), garantizando que los binarios no sobrecarguen las copias de seguridad de PostgreSQL (D-08 [§1, §7.1]). |

---

## 5. Estado Actual del Sistema

* **Fase:** Modelo físico y de dominio cerrado, verificado contra el motor PostgreSQL 16.14 el 2026-09-19 [§preámbulo, §10].
* **Esquema:** Esquema `sales` con **5 tablas en singular**, 22 columnas físicas, 8 restricciones y 12 índices validados [§3, §10].
* **Volumen de Arranque (§10.4):**
  * `category`: **5 filas** (semilla fija de solo lectura, §9.1).
  * `user`: **1 fila** (administrador inicial aprovisionado desde variables de entorno, §9.2).
  * `product`, `sale`, `sale_item`: **0 filas** (listas para operar en producción).
