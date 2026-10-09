# Historias de Usuario — Backlog del Sistema

> **Documento:** `04-requirements/user-stories.md`  
> **Sistema:** Simple Stock Flow  
> **Fuente de Verdad:** Deducido directamente de las entidades, patrones de acceso y reglas de [`spec/data-model.md`](../spec/data-model.md).

---

## Épica 1: Identidad y Control de Acceso (Operadores Internos)

### HU-IAM-01 — Inicio de Sesión de Operador Interno
* **Épica:** EP-01
* **Prioridad:** Alta (Bloqueante para operar en caja)
* **Patrón de Acceso Asociado:** Q10 (`user` por igualdad exacta de nombre) [§6.1]

> **Como** Operador interno del negocio (Administrador o Vendedor)  
> **Quiero** autenticarme en el sistema ingresando mi nombre de usuario y contraseña  
> **Para** acceder a las funcionalidades correspondientes a mi rol (`admin` o `seller`) sin exponer credenciales.

#### Criterios de Aceptación (Gherkin):
```gherkin
Escenario 1: Inicio de sesión exitoso con normalización de usuario
  Dado que existe un usuario registrado con username "ana" y rol "seller" (§2.5)
  Cuando el operador envía las credenciales con usuario " Ana  " y su contraseña válida en texto plano
  Entonces el sistema normaliza el nombre a minúsculas y sin espacios ("ana") (§2.5, T-20)
  Y el puerto IPasswordHasher verifica exitosamente la clave contra user.password_hash (D-09)
  Y el sistema concede acceso con el rol correspondiente sin retornar el hash en la respuesta (§7)

Escenario 2: Intento con contraseña incorrecta
  Dado que existe el usuario "carlos"
  Cuando el operador ingresa una contraseña que no coincide con password_hash
  Entonces el sistema rechaza la autenticación con error 401 Unauthorized [Supuesto]
  Y no revela si el error se debió a usuario o contraseña inexistente (§7)
```

**Trazabilidad Técnica:**
* Tabla: `sales.user` (`username`, `password_hash`, `role`) [§3, §7].
* Reglas: `IX_user_username` único [§4, §10.3]; rol en `('admin','seller')` [§2.5].

---

### HU-IAM-02 — Aprovisionamiento de Operadores por el Administrador
* **Épica:** EP-01
* **Prioridad:** Media

> **Como** Administrador del sistema (`role = 'admin'`)  
> **Quiero** dar de alta cuentas de operadores para los vendedores del mostrador  
> **Para** atribuir la autoría de las ventas a personas identificadas sin permitir autorregistro anónimo.

#### Criterios de Aceptación (Gherkin):
```gherkin
Escenario 1: Alta exitosa de un vendedor
  Dado que el usuario autenticado tiene rol "admin"
  Cuando registra un nuevo operador con username "Pedro" y rol "seller"
  Entonces el sistema almacena el username normalizado como "pedro"
  Y genera el password_hash mediante el puerto criptográfico (D-09, §2.5)
  Y la cuenta queda disponible para iniciar sesión

Escenario 2: Intento de otorgar rol admin en tiempo de ejecución
  Dado que el sistema tiene una política estricta de aprovisionamiento (DP-04, H-3, §11.1)
  Cuando un usuario intenta crear o elevar una cuenta al rol "admin"
  Entonces la operación es rechazada porque las cuentas admin solo se aprovisionan desde variables de entorno en el despliegue (§9.2, DP-04)
```

---

## Épica 2: Gestión de Catálogo de Productos

### HU-CAT-01 — Registro de Producto en el Catálogo
* **Épica:** EP-02
* **Prioridad:** Alta
* **Patrón de Acceso Asociado:** Q4 (Listar categorías), Q5 (Categoría por ID) [§6.1]

> **Como** Administrador  
> **Quiero** registrar un nuevo producto especificando su nombre, precio, stock inicial, categoría e imagen opcional  
> **Para** ponerlo a disposición inmediata de los vendedores en el punto de venta.

#### Criterios de Aceptación (Gherkin):
```gherkin
Escenario 1: Registro exitoso de producto con categoría válida
  Dado que existen las 5 categorías sembradas en la base de datos (§9.1)
  Cuando el Administrador registra un producto con nombre " Destornillador de Pala ", precio 15.50, stock 10 y category_id de "Herramientas"
  Entonces el nombre se guarda recortado como "Destornillador de Pala" (§2.2)
  Y el precio se valida estrictamente mayor a 0 (Product.ChangePrice, §2.2)
  Y el stock inicial respeta la restricción ck_product_stock_non_negative (§2.2, ADR-002)
  Y la relación con category_id se valida mediante FK-1 RESTRICT (§5)

Escenario 2: Asignación de imagen opcional externa
  Dado que el Administrador adjunta un archivo de imagen válido
  Cuando se procesa el alta del producto
  Entonces el sistema sube el archivo al storage externo y guarda en product.image_key únicamente la clave opaca de 512 caracteres (D-08, §1)
  Y si no se adjunta imagen, product.image_key se guarda como NULL, nunca como cadena vacía (§1, §2.2)
```

