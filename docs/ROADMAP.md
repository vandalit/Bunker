# Roadmap - WARZONE

## Fases y Milestones

### Fase 0: Preparación (Concepto)
- ✅ Documentación conceptual y PRD completados.
- ✅ Estructura inicial del repositorio creada.

### Fase 1: MVP (Proof of Concept)
- **Objetivo:** Tener una aplicación funcional que permita registrar inventario y calcular déficit.
- **Milestones:**
  - Autenticación con Google.
  - CRUD de productos (nombre, categoría, unidad, cantidad objetivo y disponible).
  - Visualización de capas (roja, amarilla, verde) con progreso.
  - Registro de movimientos (compra, consumo, merma).
  - Alertas de caducidad (3 días antes).
  - Cálculo de costo estimado para completar cada capa.
  - Panel de estado con resumen de alertas y costos.

### Fase 2: Mejora de Usabilidad
- UI premium con micro‑animaciones y diseño responsive.
- Export/Import de datos en formato JSON.
- Historial de precios y estimaciones de ahorro.

### Fase 3: Escalado y Persistencia
- Integración con backend (Firestore / Supabase) para persistencia multi‑dispositivo.
- Despliegue en Vercel con CI/CD.
- Tests automatizados y métricas de desempeño.

### Fase 4: Extensiones Futuras
- Integración de recetas y planificación de comidas.
- Comparador de precios de supermercados.
- Soporte multi‑hogar y roles de usuarios.

---

*Este roadmap es evolutivo y será refinado a medida que avance el desarrollo.*
