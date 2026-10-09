# Convenciones Ágiles — Simple Stock Flow

> **Documento:** `00-governance/agile-conventions.md`  
> **Sistema:** Simple Stock Flow  
> **Metodología:** Scrum / Kanban adaptado a proyectos formativos de desarrollo de software (ADSO - SENA).

---

## 1. Ciclo de Sprints y Ceremonias

El desarrollo de Simple Stock Flow se organiza en **Sprints de 2 semanas de duración**:

| Ceremonia | Frecuencia y Duración | Objetivo Principal | Participantes |
| :--- | :--- | :--- | :--- |
| **Sprint Planning** | Primer día del Sprint (2 horas) | Seleccionar y comprometer las Historias de Usuario del Product Backlog que formarán el Sprint Backlog. | Todo el equipo |
| **Daily Standup** | Diaria (15 minutos máx.) | Sincronización rápida: qué hice ayer, qué haré hoy y qué impedimentos tengo. | Equipo de desarrollo |
| **Backlog Refinement** | Mitad del Sprint (1 hora) | Desglosar épicas, redactar criterios de aceptación Gherkin y validar que las HUs cumplan la Definición de Preparado (DoR). | Product Owner y Tech Lead |
| **Sprint Review** | Último día del Sprint (1 hora) | Demostración funcional en vivo del incremento de software terminado (cumpliendo la DoD) ante el instructor/stakeholders. | Todo el equipo y evaluadores |
| **Retrospectiva** | Tras la Review (45 minutos) | Identificar mejoras del proceso técnico: qué funcionó, qué falló y qué acuerdos de gobernanza se ajustan. | Todo el equipo |

---

## 2. Estimación y Puntos de Historia (Story Points)

Para la estimación del esfuerzo relativo de cada Historia de Usuario se utiliza la **secuencia de Fibonacci modificada**:

$$1, 2, 3, 5, 8, 13, 20$$

### Criterios de Calibración:
* **1 Punto:** Tarea trivial, cambio cosmético o ajuste menor de validación.
* **2–3 Puntos:** HU estándar con CRUD simple, puertos definidos y pruebas unitarias directas (ej: `HU-CAT-03` Restock de inventario).
* **5 Puntos:** HU con lógica de negocio moderada, múltiples entidades o invariantes críticas (ej: `HU-CAT-01` Alta de producto con validaciones y subida de imagen opaca).
* **8 Puntos:** HU compleja con alta transaccionalidad, concurrencia o interacción atómica entre agregados (ej: `HU-SALE-01` Venta en mostrador con descuento de stock y datos congelados).
* **13+ Puntos:** Demasiado grande para entrar a un sprint. Debe desglosarse obligatoriamente en dos o más historias durante el refinamiento.
