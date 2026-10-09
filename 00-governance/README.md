# 00 — Gobernanza del Equipo: Simple Stock Flow

> **Reto SDD · Ficha ADSO 3239188**  
> **Marco Metodológico:** Adaptado de la base de gobernanza de software para regir el desarrollo, control de cambios y calidad de **Simple Stock Flow**.  
> **Regla de Oro:** Todo acuerdo en este documento es vinculante para el equipo de desarrollo. Si una decisión local entra en conflicto con la gobernanza, prevalece este marco salvo aprobación expresa de un ADR.

---

## Índice de Documentos de Gobernanza

| Documento | Propósito |
| :--- | :--- |
| **[git-conventions.md](./git-conventions.md)** | Estrategia de ramas (GitFlow simplificado), formato de Conventional Commits y política de Pull Requests para el repositorio C# / .NET. |
| **[agile-conventions.md](./agile-conventions.md)** | Ciclo de Sprints, ceremonias ágiles, estimación en Story Points y gestión del backlog. |
| **[definition-of-done.md](./definition-of-done.md)** | Lista de chequeo obligatoria que toda Historia de Usuario debe cumplir para considerarse terminada (código, pruebas, migraciones EF Core). |
| **[definition-of-ready.md](./definition-of-ready.md)** | Criterios mínimos que una Historia de Usuario debe satisfacer antes de ingresar a un Sprint de desarrollo. |
| **[documentation-rules.md](./documentation-rules.md)** | Normas de redacción, convenciones de idioma (código en inglés, documentación en español), nombres de archivos y trazabilidad con `spec/data-model.md`. |
| **[security-policy.md](./security-policy.md)** | Política de seguridad del sistema: gestión de accesos para roles `admin` y `seller`, protección de credenciales (D-09) y privacidad de datos (§7). |
| **[security-rules.md](./security-rules.md)** | Reglas técnicas de código seguro alineadas con OWASP Top 10 aplicadas a C# / .NET y PostgreSQL 16. |

---

## Principios Rectores del Proyecto

1. **La Base de Datos es la Fuente de Integridad:** Las invariantes que puedan bajarse al motor PostgreSQL se defienden con restricciones físicas (`ck_product_stock_non_negative`, claves foráneas restrictivas e índices únicos) [ADR-002, §4].
2. **El Esquema lo Gobiernan las Migraciones:** Ningún cambio de DDL se ejecuta manualmente en base de datos; todo cambio estructural viaja exclusivamente como migración versionada de Entity Framework Core [ADR-001, §3.2].
3. **El Pasado Contable es Inmutable:** Ningún cambio funcional puede alterar los registros históricos de ventas ni modificar reportes de períodos contables ya cerrados (D-06, ADR-004 [§1, §11.1]).
4. **La Documentación es Código:** Si un cambio en el código altera una regla o contrato y la documentación no se actualiza, la Historia de Usuario no pasa la Definición de Terminado (DoD).
