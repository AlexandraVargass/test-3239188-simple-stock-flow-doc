# Encuadre del Problema (Problem Framing)

> **Documento:** `03-product/problem-framing.md`  
> **Sistema:** Simple Stock Flow  
> **Fuente de Verdad Técnica:** Deducido de las limitaciones, reglas e invariantes de [`spec/data-model.md`](../spec/data-model.md).

---

## 1. El Contexto Comercial: Venta en Mostrador

El negocio al que sirve Simple Stock Flow corresponde a comercios de venta directa al público en mostrador —típicamente almacenes de suministros, ferreterías, depósitos de materiales o tiendas técnicas, como lo demuestran de forma incontestable las cinco categorías semilla fijas sembradas en la base de datos: **General, Herramientas, Electricidad, Fontanería y Pinturas** [§9.1].

En este entorno, la velocidad de despacho es crítica, los precios de los insumos cambian con frecuencia por inflación o costos de reposición, y la operación diaria es atendida por varios vendedores trabajando simultáneamente en turnos de caja [§2.5, §6.1 Q3, Q10].

---

## 2. Los Cuatro Dolores Centrales del Negocio

```
┌────────────────────────────────────────────────────────────────────────┐
│                   LOS CUATRO DOLORES DEL NEGOCIO                       │
├──────────────────────────┬─────────────────────────────────────────────┤
│ 1. Sobreventas y         │ Múltiples vendedores en mostrador intentan  │
│    Descuadre de Stock    │ vender el mismo producto escaso en simultáneo│
├──────────────────────────┼─────────────────────────────────────────────┤
│ 2. Corrupción Histórica  │ Actualizar el precio o categoría de un ítem │
│    de Reportes Pasados   │ reescribe retrospectivamente meses contables │
├──────────────────────────┼─────────────────────────────────────────────┤
│ 3. Sobreingeniería y     │ Sistemas que imponen multimoneda, pasarelas │
│    Lentitud de ERPs      │ y módulos complejos que frenan la caja       │
├──────────────────────────┼─────────────────────────────────────────────┤
│ 4. Riesgo Legal de       │ Almacenar datos personales de clientes y    │
│    Privacidad y Tarjetas │ tarjetas cuando el negocio solo cobra en caja│
└──────────────────────────┴─────────────────────────────────────────────┘
```

---

### Dolor 1: Sobreventas y Ruptura de Inventario en Horas Pico [§2.2, §2.3, ADR-002]
* **Situación sin el sistema:** En horas de alta afluencia, dos vendedores consultan la existencia de la última unidad de un producto (por ejemplo, un taladro de alta gama). Ambos ven disponibilidad en una planilla o sistema desconectado y ambos concretan la venta con sus clientes.
* **Impacto en el negocio:** Incumplimiento al cliente, pérdida de reputación comercial, tiempo perdido gestionando devoluciones y descuadre de inventario físico vs. contable.
* **Cómo lo resuelve el modelo:**
  1. Descontar el stock y registrar la línea de venta son **una sola operación atómica** dentro de la misma transacción de base de datos (`Sale.AddItem` invoca `Product.Withdraw`) [§2.3].
  2. Concurrencia optimista mediante `xmin` en PostgreSQL (D-04 [§2.2, T-10]): la segunda transacción es rechazada automáticamente sin bloquear las cajas.
  3. Respaldo definitivo en el motor: la restricción física `ck_product_stock_non_negative` (`stock >= 0`) imposibilita físicamente que el inventario caiga por debajo de cero, incluso si un script o consulta externa intentara escribir en la base [ADR-002, §4].

---

### Dolor 2: La Corrupción de Reportes Contables de Períodos Cerrados [D-06, ADR-004, §1, §2.4, §11.1]
* **Situación sin el sistema:** En la mayoría de aplicaciones convencionales, una línea de venta guarda únicamente el `product_id`. Cuando el sistema emite el reporte de ventas del mes pasado, hace un `JOIN` hacia la tabla de productos para leer el nombre, el precio y la categoría vigentes.
* **Impacto en el negocio:** Si un artículo que costaba $10 en enero se sube a $15 en febrero, o si se reclasifica de "Herramientas" a "Ferretería", **el balance contable de enero cambia retrospectivamente al consultarlo en febrero**. Esto invalida auditorías fiscales y destruye la estabilidad de los libros contables.
* **Cómo lo resuelve el modelo (Patrón *Frozen Snapshot*):**
  1. La línea de venta `sale_item` guarda una **copia congelada del precio unitario, del nombre del producto y del nombre de la categoría en el instante exacto de la venta** [§1, §2.4, D-06, ADR-004].
  2. `sale_item.category_name` **carece deliberadamente de clave foránea hacia la tabla `category`** [§2.4, §3]: si tuviera clave foránea, renombrar una categoría reescribiría el histórico.
  3. El reporte agregado comercial (Q9) agrupa estrictamente por los valores congelados (`product_id`, `product_name`, `category_name`), garantizando que **un reporte de un período cerrado jamás cambie su resultado en el tiempo** [§11.1 H-1].

---

### Dolor 3: Sobreingeniería, Lentitud y Complejidad Innecesaria de los ERPs [§1, §3, §8]
* **Situación sin el sistema:** Los ERPs corporativos obligan a configurar tablas de divisas, tipos de cambio diarios, árboles jerárquicos de categorías infinitos, variantes complejas de producto y decenas de campos de auditoría (`created_at`, `updated_at`, `created_by`, etc.) con disparadores en base de datos.
* **Impacto en el negocio:** Lentitud intolerable al despachar una fila de clientes en mostrador, alta tasa de error de digitación por formularios sobrecargados y costos astronómicos de mantenimiento de software.
* **Cómo lo resuelve el modelo:**
  1. **Monomoneda estricta por diseño (D-05):** No existe columna de divisa en ninguna tabla [§1, §3]. Toda la lógica opera bajo una moneda fija.
  2. **Catálogo Esencial (DP-03):** El producto posee exclusivamente nombre, precio, stock, categoría e imagen opcional, y nada más [§1, §12]. Sin descripciones kilométricas, sin SKUs redundantes.
  3. **Sin sobrecosto de auditoría forense (§8):** No se crean columnas `created_at` / `updated_at` ni disparadores ocultos que ralenticen las inserciones. La única marca temporal es el instante real de la venta (`sale.sold_at` en UTC).

---

### Dolor 4: Exposición a Riesgos Regulatorios de Privacidad y Medios de Pago [§1, §7]
* **Situación sin el sistema:** Sistemas que registran innecesariamente cédulas, nombres, teléfonos y datos de tarjetas de crédito de compradores para compras presenciales menores en mostrador.
* **Impacto en el negocio:** Obligación de responder ante estrictas regulaciones de protección de datos (Habeas Data) y normas de seguridad de medios de pago (PCI-DSS), con riesgo de sanciones por filtraciones.
* **Cómo lo resuelve el modelo:**
  1. **Minimización Radical de Datos:** El sistema no tiene entidad comprador ni cliente final [§1, §7].
  2. La venta registra únicamente la autoría del **operador interno de la empresa** que cobró en la caja (`sold_by` / `sold_by_username`) [§3, §7].
  3. El reporte agregado comercial Q9 prohíbe el desglose por vendedor (DP-02 [§7, §7.1]), protegiendo la privacidad laboral y enfocándose netamente en el movimiento de mercancía.
  4. La contraseña del operador vive bajo hash criptográfico (`password_hash`), jamás viaja en claro y nunca se indexa [§2.5, §7].
