# Definición de Terminado (Definition of Done - DoD)

> **Documento:** `00-governance/definition-of-done.md`  
> **Sistema:** Simple Stock Flow  
> **Regla de Calidad:** Una Historia de Usuario se considera **TERMINADA (DONE)** únicamente cuando cumple con la totalidad de los criterios de esta lista de chequeo. Si falta un solo punto, la historia permanece en estado *In Progress*.

---

## Lista de Chequeo Obligatoria (Checklist)

### 1. Código y Arquitectura
- [ ] El código implementa todos los Criterios de Aceptación especificados en formato Gherkin.
- [ ] La solución respeta estrictamente la Arquitectura Hexagonal: el dominio puro no tiene dependencias de Entity Framework Core ni de la infraestructura web.
- [ ] Los Value Objects (`Money`, `Quantity`) son inmutables y encapsulan sus propias validaciones.
- [ ] El código pasa el análisis estático y formateo oficial (`dotnet format`) sin advertencias.
- [ ] Fue revisado y aprobado por al menos un compañero de equipo en Pull Request.

### 2. Base de Datos y Persistencia
- [ ] Si la HU introdujo o modificó entidades/columnas, se generó la migración correspondiente de Entity Framework Core mediante comando CLI (`dotnet ef migrations add ...`) [ADR-001, §3.2].
- [ ] Ningún cambio de DDL fue ejecutado manualmente en PostgreSQL.
- [ ] Las restricciones de motor están verificadas (por ejemplo: `ck_product_stock_non_negative`, claves foráneas con la acción correcta `RESTRICT`/`CASCADE`).
- [ ] Las marcas temporales utilizan estrictamente `timestamptz` en UTC [§3].
- [ ] No se añadieron columnas `created_at` ni `updated_at` (decisión cerrada en §8).

### 3. Pruebas y Cobertura
- [ ] Pruebas unitarias escritas para todas las invariantes del dominio nuevo o modificado (ej: `Product.Withdraw`, cálculo de totales de venta).
- [ ] Cobertura de pruebas unitarias superior al **80% de líneas** en la capa de dominio (`src/domain/`).
- [ ] Pruebas de integración ejecutadas contra contenedor Docker de PostgreSQL 16.14.
- [ ] En casos que tocan stock concurrente, se probó que `xmin` arroja excepción ante sobreescrituras perdidas (D-04, T-10 [§2.2]).
- [ ] Todas las pruebas pasan exitosamente en local y en el pipeline de CI (`dotnet test`).

### 4. Documentación y Trazabilidad
- [ ] Si la HU alteró el comportamiento público, se actualizó la sección correspondiente de `04-requirements/` y `02-domain/`.
- [ ] Si se tomó una decisión arquitectónica no obvia, se redactó o actualizó el respectivo registro ADR en `05-architecture/decisions/`.
- [ ] La documentación conserva las citas al modelo de datos de producción (`spec/data-model.md`).
