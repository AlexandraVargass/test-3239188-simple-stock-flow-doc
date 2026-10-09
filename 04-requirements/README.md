# 04 — Requisitos del Sistema: Simple Stock Flow

> **Reto SDD · Ficha ADSO 3239188**  
> **Fase 2 de 6:** Reconstrucción de Requisitos e Historias de Usuario desde el Modelo de Datos.  
> **Único insumo base:** [`spec/data-model.md`](../spec/data-model.md).  
> **Regla de Trazabilidad:** Cada historia de usuario y requisito no funcional se fundamenta en las columnas, índices, restricciones y decisiones del modelo (`§X.Y`, `Q-N`, `FK-N`, `D-NN`, `ADR-NNN`). Aquello deducido operativamente se marca formalmente como **`[Supuesto]`**.

---

## Índice de Documentos de esta Sección

Esta carpeta descompone los requerimientos que hacen estrictamente necesario el modelo de datos de producción:

| Documento | Enfoque | Contenido Principal |
| :--- | :--- | :--- |
| **[user-stories.md](./user-stories.md)** | **Historias de Usuario (HUs)** | Backlog de 9 Historias de Usuario organizadas en 4 Épicas, estructuradas con la plantilla estándar (Como / Quiero / Para), Criterios de Aceptación en formato Gherkin (*Given/When/Then*), Definition of Done y trazabilidad a consultas Q1–Q10. |
| **[non-functional.md](./non-functional.md)** | **Requisitos No Funcionales (RNF)** | 7 Requisitos No Funcionales con métricas objetivas cuantificables: concurrencia optimista (`xmin`), rendimiento en caja, seguridad de credenciales (D-09), privacidad de datos personales (§7), precisión aritmética (`AwayFromZero`) y manejo UTC. |
| **[traceability-matrix.md](./traceability-matrix.md)** | **Matriz de Trazabilidad** | Matriz bidireccional que conecta cada Requisito Funcional con su HU, sus RNF asociados, los patrones de acceso (Q1–Q10) y las 22 columnas físicas de PostgreSQL. |

---

## Resumen del Backlog de Requerimientos

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        MAPA DE ÉPICAS E HISTORIAS DE USUARIO                          │
├─────────────────┬─────────────────────────────────────────────────┬────────────────────┤
│ Épica           │ Historias de Usuario Comprendidas               │ Citas al Modelo    │
├─────────────────┼─────────────────────────────────────────────────┼────────────────────┤
│ EP-01: Identidad│ HU-IAM-01: Inicio de sesión de operador interno │ §2.5, §6.1 Q10, §7 │
│ y Acceso        │ HU-IAM-02: Aprovisionamiento de operadores     │ §2.5, §9.2, DP-04  │
├─────────────────┼─────────────────────────────────────────────────┼────────────────────┤
│ EP-02: Catálogo │ HU-CAT-01: Registro de nuevo producto           │ §2.2, §3, FK-1     │
│ de Productos    │ HU-CAT-02: Actualización de precio de catálogo  │ §2.2, D-04, §4     │
│                 │ HU-CAT-03: Reabastecimiento de stock (Restock)  │ §2.2, Q2, D-04     │
│                 │ HU-CAT-04: Baja lógica de producto (Soft Delete)│ §2.2, ADR-003, T-09│
│                 │ HU-CAT-05: Búsqueda y listado de productos      │ §6.1 Q1, T-13      │
├─────────────────┼─────────────────────────────────────────────────┼────────────────────┤
│ EP-03: Ventas   │ HU-SALE-01: Registro atómico de venta mostrador │ §2.3, §2.4, ADR-004│
│ e Inventario    │ HU-SALE-02: Consulta de venta histórica         │ §2.3, §6.1 Q6, §7  │
│                 │ HU-SALE-03: Listado de ventas por fecha         │ §6.1 Q7            │
├─────────────────┼─────────────────────────────────────────────────┼────────────────────┤
│ EP-04: Reportes │ HU-REP-01: Reporte agregado de ventas por rango │ §1, §6.1 Q9, §11.1 │
└─────────────────┴─────────────────────────────────────────────────┴────────────────────┘
```

> **Aviso Clave sobre Actores:** En todo el backlog, los únicos actores son **Administrador (`admin`)** y **Vendedor (`seller`)**. El sistema no modela clientes externos ni compras anónimas [§1, §7].
