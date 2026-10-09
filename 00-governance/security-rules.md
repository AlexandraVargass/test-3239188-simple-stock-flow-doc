# Reglas Técnicas de Seguridad de Código

> **Documento:** `00-governance/security-rules.md`  
> **Sistema:** Simple Stock Flow  
> **Referencia:** Controles técnicos obligatorios basados en OWASP Top 10 aplicados al stack C# / .NET y PostgreSQL 16.14.

---

## 1. Controles OWASP Top 10 Aplicados

### A01 — Control de Acceso Roto (Broken Access Control)
* **Regla:** La autorización se valida en la capa de Aplicación (Casos de Uso), nunca confiando en datos enviados arbitrariamente por el cliente.
* **Separación de Roles:** Las rutas de catálogo y reportes verifican que el token contenga el reclamo de rol `admin`. Las rutas de venta verifican `seller` o `admin`.
* **Aprovisionamiento:** Se prohíbe exponer endpoints públicos para asignación del rol `admin` (DP-04, §9.2).

### A02 — Fallos Criptográficos (Cryptographic Failures)
* **Regla:** Queda terminantemente prohibido el uso de MD5, SHA-1 o SHA-256 sin sal para almacenar contraseñas.
* Se utiliza exclusivamente `IPasswordHasher` configurado con bcrypt o Argon2id (D-09 [§2.5]).
* En producción, toda la comunicación HTTP debe realizarse bajo TLS 1.3 / HTTPS obligatorio.

### A03 — Inyección SQL (Injection)
```csharp
// ❌ PROHIBIDO — Concatenación directa de cadenas en SQL
var query = $"SELECT * FROM sales.product WHERE name = '{userInput}'";

// ✅ PERMITIDO — Consultas tipadas con Entity Framework Core
var product = await dbContext.Products
    .Where(p => p.Name == userInput)
    .FirstOrDefaultAsync();

// ✅ PERMITIDO — Consultas SQL sin procesar con parámetros explícitos
var product = await dbContext.Products
    .FromSqlInterpolated($"SELECT * FROM sales.product WHERE name = {userInput}")
    .FirstOrDefaultAsync();
```
* **Regla de Cero Concatenación:** El 100% de las consultas a PostgreSQL deben ser parametrizadas a través de EF Core o Npgsql.

### A04 — Diseño Inseguro (Insecure Design)
* **Concurrencia de Inventario:** Para evitar sobreventas, no se confía en validaciones en memoria sin control de concurrencia. Se implementa obligatoriamente el token `xmin` y la restricción física `ck_product_stock_non_negative` [ADR-002, D-04, §2.2].
* **Baja Lógica:** No se permite el borrado físico de productos para no romper la integridad de las ventas históricas (`FK-3 RESTRICT`) [ADR-003, §5].

### A05 — Desconfiguración de Seguridad (Security Misconfiguration)
* En producción se deshabilitan las páginas de error de desarrollo (`app.UseDeveloperExceptionPage()`).
* Las excepciones de base de datos nunca devuelven sentencias SQL internas o esquemas de tablas al cliente; devuelven mensajes de error genéricos y códigos estandarizados.

### A06 — Vulnerabilidades en Dependencias
* Todo paquete NuGet incorporado debe someterse a análisis de vulnerabilidades mediante `dotnet list package --vulnerable`.
* Ninguna dependencia con vulnerabilidad de severidad Crítica o Alta puede entrar a producción.

### A09 — Fallos de Registro y Monitoreo (Logging Failures)
* Se registran todos los intentos de inicio de sesión fallidos con timestamp en UTC e IP de origen.
* **Prohibición de Datos Sensibles en Logs:** Queda terminantemente prohibido registrar en consola o archivos de log contraseñas, hashes, claves criptográficas o datos de pago [§7].
