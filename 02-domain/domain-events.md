# Eventos de Dominio — Simple Stock Flow

> **Documento:** `02-domain/domain-events.md`  
> **Sistema:** Simple Stock Flow  
> **Fuente de Verdad:** Deducido de las transiciones de estado, operaciones atómicas e invariantes de [`spec/data-model.md`](../spec/data-model.md).

---

## 1. Naturaleza de los Eventos de Dominio

Un **Evento de Dominio** es un hecho consumado que ocurrió en el negocio, inmutable y nombrado siempre en tiempo pasado.

En Simple Stock Flow, los eventos comunican hechos relevantes que ocurren dentro de los agregados (`Product`, `Sale`, `User`), permitiendo reacciones desacopladas (auditoría, alertas de inventario o integración con sistemas externos futuros `[Supuesto]`):

```
Comando: RecordSale  ──►  Agregado Sale  ──►  Evento: SaleConfirmed
Comando: ChangePrice ──►  Agregado Product ──►  Evento: ProductPriceChanged
```

---

## 2. Catálogo de Eventos de Dominio

### 2.1 Evento: `SaleConfirmed`
* **Agregado Emisor:** `Sale` (Raíz) [§2.3]
* **Disparador:** La venta es confirmada con éxito tras validar que tiene al menos una línea, que no hay productos duplicados y que el stock fue retirado de cada producto [§2.3, §2.4].
* **Garantía:** Persistido en la misma transacción atómica de base de datos que las tablas `sales.sale` y `sales.sale_item`.

#### Estructura del Evento (JSON Schema):
```json
{
  "eventId": "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d",
  "eventType": "SaleConfirmed",
  "occurredAt": "2026-09-19T14:30:00Z",
  "aggregateId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "aggregateType": "Sale",
  "payload": {
    "saleId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "soldAt": "2026-09-19T14:30:00Z",
    "soldByUsername": "ana",
    "soldByUserId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
    "totalAmount": 75.50,
    "items": [
      {
        "productId": "8f8b34f2-9594-4d8b-90f9-2c7c00e12345",
        "productName": "Martillo de Bola",
        "categoryName": "Herramientas",
        "unitPrice": 25.00,
        "quantity": 2,
        "subtotal": 50.00
      },
      {
        "productId": "7e7a23e1-8483-3c7a-80e8-1b6b99d09876",
        "productName": "Cinta Aislante",
        "categoryName": "Electricidad",
        "unitPrice": 12.75,
        "quantity": 2,
        "subtotal": 25.50
      }
    ]
  }
}
```

---

### 2.2 Evento: `ProductStockWithdrawn`
* **Agregado Emisor:** `Product` (Raíz) [§2.2]
* **Disparador:** Invocación exitosa de `Product.Withdraw(quantity)` durante el flujo de una venta en mostrador [§2.2, §2.3].
* **Importancia:** Registra la salida física de inventario para fines de trazabilidad de flujo de stock.

#### Estructura del Evento (JSON Schema):
```json
{
  "eventId": "b2c3d4e5-f6a7-8b9c-0d1e-2f3a4b5c6d7e",
  "eventType": "ProductStockWithdrawn",
  "occurredAt": "2026-09-19T14:30:00Z",
  "aggregateId": "8f8b34f2-9594-4d8b-90f9-2c7c00e12345",
  "aggregateType": "Product",
  "payload": {
    "productId": "8f8b34f2-9594-4d8b-90f9-2c7c00e12345",
    "quantityWithdrawn": 2,
    "remainingStock": 8,
    "associatedSaleId": "f47ac10b-58cc-4372-a567-0e02b2c3d479"
  }
}
```

---

### 2.3 Evento: `ProductOutOfStock`
* **Agregado Emisor:** `Product` (Raíz) [§2.2]
* **Disparador:** Se emite cuando, tras un retiro de existencias, el stock disponible alcanza exactamente cero (`remainingStock == 0`) [ADR-002, §2.2].
* **Impacto:** Permite al sistema alertar en pantalla que el producto se ha agotado y debe solicitarse restock `[Supuesto]`.

#### Estructura del Evento (JSON Schema):
```json
{
  "eventId": "c3d4e5f6-a7b8-9c0d-1e2f-3a4b5c6d7e8f",
  "eventType": "ProductOutOfStock",
  "occurredAt": "2026-09-19T14:35:12Z",
  "aggregateId": "8f8b34f2-9594-4d8b-90f9-2c7c00e12345",
  "aggregateType": "Product",
  "payload": {
    "productId": "8f8b34f2-9594-4d8b-90f9-2c7c00e12345",
    "productName": "Martillo de Bola",
    "categoryId": "22222222-2222-4222-8222-222222222222"
  }
}
```

---

### 2.4 Evento: `ProductPriceChanged`
* **Agregado Emisor:** `Product` (Raíz) [§2.2]
* **Disparador:** El Administrador ejecuta `Product.ChangePrice(newPrice)` asignando un nuevo valor comercial vigente en catálogo [§2.2].

#### Estructura del Evento (JSON Schema):
```json
{
  "eventId": "d4e5f6a7-b8c9-0d1e-2f3a-4b5c6d7e8f9a",
  "eventType": "ProductPriceChanged",
  "occurredAt": "2026-09-20T09:15:00Z",
  "aggregateId": "8f8b34f2-9594-4d8b-90f9-2c7c00e12345",
  "aggregateType": "Product",
  "payload": {
    "productId": "8f8b34f2-9594-4d8b-90f9-2c7c00e12345",
    "oldPrice": 25.00,
    "newPrice": 28.50,
    "changedByUsername": "admin"
  }
}
```

---

### 2.5 Evento: `ProductSoftDeleted`
* **Agregado Emisor:** `Product` (Raíz) [§2.2]
* **Disparador:** El Administrador da de baja el producto asignando `deleted_at = now()` en PostgreSQL [ADR-003, T-09, §2.2].

#### Estructura del Evento (JSON Schema):
```json
{
  "eventId": "e5f6a7b8-c9d0-1e2f-3a4b-5c6d7e8f9a0b",
  "eventType": "ProductSoftDeleted",
  "occurredAt": "2026-09-21T18:00:00Z",
  "aggregateId": "8f8b34f2-9594-4d8b-90f9-2c7c00e12345",
  "aggregateType": "Product",
  "payload": {
    "productId": "8f8b34f2-9594-4d8b-90f9-2c7c00e12345",
    "deletedAt": "2026-09-21T18:00:00Z"
  }
}
```
