# Definición de Preparado (Definition of Ready - DoR)

> **Documento:** `00-governance/definition-of-ready.md`  
> **Sistema:** Simple Stock Flow  
> **Regla de Entrada:** Una Historia de Usuario solo puede ser planificada y tomada en un Sprint si cumple con todos los criterios de esta lista de verificación. Cualquier historia ambigua o incompleta se devuelve a refinamiento.

---

## Lista de Chequeo de Entrada (Checklist)

### 1. Formato y Valor de Negocio
- [ ] La Historia de Usuario está redactada en el formato estándar:  
  *«Como [rol: admin o seller] quiero [acción concreta] para [beneficio medible de negocio]».*
- [ ] El rol de usuario pertenece estrictamente al conjunto cerrado permitido (`admin` o `seller`) [§1, §2.5]. No se aceptan historias redactadas desde la perspectiva de "compradores externos" o "clientes anónimos".
- [ ] La historia aporta valor tangible a uno de los cuatro módulos del sistema: Identidad, Catálogo, Ventas o Reportería.

### 2. Criterios de Aceptación y Pruebas
- [ ] Los Criterios de Aceptación están formalizados en sintaxis Gherkin (`Dado que... Cuando... Entonces...`).
- [ ] Incluye al menos un escenario de camino feliz (*happy path*) y los escenarios de error o borde más relevantes (por ejemplo: stock insuficiente, producto duplicado, precio <= 0).
- [ ] Se especifica claramente cómo se calcula el total o subtotal si la historia involucra importes comerciales.

### 3. Trazabilidad Técnica y Arquitectónica
- [ ] Se identifica claramente qué Agregado de Dominio es el dueño de la operación (`Product`, `Sale` o `User`).
- [ ] Se especifica si las invariantes involucradas viven en el **Motor** (PostgreSQL) o en el **Dominio** (C#) [§4].
- [ ] Si involucra modificaciones de base de datos, se verificó compatibilidad con las 22 columnas físicas y las 4 claves foráneas de `spec/data-model.md`.
- [ ] La historia está estimada por el equipo en la sesión de refinamiento y tiene un tamaño de **8 Story Points o menos** (si tiene 13+, debe dividirse antes de ingresar al Sprint).
