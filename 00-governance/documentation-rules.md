# Reglas de Documentación — Simple Stock Flow

> **Documento:** `00-governance/documentation-rules.md`  
> **Sistema:** Simple Stock Flow  
> **Principio Fundamental:** *"La documentación es código. Si no está actualizada o contradice el motor de base de datos, está rota."*

---

## 1. Política de Idioma del Proyecto [§0, §1]

Siguiendo el estándar fijado en la sección 0 y sección 1 de `spec/data-model.md`:

| Artefacto / Elemento | Idioma Exigido | Justificación y Regla del Proyecto |
| :--- | :--- | :--- |
| **Código fuente (C#)** | **Inglés** | Nombres de clases, variables, métodos e interfaces (`Product`, `unit_price`, `Withdraw`). |
| **Tablas y Columnas SQL** | **Inglés** | Esquema `sales`, nombres en singular y `snake_case` (`product`, `sale_item`, `unit_price`) [§0]. |
| **Commits de Git** | **Inglés** | Conventional Commits en imperativo (`feat(sale): implement atomic checkout`). |
| **Documentación en Markdown** | **Español** | Toda la prosa explicativa, justificaciones, historias de usuario y criterios Gherkin se redactan en español formal. |
| **Términos de Negocio** | **Español con mapeo en inglés** | El glosario de términos se explica funcionalmente en español y referencia su clase técnica en inglés [§1]. |

> **Regla de Consistencia:** Está terminantemente prohibido mezclar idiomas dentro del mismo artefacto (por ejemplo, escribir explicaciones en inglés dentro de un documento en español, o usar nombres de variables en español en C#).

---

## 2. Nomenclatura y Estructura de Archivos

1. **Carpetas del Sistema:** Conservan el prefijo numérico oficial de la gobernanza SDD (`00-governance/`, `01-context/`, `02-domain/`, `03-product/`, `04-requirements/`, `05-architecture/`).
2. **Documentos de Contenido:** Se nombran en `kebab-case.md` (ejemplo: `entities-and-rules.md`, `problem-framing.md`, `non-functional.md`).
3. **Puntos de Entrada:** Cada carpeta contiene obligatoriamente un archivo `README.md` que explica el contenido de la sección y ofrece un índice de navegación.
4. **Plantillas:** Llevan el prefijo `_` para ordenarse al inicio (ejemplo: `_template-hu.md`, `_template-adr.md`).

---

## 3. Jerarquía de Fuentes de Verdad [§preámbulo]

Si se presenta una discrepancia entre documentos y código:

1. **Gana el Motor de Base de Datos:** Si un documento afirma que existe una restricción o columna y PostgreSQL no la tiene en `information_schema` o `pg_catalog`, **el documento está equivocado** y debe corregirse inmediatamente [§preámbulo, §10].
2. **Gana la Especificación Física (`spec/data-model.md`):** Gobierna sobre borradores previos o supuestos funcionales.
3. **Gana el Código de Dominio sobre los Supuestos:** La lógica implementada en C# y probada con tests unitarios valida las invariantes de dominio.

---

## 4. Trazabilidad de Afirmaciones (Regla del Reto)

Toda afirmación técnica o de requerimientos debe poder rastrearse al modelo de datos citando:
* Número de sección (ejemplo: `§2.3`, `§4`, `§11.1`).
* Clave foránea o restricción (ejemplo: `FK-1`, `FK-2`, `ck_product_stock_non_negative`).
* Decisión técnica (ejemplo: `D-05`, `D-06`, `D-08`, `DP-02`, `DP-03`).
* Registro de arquitectura (ejemplo: `ADR-001`, `ADR-002`, `ADR-003`, `ADR-004`).

Si un dato o flujo no se deduce directamente del modelo de datos de producción, **debe marcarse explícitamente con la etiqueta `[Supuesto]`**.