---

### HU-CAT-02 — Modificación de Precio de Producto
* **Épica:** EP-02
* **Prioridad:** Alta

> **Como** Administrador  
> **Quiero** actualizar el precio vigente de un producto en el catálogo  
> **Para** reflejar cambios de costos comerciales sin alterar las ventas pasadas ya registradas.

#### Criterios de Aceptación (Gherkin):
```gherkin
Escenario 1: Actualización de precio exitosa
  Dado un producto existente con precio vigente 20.00
  Cuando el Administrador cambia el precio a 25.755
  Entonces el Value Object Money redondea a 2 decimales usando MidpointRounding.AwayFromZero (25.76) (§2.2)
  Y el nuevo precio queda registrado en product.price
  Y ninguna línea de venta previa (sale_item.unit_price) es modificada (§1, §2.4, ADR-004)

Escenario 2: Rechazo de precio no positivo
  Dado un producto existente
  Cuando el Administrador intenta fijar un precio igual a 0 o negativo (-5.00)
  Entonces el dominio rechaza la operación mediante Product.ChangePrice (§2.2, §4)
```

---

### HU-CAT-03 — Reabastecimiento de Inventario (Restock)
* **Épica:** EP-02
* **Prioridad:** Alta
* **Patrón de Acceso Asociado:** Q2 (Producto por ID) [§6.1]

> **Como** Administrador  
> **Quiero** añadir unidades al stock de un producto existente  
> **Para** reponer inventario físico recibido de proveedores.

#### Criterios de Aceptación (Gherkin):
```gherkin
Escenario 1: Entrada de inventario atómica
  Dado un producto con stock actual de 5 unidades
  Cuando el Administrador registra un ingreso de 15 unidades mediante Product.Restock(15)
  Entonces el stock se incrementa a 20 unidades
  Y la columna de sistema xmin de PostgreSQL avanza registrando la nueva versión de la fila (§2.2, D-04)
```

---

### HU-CAT-04 — Baja Lógica de Producto (Soft Delete)
* **Épica:** EP-02
* **Prioridad:** Media
* **Patrón de Acceso Asociado:** Q1, Q3 (Filtro automático sobre productos activos) [§6.1]

> **Como** Administrador  
> **Quiero** retirar un producto del catálogo comercial  
> **Para** que no pueda seguir vendiéndose, sin destruir las ventas históricas asociadas a él.

#### Criterios de Aceptación (Gherkin):
```gherkin
Escenario 1: Retiro de catálogo mediante baja lógica
  Dado un producto activo que ya registra ventas históricas en sale_item
  Cuando el Administrador decide eliminarlo del catálogo
  Entonces el sistema NO ejecuta un DELETE físico en PostgreSQL (§2.2, §7.1)
  Y asigna la fecha y hora UTC en la propiedad sombra product.deleted_at (T-09, ADR-003)
  Y el producto deja de aparecer en las búsquedas de punto de venta (Q1) mediante el filtro global deleted_at IS NULL

Escenario 2: Protección de integridad ante borrado físico
  Dado que un producto posee registros asociados en sale_item
  Si se intentase un DELETE SQL directo por consola sobre la tabla product
  Entonces la clave foránea FK_sale_item_product_product_id con acción ON DELETE RESTRICT bloquea la eliminación y falla ruidosamente (§5 FK-3, ADR-003)
```

---

### HU-CAT-05 — Búsqueda de Productos en el Punto de Venta
* **Épica:** EP-02
* **Prioridad:** Alta
* **Patrón de Acceso Asociado:** Q1 (Buscar producto por texto parcial, categoría, activos) [§6.1]

> **Como** Vendedor en caja  
> **Quiero** buscar productos por coincidencia parcial de nombre o por categoría  
> **Para** seleccionarlos rápidamente durante la atención en el mostrador.

#### Criterios de Aceptación (Gherkin):
```gherkin
Escenario 1: Búsqueda rápida de productos activos
  Dado un catálogo con productos activos y productos dados de baja
  Cuando el Vendedor busca "tubo" filtrado por la categoría "Fontanería"
  Entonces el sistema devuelve únicamente productos donde deleted_at IS NULL ordenados por nombre (§6.1 Q1)
  Y excluye cualquier producto dado de baja lógica
```

---

## Épica 3: Operaciones de Venta en Mostrador

### HU-SALE-01 — Registro Atómico de Venta en Mostrador
* **Épica:** EP-03
* **Prioridad:** Crítica (Flujo principal de valor del negocio)
* **Patrón de Acceso Asociado:** Q3 (Productos por lote), Q6 (Guardar venta con líneas) [§6.1]

> **Como** Vendedor en mostrador  
> **Quiero** registrar la venta de uno o varios productos a un cliente presencial  
> **Para** facturar los artículos, descontar el stock en tiempo real y emitir el comprobante.

