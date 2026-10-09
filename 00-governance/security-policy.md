# Política de Seguridad — Simple Stock Flow

> **Documento:** `00-governance/security-policy.md`  
> **Sistema:** Simple Stock Flow  
> **Ámbito:** Seguridad de la información, protección de credenciales y privacidad operativa.

---

## 1. Principios de Seguridad del Sistema

1. **Defensa en Profundidad:** Múltiples barreras protegen los datos del negocio (validaciones en C#, restricciones en EF Core y restricciones físicas `CHECK`/`FK` en PostgreSQL).
2. **Menor Privilegio:** Cada operador tiene acceso estrictamente a las operaciones de su rol (`admin` o `seller`).
3. **Falla Segura (*Fail Secure*):** Ante cualquier error o ambigüedad, el sistema aborta la transacción y deniega el acceso.
4. **Minimización de Datos Personales:** El sistema almacena únicamente la información imprescindible para atribuir la autoría de las ventas en el mostrador [§1, §7].

---

## 2. Control de Acceso y Roles (RBAC) [§1, §2.5, §7]

El sistema establece **dos roles de usuario cerrados**:

| Rol | Atribución y Alcance | Permisos del Sistema | Restricciones de Seguridad |
| :--- | :--- | :--- | :--- |
| **`admin`** | Administrador del negocio | Gestión de catálogo (crear, cambiar precio, restock, baja lógica), aprovisionamiento de vendedores, consulta de reportes consolidados Q9. | No puede asignarse dinámicamente en tiempo de ejecución; solo se aprovisiona desde el entorno en el despliegue (DP-04, §9.2). |
| **`seller`** | Vendedor de mostrador | Búsqueda de productos activos en mostrador (Q1), consulta de stock y registro atómico de ventas presenciales (Q3, Q6). | No puede modificar precios de catálogo, dar de baja productos ni consultar reportes consolidados de otros períodos. |

---

## 3. Gestión y Protección de Credenciales (D-09) [§1, §2.5, §7]

1. **Prohibición de Contraseñas en Claro:** El dominio nunca conoce ni persiste contraseñas en texto plano. La transformación ocurre en el puerto criptográfico `IPasswordHasher` (D-09).
2. **Algoritmo Exigido:** Función hash adaptativa de alta seguridad (bcrypt con factor de coste >= 12 o Argon2id) `[Supuesto]`.
3. **Aislamiento en Almacenamiento:** El campo `user.password_hash` (`varchar(512)`) **nunca se indexa** en la base de datos [§7, §10.3]. No es una decisión de rendimiento, sino de aislamiento de secretos.
4. **Cero Fugas de Secretos:** El hash de contraseña está excluido de cualquier respuesta JSON de la API, trazas de excepción, registros de log y modelos de proyección [§7].

---

## 4. Política de Privacidad y Retención de Datos [§7, §7.1]

* **Inexistencia de Datos de Clientes:** Simple Stock Flow no captura nombres, números de identificación, teléfonos ni correos de compradores [§1, §7].
* **Privacidad del Personal Operativo (DP-02):** El reporte comercial agregado Q9 prohíbe explícitamente desglosar ventas por vendedor o cajero [§7, §7.1], protegiendo el entorno laboral y concentrando la analítica en el movimiento de producto.
* **Retención Indefinida de Ventas:** Por tratarse de hechos contables y fiscales, las ventas (`sale`) y sus líneas (`sale_item`) poseen retención indefinida y **no pueden ser eliminadas ni editadas** [§7.1].
