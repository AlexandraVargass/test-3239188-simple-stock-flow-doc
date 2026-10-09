# 03 — Definición de Producto: Simple Stock Flow

> **Reto SDD · Ficha ADSO 3239188**  
> **Fase 3 de 6:** Reconstrucción de la Visión de Producto y Encuadre del Problema desde el Modelo de Datos.  
> **Único insumo base:** [`spec/data-model.md`](../spec/data-model.md).  
> **Regla de Trazabilidad:** Cada objetivo, dolor de usuario y principio de producto se fundamenta en las decisiones explícitas del modelo (`§1`, `§2`, `§7`, `§8`, `§9`, `D-05`, `D-06`, `D-08`, `DP-02`, `DP-03`). Lo inferido del contexto comercial se señala como **`[Supuesto]`**.

---

## Índice de Documentos de esta Sección

Esta carpeta reconstruye el propósito estratégico y comercial que justifica la existencia del sistema:

| Documento | Enfoque | Contenido Principal |
| :--- | :--- | :--- |
| **[problem-framing.md](./problem-framing.md)** | **Encuadre del Problema** | Análisis del dolor operativo: ventas simultáneas con sobreventa de stock, desajustes en balances contables pasados por mutaciones de catálogo, sobrecostos de ERPs complejos y riesgos innecesarios de privacidad. |
| **[vision.md](./vision.md)** | **Visión del Producto** | Declaración de visión (plantilla Geoffrey Moore), pilares estratégicos de diseño, los 5 principios de producto innegociables (DP-02, DP-03, D-05) y delimitación estricta de alcance comercial. |

---

## Síntesis Estratégica del Producto

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                          RADIOGRAFÍA DE PRODUCTO: SIMPLE STOCK FLOW                    │
├───────────────────────┬────────────────────────────────────────────┬───────────────────┤
│ Dimensión             │ Postura del Producto                       │ Respaldo Técnico  │
├───────────────────────┼────────────────────────────────────────────┼───────────────────┤
│ Propósito Principal   │ Punto de venta (POS) y control de stock    │ §1, §2.2, §2.3    │
│                       │ ágil para mostrador comercial              │                   │
├───────────────────────┼────────────────────────────────────────────┼───────────────────┤
│ Usuarios Objetivo     │ Exclusivamente operadores internos         │ §1, §2.5, §7      │
│                       │ (Administradores y Vendedores de caja)     │                   │
├───────────────────────┼────────────────────────────────────────────┼───────────────────┤
│ Modelo Económico      │ Monomoneda estricto por diseño             │ §1, §3 (D-05)     │
│                       │ (Cero fricción de conversión de divisas)   │                   │
├───────────────────────┼────────────────────────────────────────────┼───────────────────┤
│ Catálogo Comercial    │ Esencial: Nombre, precio, stock, categoría │ §1, §12 (DP-03)   │
│                       │ e imagen opcional (sin campos superfluos)  │                   │
├───────────────────────┼────────────────────────────────────────────┼───────────────────┤
│ Integridad Contable   │ Inmutabilidad de venta y congelamiento de  │ §1, §2.4 (D-06,   │
│                       │ precios y categorías en el instante t      │ ADR-004, §11.1)   │
├───────────────────────┼────────────────────────────────────────────┼───────────────────┤
│ Privacidad / Legal    │ Superficie cero: Sin clientes finales,     │ §1, §7, §7.1      │
│                       │ sin pasarelas y sin tarjetas de crédito    │                   │
└───────────────────────┴────────────────────────────────────────────┴───────────────────┘
```
