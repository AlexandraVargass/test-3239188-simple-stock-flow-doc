# Entidades, Objetos de Valor e Invariantes del Dominio

> **Documento:** `02-domain/entities-and-rules.md`  
> **Sistema:** Simple Stock Flow  
> **Fuente de Verdad Técnica:** [`spec/data-model.md`](../spec/data-model.md)

---

## 1. Tactical DDD: Agregados y Entidades

El dominio de Simple Stock Flow se descompone en **cinco entidades, agrupadas en tres agregados y una entidad de referencia de solo lectura** [§2]:

```
┌────────────────────────────────────────────────────────┐
│                   AGREGADOS DEL DOMINIO                │
├───────────────────────┬────────────────────────────────┤
│ Agregado Product      │ Raíz: Product                  │
│                       │ Control de stock e inventario  │
├───────────────────────┼────────────────────────────────┤
│ Agregado Sale         │ Raíz: Sale                     │
│                       │ Entidad Interna: SaleItem      │
│                       │ Control de líneas y totales    │
├───────────────────────┼────────────────────────────────┤
│ Agregado User         │ Raíz: User                     │
│                       │ Identidad y roles de operador  │
├───────────────────────┼────────────────────────────────┤
│ Entidad de Referencia │ Category (Solo lectura,        │
│                       │ sembrada, sin ciclo de vida)   │
└───────────────────────┴────────────────────────────────┘
```

---

## 2. Definición Detallada de Entidades

### 2.1 Entidad de Referencia: `Category` [§2.1]
* **Naturaleza:** Entidad de catálogo fijo, no es raíz de agregado ni posee ciclo de vida modificable.
* **Semilla Inicial (D-10, §9.1):** Nace en la migración inicial con **5 identificadores fijos UUID v4**:
  * `11111111-1111-4111-8111-111111111111` — General
  * `22222222-2222-4222-8222-222222222222` — Herramientas
  * `33333333-3333-4333-8333-333333333333` — Electricidad
  * `44444444-4444-4444-8444-444444444444` — Fontanería
  * `55555555-5555-4555-8555-555555555555` — Pinturas
* **Invariantes [§2.1, §4]:**
  * `name` obligatorio, no vacío y recortado de espacios [garantizado en solo dominio por `Category.Rename`, baja al motor en T-20].
  * `name` único [garantizado en motor por el índice único `IX_category_name`].

---

### 2.2 Raíz de Agregado: `Product` [§2.2]
* **Naturaleza:** Representa un artículo comercializable en el inventario.
* **Atributos:**
  * `id` (`UUID`): Identidad inmutable de la raíz.
  * `name` (`string`): Nombre comercial (máx. 200 caracteres, recortado sin espacios).
  * `price` (`Money`): Precio de venta unitario vigente en catálogo.
  * `stock` (`integer`): Existencias físicas disponibles en almacén/mostrador.
  * `category_id` (`UUID`): Identificador de la categoría a la que pertenece (FK-1 RESTRICT [§5]).
  * `image_key` (`string?`): Clave opaca externa (máx. 512 caracteres) o `NULL`.
  * `deleted_at` (`DateTime?`): Marca temporal de baja lógica (propiedad sombra).
  * `xmin` (`uint`): Token de concurrencia optimista del motor (propiedad sombra).
* **Invariantes Tácticas [§2.2, §4]:**
  * **INV-PROD-01 (Stock No Negativo):** Tras cualquier operación de salida o venta, `stock >= 0`. Si salta la guarda de C#, el motor PostgreSQL la defiende con `ck_product_stock_non_negative` [ADR-002, §4].
  * **INV-PROD-02 (Retiro Atómico):** No es posible retirar más unidades de las actualmente disponibles (`Product.Withdraw(qty)` valida `qty <= stock`).
  * **INV-PROD-03 (Precio Positivo):** El precio de catálogo debe ser estrictamente mayor a 0 (`Product.ChangePrice` valida `price > 0`).
  * **INV-PROD-04 (Categoría Obligatoria):** Todo producto debe pertenecer a una categoría existente (`FK-1 RESTRICT` en motor).
  * **INV-PROD-05 (Representación Nula de Imagen):** Ausencia de imagen se normaliza obligatoriamente como `NULL`, jamás como cadena vacía (`""`).
  * **INV-PROD-06 (Baja Lógica):** Un producto no se destruye físicamente de la base de datos; transiciona su estado marcando `deleted_at = now()` [ADR-003, T-09].

---

### 2.3 Raíz de Agregado: `Sale` [§2.3]
* **Naturaleza:** Hecho comercial y contable consumado. Representa el acto de compra en mostrador.
* **Atributos:**
  * `id` (`UUID`): Identificador único inmutable.
  * `sold_at` (`DateTime` en UTC): Instante de la transacción (único instante de negocio del sistema, §8).
  * `sold_by` / `sold_by_username` (`string`): Nombre del operador que cobró la venta.
  * `sold_by_user_id` (`UUID`): Clave foránea al usuario operador (FK-4 RESTRICT, pendiente T-12 [§3, §5]).
  * `items` (`IReadOnlyCollection<SaleItem>`): Colección de renglones vendidos.
