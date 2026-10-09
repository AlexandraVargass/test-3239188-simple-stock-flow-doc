# 05 — Arquitectura del Sistema: Simple Stock Flow

> **Reto SDD · Ficha ADSO 3239188**  
> **Fase 1 de 6:** Reconstrucción de la Arquitectura hacia atrás desde el modelo de datos.  
> **Único insumo base:** [`spec/data-model.md`](../spec/data-model.md).  
> **Regla de Trazabilidad:** Cada componente, puerto, agregado y decisión cita la sección exacta del modelo de datos (`§X.Y`, `FK-N`, `D-NN`, `ADR-NNN`, `Q-N`). Lo no explícito se marca formalmente como **`[Supuesto]`**.

---

## Índice de Documentos de esta Carpeta

La carpeta `05-architecture/` descompone la solución en tres documentos complementarios y exhaustivos:

| Documento | Enfoque | Contenido Principal |
| :--- | :--- | :--- |
| **[overview.md](./overview.md)** | **Visión Global y C4** | Estilo arquitectónico, C4 Contexto / Contenedores / Componentes, topología de datos en PostgreSQL, concurrencia optimista y manejo de binarios externos. |
| **[hexagonal-architecture.md](./hexagonal-architecture.md)** | **Estructura Hexagonal** | Organización interna (Dominio, Aplicación, Adaptadores), catálogo de Puertos de Entrada (Casos de Uso) y Puertos de Salida (SPIs) con sus firmas y flujos. |
| **[pattern-guide.md](./pattern-guide.md)** | **Patrones y Reglas** | Diseño táctico DDD (Agregados, Value Objects), patrón *Frozen Snapshot*, política de claves foráneas y la matriz exhaustiva de **dónde vive cada regla (Motor vs. Dominio)**. |

---

## Síntesis de Evidencia Arquitectónica (Extraída del Modelo)

Esta tabla resume cómo cada aspecto de la arquitectura se deriva directamente de las especificaciones y mediciones en PostgreSQL reportadas en `spec/data-model.md`:

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                             MATRIZ DE TRAZABILIDAD ARQUITECTÓNICA                          │
├─────────────────────────┬───────────────────────────────────┬───────────────────────────────┤
│ Dimensión Arquitectónica│ Decisión Dedicada del Sistema     │ Evidencia / Cita en el Modelo │
├─────────────────────────┼───────────────────────────────────┼───────────────────────────────┤
│ Estilo del Sistema      │ Monolito Modular Hexagonal        │ §2.5, §12 ("src/domain",      │
│                         │ (Puertos y Adaptadores)           │ "src/adapters/outbound/...")  │
├─────────────────────────┼───────────────────────────────────┼───────────────────────────────┤
│ Lenguaje y Runtime      │ C# / .NET                         │ §0 ("DbSet<Product>", C#),    │
│                         │                                   │ §2.2 ("MidpointRounding")     │
├─────────────────────────┼───────────────────────────────────┼───────────────────────────────┤
│ Acceso a Datos (ORM)    │ Entity Framework Core             │ §0, §3 ("propiedades sombra", │
│                         │                                   │ "HasForeignKey"), §3.1, §3.2  │
├─────────────────────────┼───────────────────────────────────┼───────────────────────────────┤
│ Motor de Base de Datos  │ PostgreSQL 16.14                  │ Encabezado, §10               │
│                         │ (BD simple_stock_flow, sales)     │ (verificado en pg_catalog)    │
├─────────────────────────┼───────────────────────────────────┼───────────────────────────────┤
│ Manejo de Divisa        │ Monomoneda estricta por diseño    │ §1, §3 (D-05: sin columna     │
│                         │ (sin conversión ni divisa en BD)  │ de moneda en ninguna tabla)   │
├─────────────────────────┼───────────────────────────────────┼───────────────────────────────┤
│ Manejo Temporal         │ UTC obligatorio (timestamptz)     │ §3 (todas las marcas de       │
│                         │                                   │ tiempo en timestamptz)        │
├─────────────────────────┼───────────────────────────────────┼───────────────────────────────┤
│ Concurrencia de Stock   │ Optimista en BD mediante xmin     │ §2.2 (D-04, T-10), §3         │
│                         │ de PostgreSQL (sin bloqueos pes.) │ (columna del sistema xmin)    │
├─────────────────────────┼───────────────────────────────────┼───────────────────────────────┤
│ Gestión de Imágenes     │ Clave opaca externa (image_key)   │ §1, §2.2 (D-08), §7.1         │
│                         │ Desacoplada de la BD relacional   │ (varchar(512), borrado seguro)│
├─────────────────────────┼───────────────────────────────────┼───────────────────────────────┤
│ Auditoría del Sistema   │ Cerrada: Sin created_at/updated_at│ §8 (decisión cerrada: sin     │
│                         │ solo sold_at y deleted_at         │ columnas forenses para nadie) │
└─────────────────────────┴───────────────────────────────────┴───────────────────────────────┘
```
