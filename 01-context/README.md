# 01 — Contexto del Sistema: Simple Stock Flow

> **Reto SDD · Ficha ADSO 3239188**  
> **Fase 5 de 6:** Reconstrucción de la Descripción General y Alcance del Sistema desde el Modelo de Datos.  
> **Único insumo base:** [`spec/data-model.md`](../spec/data-model.md).  
> **Regla de Trazabilidad:** La descripción del sistema, sus actores, límites e inclusiones/exclusiones de alcance se fundamentan estrictamente en las secciones del modelo (`§1`, `§2`, `§3`, `§5`, `§7`, `§8`, `§9`, `D-05`, `D-06`, `D-08`, `DP-02`, `DP-03`). Aquello inferido del entorno operativo se etiqueta como **`[Supuesto]`**.

---

## Índice de Documentos de esta Sección

Esta carpeta establece el marco de referencia fundamental y las fronteras operativas del sistema:

| Documento | Enfoque | Contenido Principal |
| :--- | :--- | :--- |
| **[overview.md](./overview.md)** | **Descripción General** | Qué es Simple Stock Flow, el problema que resuelve, usuarios principales (`admin` y `seller`), stack tecnológico verificado y estado del sistema. |
| **[scope.md](./scope.md)** | **Declaración de Alcance** | Especificación formal de **In Scope** (lo que el sistema construye y mantiene) y **Out of Scope** explícito (lo que no se construye y por qué, citando decisiones del modelo D-05, DP-02, DP-03 y §8). |

---

## Síntesis Ejecutiva del Contexto

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                          FRONTERAS DEL CONTEXTO: SIMPLE STOCK FLOW                     │
├───────────────────────┬────────────────────────────────────────────┬───────────────────┤
│ Dimensión             │ Alcance del Sistema                        │ Justificación     │
├───────────────────────┼────────────────────────────────────────────┼───────────────────┤
│ Tipo de Aplicación    │ Punto de venta (POS) e inventario interno  │ §1, §2.2, §2.3    │
│                       │ de mostrador en tiempo real                │                   │
├───────────────────────┼────────────────────────────────────────────┼───────────────────┤
│ Límites de Usuarios   │ Solo operadores internos autenticados      │ §1, §2.5, §7      │
│                       │ (Inexistencia de cuentas de clientes)      │                   │
├───────────────────────┼────────────────────────────────────────────┼───────────────────┤
│ Moneda                │ Monomoneda estricta (sin multimoneda)      │ §1, §3 (D-05)     │
├───────────────────────┼────────────────────────────────────────────┼───────────────────┤
│ Auditoría             │ Cerrada: sold_at (UTC) y deleted_at        │ §8                │
│                       │ (Sin columnas created_at / updated_at)     │                   │
├───────────────────────┼────────────────────────────────────────────┼───────────────────┤
│ Ciclo de Ventas       │ Inmutable: Hechos consumados sin edición   │ §1, §2.3, §7.1    │
├───────────────────────┼────────────────────────────────────────────┼───────────────────┤
│ Clasificación         │ Fija: 5 categorías sembradas sin CRUD      │ §1, §2.1, §9.1    │
└───────────────────────┴────────────────────────────────────────────┴───────────────────┘
```