#### Criterios de Aceptación (Gherkin):
```gherkin
Escenario 1: Venta exitosa con congelamiento de datos y descuento de stock
  Dado que el producto "Llave Inglesa" tiene stock de 10 unidades a precio $30.00 en categoría "Herramientas"
  Cuando el Vendedor registra la venta de 2 unidades
  Entonces en una sola transacción ACID de base de datos (§2.3):
    1. Se crea la cabecera sales.sale con sold_at en UTC y sold_by con el nombre del operador (§2.3, §3)
    2. Se descuenta el stock del producto a 8 unidades (Product.Withdraw, §2.2)
    3. Se crea sales.sale_item copiando product_name = "Llave Inglesa", category_name = "Herramientas", unit_price = 30.00 y quantity = 2 (D-06, ADR-004)
  Y el total de la venta ($60.00) se calcula en memoria sumando los subtotales, sin guardarse en ninguna columna (§1, Artículo VII)

Escenario 2: Rechazo por stock insuficiente
  Dado que un producto tiene únicamente 1 unidad en stock
  Cuando el Vendedor intenta registrar una venta solicitando 3 unidades
  Entonces la operación es rechazada por el dominio antes de persistir (Product.Withdraw, §2.2)
  Y si concurre una venta paralela en otra caja, la restricción ck_product_stock_non_negative de PostgreSQL garantiza que el stock nunca sea inferior a 0 (ADR-002, §4)

Escenario 3: Rechazo de producto duplicado en la misma venta
  Dado que el Vendedor ya agregó el producto "Pintura Blanca" a la venta actual
  Cuando intenta agregar nuevamente una línea con "Pintura Blanca"
  Entonces el sistema rechaza el duplicado en el dominio y en el motor gracias al índice único IX_sale_item_sale_id_product_id (§2.3, §4, T-20)
  Y exige que se incremente la cantidad en la línea existente

Escenario 4: Rechazo de venta vacía
  Cuando el Vendedor intenta confirmar una venta sin haber agregado ningún producto
  Entonces el agregado rechaza la confirmación mediante Sale.EnsureConfirmable (§2.3)
```

---

### HU-SALE-02 — Consulta de Venta Histórica
* **Épica:** EP-03
* **Prioridad:** Media
* **Patrón de Acceso Asociado:** Q6 (Venta con sus líneas) [§6.1]

> **Como** Administrador o Vendedor  
> **Quiero** consultar el detalle completo de una venta previamente realizada  
> **Para** verificar qué productos se despacharon y el importe cobrado.

#### Criterios de Aceptación (Gherkin):
```gherkin
Escenario 1: Lectura de venta inmutable con datos históricos congelados
  Dado que una venta fue confirmada en el pasado
  Y que posteriormente los productos vendidos cambiaron de precio o nombre en el catálogo
  Cuando se consulta el detalle de la venta por su identificador único (Q6)
  Entonces el sistema devuelve los nombres, categorías y precios unitarios exactamente como quedaron congelados en sale_item (§1, §2.4)
  Y muestra la autoría de quién realizó la venta (sale.sold_by) (§3, §7)
  Y el sistema no permite ninguna opción de edición ni eliminación sobre la venta (§1, §2.3)
```

---

### HU-SALE-03 — Listado Paginado de Ventas por Fecha
* **Épica:** EP-03
* **Prioridad:** Media
* **Patrón de Acceso Asociado:** Q7 (Ventas por rango de fecha) [§6.1]

> **Como** Administrador  
> **Quiero** listar las ventas ocurridas dentro de un rango de fechas con paginación  
> **Para** auditar las operaciones del día o del período comercial.

#### Criterios de Aceptación (Gherkin):
```gherkin
Escenario 1: Listado de ventas por rango cronológico
  Dado un conjunto de ventas registradas en el sistema
  Cuando el Administrador solicita las ventas del rango [2026-09-01, 2026-09-30] en página 1 con tamaño 20
  Entonces el sistema consulta sales.sale apoyándose en el índice IX_sale_sold_at (§6.2, Q7)
  Y retorna las ventas ordenadas por sold_at de forma descendente
```

---

## Épica 4: Reportería Comercial Agregada

### HU-REP-01 — Reporte de Ventas Agregadas por Rango de Fechas
* **Épica:** EP-04
* **Prioridad:** Alta (Patrón de lectura analítica Q9) [§6.1]

> **Como** Administrador  
> **Quiero** generar un reporte consolidado de ventas por producto en un rango temporal  
> **Para** conocer qué artículos generaron mayor volumen de facturación y unidades vendidas sin que el reporte cambie si el catálogo muta.

#### Criterios de Aceptación (Gherkin):
```gherkin
Escenario 1: Agregación comercial agrupando por valores congelados
  Dado que un producto se vendió como "Herramientas" en septiembre y luego se recategorizó como "Ferretería" en octubre
  Cuando el Administrador genera el reporte consolidado para el período septiembre-octubre
  Entonces el reporte agrupa estrictamente por los valores congelados (product_id, product_name, category_name) (§11.1 H-1)
  Y muestra dos líneas separadas con sus respectivos importes y cantidades totales
  Y la consulta se ejecuta directamente en el motor PostgreSQL sin instanciar agregados en memoria (D-06, §1)
  Y la consulta no expone desgloses por vendedor para proteger la privacidad del operador (DP-02, §7, §7.1)
```
