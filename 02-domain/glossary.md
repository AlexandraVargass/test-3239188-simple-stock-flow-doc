# Glosario del Dominio (Lenguaje Ubicuo)

> **Documento:** `02-domain/glossary.md`  
> **Sistema:** Simple Stock Flow  
> **Fuente de Verdad Técnica:** [`spec/data-model.md`](../spec/data-model.md) §1

---

## 1. El Lenguaje Ubicuo de Simple Stock Flow

El lenguaje ubicuo establece los términos exactos y sin ambigüedades utilizados en las conversaciones con los interesados del negocio, en la documentación y en el código fuente de C#. El código y los nombres de columnas van en inglés (artículo XI); este glosario explica su definición funcional en español y dónde reside técnicamente [§1]:

| Término (Negocio) | Definición Funcional | Dónde Vive en el Sistema (Técnico) | Cita en el Modelo |
| :--- | :--- | :--- | :--- |
| **Producto (`Product`)** | Artículo vendible del catálogo comercial. Tiene **nombre, precio, stock, categoría e imagen opcional, y nada más** (DP-03). Sin SKU, sin descripción larga. | `Product` (C#) · tabla `sales.product` (PostgreSQL) | §1, §2.2, §3 |
| **Categoría (`Category`)** | Clasificación a la que pertenece un producto. Conjunto **fijo de cinco categorías sembradas**, sin mantenimiento ni CRUD (D-10). | `Category` (C#) · tabla `sales.category` | §1, §2.1, §9.1 |
| **Precio (`Price`)** | Valor monetario vigente del producto en el catálogo. Debe ser estrictamente positivo (`> 0`). | Value Object `Money` · columna `product.price` | §1, §2.2 |
| **Stock** | Unidades físicas disponibles del producto para despacho. Nunca puede ser negativo (`>= 0`). | Columna `product.stock` · restricción `ck_product_stock_non_negative` | §1, §2.2, ADR-002 |
| **Imagen del producto** | **Clave opaca externa** del archivo en el almacenamiento. Ni el binario ni la ruta residen en BD (D-08). Ausente se representa como `NULL`. | Columna `product.image_key` (`varchar(512)`) | §1, §2.2, §7.1 |
| **Venta (`Sale`)** | Hecho comercial consumado e **inmutable**: quién, cuándo y qué se despachó. Una vez registrada no se edita ni se borra. | `Sale` (C#) · tabla `sales.sale` | §1, §2.3, §7.1 |
| **Línea de venta (`SaleItem`)** | Renglón individual de la venta: producto, cantidad y **precio/categoría congelados** del momento. No existe fuera de su venta. | `SaleItem` (C#) · tabla `sales.sale_item` | §1, §2.4, ADR-004 |
| **Cantidad (`Quantity`)** | Unidades vendidas en una línea de venta. Estrictamente positiva (`> 0`). | Value Object `Quantity` · columna `sale_item.quantity` | §1, §2.4 |
| **Total de la venta** | Suma aritmética de subtotales. **Se calcula al vuelo, no se almacena en base de datos** (artículo VII). | Propiedad calculada `Sale.Total` · **sin columna física** | §1, §2.3 |
| **Subtotal de la línea** | Precio unitario por cantidad vendida. **Se calcula al vuelo, no se almacena**. | Propiedad calculada `SaleItem.Subtotal` · **sin columna física** | §1, §2.4 |
| **Usuario (`User`)** | Operador interno que se autentica y registra ventas o administra. **No hay entidad cliente ni comprador final**. | `User` (C#) · tabla `sales.user` | §1, §2.5, §7 |
| **Rol (`Role`)** | Atribución operativa del usuario dentro de un conjunto cerrado de dos: `admin` o `seller`. | Columna `user.role` (`varchar(40)`) | §1, §2.5 |
| **Hash de clave (`password_hash`)**| Huella criptográfica irreversible de la contraseña. El dominio **nunca ve la clave en claro** (D-09). | Columna `user.password_hash` (`varchar(512)`) | §1, §2.5, §7 |
| **Rango de fechas (`DateRange`)** | Ventana temporal en UTC del reporte. El fin no puede ser anterior al inicio. | Objeto de valor de capa de aplicación · **sin tabla** | §1 |
| **Reporte de ventas** | Agregación por producto sobre un rango temporal. **No se persiste**: se calcula directamente en el motor (D-06). | Consulta SQL directa Q9 · **sin tabla de reporte** | §1, §6.1 Q9, §11.1 |
| **Nombre y Precio Congelados (*Frozen Snapshot*)** | Copia literal del valor en el instante exacto de la venta. **No sigue las mutaciones del catálogo**. | Columnas `sale_item.product_name`, `sale_item.category_name`, `sale_item.unit_price` | §1, §2.4, ADR-004 |

---

## 2. Decisiones de Lenguaje Cerradas en el Modelo [§1]

1. **Monomoneda:** El glosario original de borradores previos mencionaba *moneda de la venta*; el modelo oficial cierra esta ambigüedad: **el sistema es monomoneda por construcción (D-05)** y no hay concepto de moneda variable ni columna de divisa.
2. **Catálogo Esencial:** Queda prohibida la inclusión de campos adicionales en `Product` (DP-03).
3. **Comprador Inexistente:** Queda expresamente aclarado que el término "Cliente" se refiere a la persona física atendida en el mostrador presencial, pero que **no existe entidad de datos Comprador** en el software (§7).
