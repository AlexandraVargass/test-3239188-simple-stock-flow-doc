# Convenciones de Git — Simple Stock Flow

> **Documento:** `00-governance/git-conventions.md`  
> **Sistema:** Simple Stock Flow  
> **Ámbito:** Repositorio de código y documentación del proyecto.

---

## 1. Estrategia de Ramas (Branching Strategy)

El proyecto adopta un modelo **GitFlow simplificado**:

```
main        ← Producción / Entregable evaluable. Siempre estable y funcional.
  └── dev   ← Integración continua del equipo. Integración de features revisadas.
        ├── feat/[descripción-kebab-case]   ← Una rama por Historia de Usuario
        ├── fix/[descripción-kebab-case]    ← Corrección de defectos
        ├── chore/[descripción-kebab-case]  ← Infraestructura, Docker, dependencias NuGet
        ├── docs/[descripción-kebab-case]   ← Cambios exclusivos de documentación
        └── hotfix/[descripción-kebab-case] ← Corrección crítica directa a main
```

### Reglas Innegociables de Ramas:
1. **Nadie commitea directamente sobre `main` o `dev`** en flujos ordinarios de equipo.
2. Cada tarea se desarrolla en una rama aislada con el prefijo correspondiente.
3. Se realiza **Pull Request (PR)** obligatorio antes de fusionar a `dev`.
4. Tras la fusión aprobada, la rama de trabajo se elimina para mantener limpio el árbol.

---

## 2. Formato de Commits (Conventional Commits)

Cada commit debe describir claramente el propósito del cambio en tiempo imperativo y en minúsculas:

```
[tipo]([ámbito]): [descripción en imperativo minúsculas sin punto final]

[cuerpo opcional — explica el POR QUÉ del cambio, no el cómo]

[pie opcional — referencias a tareas o HU, ej: Ref: HU-SALE-01, T-20]
```

### Tipos Permitidos:
| Tipo | Cuándo se usa | Ejemplo |
| :--- | :--- | :--- |
| `feat` | Nueva funcionalidad de negocio | `feat(sales): implement atomic sale registration with stock withdrawal` |
| `fix` | Corrección de un defecto | `fix(auth): normalize username to lowercase before lookup` |
| `docs` | Cambios exclusivos de documentación | `docs(architecture): add C4 container diagram and xmin concurrency notes` |
| `refactor` | Refactorización de código sin cambio funcional | `refactor(domain): extract money value object arithmetic operators` |
| `test` | Incorporación o ajuste de pruebas unitarias/integración | `test(product): add concurrency test verifying ck_product_stock_non_negative` |
| `chore` | Configuración, dependencias NuGet, migraciones EF | `chore(database): add EF migration for sale_item foreign key` |

---

## 3. Política de Pull Requests (PR)

Antes de fusionar cualquier cambio a `dev` o `main`:
1. El código debe compilar limpiamente (`dotnet build`) sin advertencias (*warnings as errors*).
2. Todas las pruebas unitarias y de integración deben pasar en verde (`dotnet test`).
3. Si el cambio toca el esquema de base de datos, debe incluir la migración correspondiente de EF Core generada y el DDL no debe ejecutarse a mano [ADR-001, §3.2].
4. Requiere aprobación de al menos un revisor (*Peer Review*).
