# Decisiones Pendientes - Bunker

> Documento vivo para ir resolviendo las definiciones críticas del proyecto

---

## 1. Backend y Base de Datos

**Estado:** ✅ DECIDIDO  
**Prioridad:** 🔴 Crítica — todo depende de esto

### Contexto
- Tienes cuenta Firebase activa
- El modelo de datos es complejo y tiene múltiples facetas
- **Offline-first es mandatorio** — coherente con cultura prep y escenarios de colapso

### Decisión: **SQLite local + Firebase Auth/Storage**

**Arquitectura:**
- **SQLite (expo-sqlite):** Base de datos local, queries SQL completas, cero latencia, funciona sin conexión
- **Firebase Auth:** Login con Google (solo requiere conexión en autenticación inicial)
- **Firebase Storage:** Backup opcional de la BD, sincronización entre dispositivos (export/import)

**Justificación:**
- App debe funcionar sin internet (narrativa prep + escenarios de crisis)
- Queries complejas para dimensiones son naturales en SQL
- Performance máximo (todo local)
- Cero costos por query
- Firebase solo para Auth (lo tienes activo) + backup opcional

**Stack confirmado:**
- DB: expo-sqlite (SQLite)
- Auth: Firebase Auth
- Backup/Sync: Firebase Storage (JSON export/import)
- Estado: Zustand
- ORM: Drizzle (opcional, para type-safety)

---

## 2. Gamificación

**Estado:** ⚪ No analizado  
**Prioridad:** 🟡 Media — afecta UX pero no bloquea MVP

### Opciones

**A) Sistema XP clásico**
- Ganas XP por acciones (compra, consumo, rotación)
- Niveles desbloqueables
- Cada nivel → rewards (items, badges, temas)

**B) Sistema de logros/achievements**
- Badges por milestones
- Más narrativo, menos numérico
- Coleccionable

**C) Híbrido (XP + achievements)**
- XP para progreso continuo
- Badges para hitos específicos

**Decisión:** [ ]

---

## 3. iPhone Duo (foldables)

**Estado:** ⚪ No analizado  
**Prioridad:** 🟢 Baja — showcase de portafolio, no bloqueante

### Opciones

**A) Features dual-screen específicas**
- Inventario en pantalla 1, receta en pantalla 2
- Comparador de productos lado a lado
- Aprovecha hardware único

**B) Diseño adaptado premium**
- Responsive espectacular
- Transiciones smooth entre configuraciones
- Layout optimizado para pantalla grande
- Sin features exclusivas

**Decisión:** [ ]

---

## 4. Monetización

**Estado:** ✅ DECIDIDO  
**Prioridad:** 🟡 Media — define features premium

### Decisión: **Freemium con multi-vault + sync**

**Free (gratis total):**
- 1 vault personal
- Solo propietario la usa
- Puede invitar viewers (solo lectura) — gratis para invitados
- Backup manual
- Mobile (Android/iOS)

**Premium ($2-3 USD/mes o $20/año):**
- Múltiples vaults (familiar, secundaria)
- Invitar editors (lectura + escritura)
- Sync automático multi-device
- Acceso web desde PC
- Backup automático
- Temas premium

**Justificación:**
- Feature real y funcional (no cosmética)
- Familias justifican fácil ($3/mes para todos)
- Diferenciador claro vs gratis

---

## 5. Recetas

**Estado:** ⚪ No analizado  
**Prioridad:** 🟡 Media — core feature pero puede empezar simple

### Opciones

**A) Seed inicial hardcoded**
- 20-30 recetas básicas pre-cargadas
- Mantenido por equipo
- Menos flexible pero funcional desde día 1

**B) User-generated**
- Sistema para que usuarios creen recetas
- Más engagement
- Requiere moderación o calidad variable

**C) Híbrido**
- Seed oficial para empezar
- Permite creación custom
- Lo mejor de ambos mundos pero más complejo

**Decisión:** [ ]

---

## 6. Barcode Scanner

**Estado:** ⚪ No analizado  
**Prioridad:** 🟡 Media — QOL feature útil

### Opciones

**A) MVP (Fase 1)**
- Usar librería React Native (expo-barcode-scanner)
- Integrar con API de productos (OpenFoodFacts)
- Acelera ingreso de datos

**B) V2 (Fase 2+)**
- Priorizar flujo manual primero
- Agregar scanner después de validar core

**Decisión:** [ ]

---

## Resumen de estado

| Decisión | Prioridad | Estado |
|---|---|---|
| Backend/DB | 🔴 Crítica | 🟡 En análisis |
| Gamificación | 🟡 Media | ⚪ Pendiente |
| iPhone Duo | 🟢 Baja | ⚪ Pendiente |
| Monetización | 🟢 Baja | ⚪ Pendiente |
| Recetas | 🟡 Media | ⚪ Pendiente |
| Barcode Scanner | 🟡 Media | ⚪ Pendiente |

---

**Siguiente paso:** Resolver decisión de Backend/DB analizando modelo de datos en profundidad.