* **Invariantes Tácticas [§2.3, §4]:**
  * **INV-SALE-01 (Inmutabilidad Absoluta):** Una vez confirmada y persistida, la venta **no puede ser editada ni eliminada bajo ninguna circunstancia** [§1, §2.3, §7.1].
  * **INV-SALE-02 (Al menos una línea):** Una venta vacía no es válida; requiere al menos un artículo para poder confirmarse (`Sale.EnsureConfirmable`).
  * **INV-SALE-03 (No repetición de producto):** Un producto no puede registrarse dos veces como líneas independientes dentro de la misma venta. Si se compra más del mismo artículo, se incrementa la cantidad en la línea existente. Garantizado en dominio por `Sale.AddItem` y en motor por `IX_sale_item_sale_id_product_id` (`UNIQUE INCLUDE quantity, unit_price`) [T-20, §4].
  * **INV-SALE-04 (Total Calculado):** El total de la venta no se almacena en ninguna columna; se calcula dinámicamente sumando los subtotales de sus líneas (`Sale.Total = sum(items.Subtotal)`) [§1, Artículo VII].
  * **INV-SALE-05 (Atomicidad Venta-Stock):** Descontar existencias del producto (`Product.Withdraw`) y crear la línea de venta (`SaleItem`) ocurren como una operación atómica indivisible [§2.3].

---

### 2.4 Entidad Interna: `SaleItem` [§2.4]
* **Naturaleza:** Renglón individual de una venta. Es una entidad interna dependiente del agregado `Sale`.
* **Ciclo de Vida:** No puede instanciarse ni persistirse fuera de su venta cabecera (`FK-2 ON DELETE CASCADE` [§5]). Su constructor es de visibilidad interna (`internal`), invocado exclusivamente por `Sale.AddItem` [§2.4].
* **Atributos:**
  * `id` (`UUID`): Identificador único de la línea.
  * `sale_id` (`UUID`): Clave foránea obligatoria a la venta cabecera (`sale_id NOT NULL`, T-20).
  * `product_id` (`UUID`): Clave foránea restrictiva al producto vendido (FK-3 RESTRICT, T-20).
  * `product_name` (`string`): Nombre del producto **congelado** en el momento de la venta.
  * `category_name` (`string`): Nombre de la categoría **congelado** en el momento de la venta.
  * `unit_price` (`Money`): Precio unitario **congelado** en el momento de la venta.
  * `quantity` (`Quantity`): Número de unidades vendidas.
* **Invariantes Tácticas [§2.4, §4]:**
  * **INV-ITEM-01 (Snapshot Congelado):** `product_name`, `category_name` y `unit_price` son copias inmutables del estado del producto al momento de venderse. No mutan si el producto se renombra o cambia de precio en el futuro (D-06, ADR-004).
  * **INV-ITEM-02 (Sin FK de Categoría en Línea):** `sale_item.category_name` carece deliberadamente de clave foránea hacia la tabla `category` para evitar que renombrar una categoría reescriba el histórico [ADR-004, §2.4, §3].
  * **INV-ITEM-03 (Subtotal Calculado):** El subtotal de la línea (`unit_price * quantity`) se calcula al vuelo y no se almacena en base de datos [§1].

---

### 2.5 Raíz de Agregado: `User` [§2.5]
* **Naturaleza:** Cuenta de operador interno del sistema.
* **Atributos:**
  * `id` (`UUID`): Identificador primario.
  * `username` (`string`): Nombre de usuario (normalizado a minúsculas y sin espacios, máx. 120 caracteres).
  * `password_hash` (`string`): Huella criptográfica irreversible (máx. 512 caracteres).
  * `role` (`string`): Rol operativo, restringido a `'admin'` o `'seller'`.
* **Invariantes Tácticas [§2.5, §4]:**
  * **INV-USER-01 (Unicidad y Normalización de Nombre):** El nombre de usuario se normaliza a minúsculas antes de persistir (`User.NormalizeUsername`) y es único en base de datos (`IX_user_username`).
  * **INV-USER-02 (Cero Exposición de Clave Plana):** El dominio nunca conoce ni manipula la contraseña en claro; solo recibe el hash generado por el puerto criptográfico (D-09).
  * **INV-USER-03 (Conjunto Cerrado de Roles):** El atributo `role` solo admite los valores literales `'admin'` o `'seller'`.

---

## 3. Objetos de Valor (Value Objects)

Los objetos de valor son inmutables por definición, carecen de identificador propio y se comparan por el valor de sus atributos (D-07 [§1, §2]):

### 3.1 `Money` [§1, §2.2]
```csharp
// [Supuesto: Implementación representativa de Value Object en C#]
public sealed record Money(decimal Amount)
{
    public Money
    {
        if (Amount < 0)
            throw new DomainInvariantException("El importe no puede ser negativo.");
        
        // Redondeo obligatorio AwayFromZero a 2 decimales (§2.2)
        Amount = Math.Round(Amount, 2, MidpointRounding.AwayFromZero);
    }

    public static Money operator +(Money a, Money b) => new(a.Amount + b.Amount);
    public static Money operator *(Money a, Quantity q) => new(a.Amount * q.Value);
}
```
* **Características:** Monomoneda estricta por diseño (sin atributo de divisa, D-05 [§1, §3]). Permite `new Money(0)` pero prohíbe negativos [§2.2].

### 3.2 `Quantity` [§1, §2.4]
```csharp
public sealed record Quantity(int Value)
{
    public Quantity
    {
        if (Value <= 0)
            throw new DomainInvariantException("La cantidad debe ser estrictamente positiva (> 0).");
    }
}
```
* **Características:** Modela unidades físicas de venta. Rechaza cero y números negativos [§2.4].

### 3.3 `DateRange` [§1]
```csharp
public sealed record DateRange(DateTime StartUtc, DateTime EndUtc)
{
    public DateRange
    {
        if (EndUtc < StartUtc)
            throw new DomainInvariantException("La fecha de fin no puede ser anterior a la de inicio.");
    }
}
```
* **Características:** Objeto de valor de la capa de aplicación para definir la ventana temporal de reportes de ventas (Q9 [§6.1]).
