# WARZONE - Documento de Requerimientos del Producto (PRD)

## Visión General

WARZONE es una aplicación web que permite a los usuarios gestionar su reserva doméstica de alimentos, incluyendo inventario, fechas de caducidad, cálculo de déficit, estimación de costos y seguimiento por capas de abastecimiento (roja, amarilla, verde).

## Alcance Funcional Inicial

1. **Autenticación** mediante cuenta Google.
2. **Gestión de inventario**: crear, editar y eliminar productos.
3. **Capas de abastecimiento**: asignar productos a capas roja, amarilla y verde y ver progreso.
4. **Registro de movimientos**: compra, consumo, merma, ajuste.
5. **Alertas de caducidad** y estado temporal.
6. **Cálculo de déficit y costos** para completar capas.
7. **Panel de estado** con resumen de alertas, costos y cobertura estimada.
8. **Persistencia** de datos en backend (se definirá en fase posterior).

## Fuera de Alcance Inicial

- Integración con supermercados o precios dinámicos.
- Planificación de comidas, recetas.
- Gestión multi‑hogar o usuarios compartidos.
- Predicciones avanzadas de consumo.

## Requisitos No Funcionales

- **Seguridad**: autenticación OAuth, aislamiento de datos por usuario.
- **Escalabilidad**: arquitectura preparada para despliegue en Vercel.
- **Usabilidad**: UI limpia, responsive, accesible.
- **Privacidad**: datos almacenados bajo control del usuario.

## Métricas de Éxito

- Usuario puede registrar al menos 10 productos y ver déficit y costo estimado.
- Alertas de caducidad aparecen con al menos 3 días de antelación.
- El tiempo de carga de la página principal < 2 s.

---

*Este documento está sujeto a cambios a medida que el proyecto evoluciona.*
