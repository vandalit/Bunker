# Bitácora de Proyecto - Bunker

> **Fecha de inicio:** 2026-10-31  
> **Estado:** Fase 0 - Análisis y definición

---

## Sesión 01 - Concepto expandido y análisis técnico

### Visión del producto (tu prompt interpretado)

**Bunker** es una app mobile que gamifica la gestión de reservas alimenticias domésticas, aprovechando la cultura prep y el contexto de incertidumbre (económica, política, crisis). El objetivo es reducir ansiedad mediante la sensación de control y preparación, convirtiendo el acopio progresivo en un sistema de capas desbloqueables.

#### Mecánica central: Sistema de capas y dimensiones

**3 capas temporales (horizontes de autonomía):**
- Capa 1: **30 días** de autonomía
- Capa 2: **60 días** (se desbloquea al completar cierta cobertura en capa 1)
- Capa 3: **90 días** (objetivo final)

**4 dimensiones de calidad por capa** (se upgradan progresivamente):
1. **Hambre** → Cobertura calórica mínima para supervivencia
2. **Energética** → Balance energético saludable
3. **Nutricional** → Macros y micronutrientes esenciales cubiertos
4. **Variedad** → Diversidad de sabores, texturas, experiencia alimenticia

**Progresión:**
- Empiezas solo con acceso a la capa 1, dimensión "Hambre"
- A medida que cubres esa dimensión, desbloqueas la siguiente (Energética)
- Al completar todas las dimensiones de los 30 días, desbloqueas los 60 días
- El juego es ir "upgradeando" cada capa en todas sus dimensiones

**Items básicos de arranque:**
- Arroz, papa, atún en lata, legumbres secas
- Nuevos alimentos se desbloquean según progresas (más variedad, más complejos)

---

### Características clave

#### Stock dinámico y rotación
- Cada ítem tiene fecha de caducidad registrada
- Sistema de alertas para consumo antes de vencimiento
- Rotación sugerida automáticamente (qué cocinar hoy según lo que caduca pronto)
- **Experiencia por uso:** consumir alimentos correctamente (rotar, no desperdiciar) genera XP

#### Modelo de datos sofisticado
Cada alimento tiene **3 facetas**:
1. **Alimento** → propiedades nutricionales (calorías, proteínas, carbos, vitaminas, peso por porción)
2. **Producto** → presentación comercial (lata de atún 170g, saco de arroz 1kg, fecha de caducidad)
3. **Ingrediente** → unidad en recetas (no se puede fraccionar arbitrariamente — no usas "13g de atún", usas 1 lata o media lata según la receta)

#### Minutas / Recetas
- Sistema tipo "Papers, Please" → interfaz de aprobación/selección de recetas del día
- Las recetas consumen ingredientes completos o porciones lógicas
- Cubren necesidades del día/semana
- Escenarios extremos: "modo guerra" → arroz + papa básico sin variedad

#### Unidad familiar
- Empiezas solo
- Puedes agregar miembros del hogar (ajusta necesidades automáticamente)

#### Monetización satírica
- Versión de pago **no añade funciones** (lo mencionas como sátira)
- ¿Posible modelo: cosmetics, temas visuales estilo Fallout, badges especiales?

---

### Plataformas objetivo

- **Mobile:** Android (prioritario) + iOS
- **Showcase especial:** versión adaptada para **iPhone Duo** (foldable) — diferenciador UX en portafolio
- (¿Web responsive también? Pendiente de decidir si PWA o native puro)

---

### Contexto y objetivos

**Propósito emocional:**
- Reducir ansiedad por incertidumbre (crisis económica, polarización, guerra potencial)
- Gamificación como motivador para preparación real y práctica
- Cultura prep sin el estigma conspiranoico — enfoque práctico y accesible

**Objetivos comerciales/profesionales:**
- MVP funcional para testeo de bajo costo
- Potencial para hackathon
- Caso de estudio para portafolio UX/UI (especialmente diseño para foldables)
- Validación de salida comercial real en Latinoamérica

**Atractores complejos:**
- Incertidumbre económica (inflación, desempleo)
- Transición IA (reconversión laboral, ansiedad por futuro)
- Polarización política y tensiones sociales
- Necesidad de control y sensación de preparación

---

## Análisis técnico - Stack y arquitectura

### Opciones de stack para mobile

#### Opción 1: React Native + Expo
**Pros:**
- Un solo codebase para Android + iOS
- Expo simplifica desarrollo, testing y despliegue
- Hot reload, over-the-air updates
- Gran ecosistema de librerías
- Experiencia previa del equipo (asumido)
- Soporte decente para foldables con APIs de pantalla

**Contras:**
- Performance ligeramente inferior a native puro
- Limitaciones en APIs nativas muy específicas
- Bundle size puede crecer

**Evaluación:** ⭐⭐⭐⭐⭐ — recomendado para MVP con recursos limitados

---

#### Opción 2: Flutter
**Pros:**
- Performance cercana a native
- UI hermosa por defecto (Material/Cupertino)
- Hot reload excelente
- Codebase único para ambas plataformas
- Dart es lenguaje limpio y tipado

**Contras:**
- Curva de aprendizaje si no hay experiencia previa
- Ecosistema más chico que React Native
- Menos recursos/tutorials comparado con RN

**Evaluación:** ⭐⭐⭐⭐ — excelente opción si hay tiempo para aprender

---

#### Opción 3: Native separado (Kotlin + Swift)
**Pros:**
- Máxima performance y control
- Acceso completo a APIs de plataforma
- Mejor para features experimentales (iPhone Duo)

**Contras:**
- Doble esfuerzo de desarrollo
- Más costoso mantener dos codebases
- Mayor tiempo de desarrollo inicial

**Evaluación:** ⭐⭐ — no recomendado para MVP con recursos limitados

---

#### Opción 4: PWA (Progressive Web App)
**Pros:**
- Deploy instantáneo sin stores
- Codebase web reutilizable
- Fácil iteración y testing

**Contras:**
- Limitaciones en notificaciones push (especialmente iOS)
- Performance inferior a native
- Menos "app-like" feel
- Difícil para features offline complejas

**Evaluación:** ⭐⭐⭐ — viable como complemento pero no como plataforma principal

---

### Recomendación de stack inicial

**React Native + Expo** con TypeScript

**Justificación:**
- Balance óptimo entre velocidad de desarrollo y funcionalidad
- Experiencia de usuario sólida sin el overhead de native doble
- Ecosistema maduro para features necesarias (local storage, notificaciones, cámara para escanear códigos de barras futuro)
- Fácil testeo con Expo Go
- Deploy a stores simplificado con EAS

**Stack técnico completo propuesto:**

**Frontend/Mobile:**
- React Native + Expo
- TypeScript
- React Navigation (navegación)
- Zustand (estado global ligero)
- React Query (cache y sincronización)
- Async Storage (persistencia local)

**Backend/Database:**
- Opción A: **Supabase** (Postgres + Auth + Storage + Realtime)
- Opción B: **Firebase** (Firestore + Auth + Cloud Functions)
- Opción C: **SQLite local + sincronización opcional futura**

**Diseño:**
- Figma para diseño y prototipado
- Sistema de diseño basado en React Native Paper o NativeBase

---

### Arquitectura de datos - Modelo relacional propuesto

#### Entidades principales

**Users**
```
id: uuid (PK)
email: string
display_name: string
created_at: timestamp
household_size: integer (default 1)
xp_points: integer (default 0)
current_level: integer (default 1)
```

**FoodItems** (catálogo maestro de alimentos)
```
id: uuid (PK)
name: string
category: enum (grains, proteins, canned, dairy, etc.)
// Faceta ALIMENTO
calories_per_100g: float
protein_per_100g: float
carbs_per_100g: float
fat_per_100g: float
fiber_per_100g: float
vitamins: jsonb (flexible metadata)
minerals: jsonb
// Faceta INGREDIENTE
base_unit: enum (grams, ml, units, cans, bags)
min_recipe_portion: float (ej: 1 lata, 100g, 1 unidad)
```

**Products** (instancias comerciales de alimentos)
```
id: uuid (PK)
food_item_id: uuid (FK -> FoodItems)
user_id: uuid (FK -> Users)
brand: string (optional)
package_size: float (ej: 1000 para 1kg)
package_unit: enum (grams, ml, units)
purchase_date: date
expiry_date: date
quantity_available: integer (cuántos paquetes tengo)
cost_per_unit: float (precio por paquete)
location: string (optional, ej: "despensa arriba")
```

**SupplyLayers** (configuración de capas por usuario)
```
id: uuid (PK)
user_id: uuid (FK -> Users)
layer_number: integer (1=30días, 2=60días, 3=90días)
dimension: enum (hunger, energy, nutrition, variety)
target_days: integer (30, 60, 90)
is_unlocked: boolean
progress_percentage: float
```

**Recipes** (recetas para consumo)
```
id: uuid (PK)
name: string
difficulty: enum (war_mode, basic, normal, comfort)
servings: integer
prep_time_minutes: integer
instructions: text
nutritional_summary: jsonb
```

**RecipeIngredients**
```
id: uuid (PK)
recipe_id: uuid (FK -> Recipes)
food_item_id: uuid (FK -> FoodItems)
quantity: float
unit: enum (must match food_item.base_unit)
```

**StockMovements** (log de todo movimiento)
```
id: uuid (PK)
user_id: uuid (FK -> Users)
product_id: uuid (FK -> Products)
movement_type: enum (purchase, consumption, waste, adjustment)
quantity: integer
date: timestamp
recipe_id: uuid (FK -> Recipes, nullable)
xp_earned: integer (default 0)
notes: text (optional)
```

**Alerts**
```
id: uuid (PK)
user_id: uuid (FK -> Users)
product_id: uuid (FK -> Products)
alert_type: enum (expiring_soon, expired, low_stock, dimension_incomplete)
severity: enum (info, warning, critical)
message: text
created_at: timestamp
dismissed_at: timestamp (nullable)
```

---

### Sistema de dimensiones - Cálculo propuesto

#### Dimensión 1: Hambre
**Métrica:** Calorías totales disponibles vs. necesarias
```
Necesidad diaria base: 2000 kcal/persona
Target 30 días: 60,000 kcal
Disponible: Σ(productos.quantity * food_item.calories_per_100g * product.package_size / 100)
Progress: (Disponible / Target) * 100
```

#### Dimensión 2: Energética
**Métrica:** Balance macro (carbos, proteínas, grasas) + densidad calórica sostenible
```
Target ratios:
- Carbohidratos: 50-60% calorías
- Proteínas: 15-25%
- Grasas: 20-30%
Desviación aceptable: ±10%
Progress basado en cuán cerca están los ratios + cobertura calórica
```

#### Dimensión 3: Nutricional
**Métrica:** Micronutrientes esenciales cubiertos (vitaminas A, C, D, B12, hierro, calcio, etc.)
```
Checklist de 10-15 nutrientes clave
Progress: % de nutrientes con cobertura >70% del RDA (Recommended Daily Allowance)
```

#### Dimensión 4: Variedad
**Métrica:** Diversidad de categorías y rotación
```
Categorías únicas en stock: grains, proteins, vegetables, fruits, dairy, fats
Progress: (categorías presentes / categorías totales) * peso + factor de rotación
```

---

## Preguntas abiertas y decisiones pendientes

### 1. Backend: ¿Supabase, Firebase o local-first?

**Supabase:**
- ✅ Postgres relacional (perfecto para modelo complejo)
- ✅ Auth incluido
- ✅ Realtime subscriptions
- ✅ Políticas RLS (Row Level Security) nativas
- ⚠️ Requiere conexión para sincronizar

**Firebase:**
- ✅ Ecosistema maduro y robusto
- ✅ Auth excelente
- ✅ Offline-first con Firestore
- ⚠️ NoSQL puede complicar queries complejas (joins, agregaciones)
- ⚠️ Costos pueden escalar rápido

**SQLite local + sync futuro:**
- ✅ Offline-first total desde día 1
- ✅ Rendimiento máximo
- ✅ No dependencia de terceros para MVP
- ⚠️ Sincronización multi-dispositivo compleja
- ⚠️ Backup/restore responsabilidad del usuario

**¿Tu preferencia?**

---

### 2. Gamificación: ¿XP y niveles o achievements?

Opción A: Sistema de XP clásico
- Ganas XP por acciones (registrar compra, consumir correctamente, rotar stock)
- Subes de nivel
- Cada nivel desbloquea algo (¿items, badges, temas visuales?)

Opción B: Sistema de logros/achievements
- Badges por milestones (primera capa completa, 0 desperdicios en un mes, etc.)
- Más narrativo y menos numérico

Opción C: Híbrido (XP + badges)

**¿Cuál prefieres?**

---

### 3. iPhone Duo: ¿Features específicas o solo diseño adaptado?

- ¿Quieres features únicas que aprovechen el dual screen (ej: inventario en una pantalla, receta en otra)?
- ¿O solo diseño responsivo premium que se vea espectacular en foldable?

---

### 4. Monetización: ¿Realmente 100% gratis?

Mencionaste versión de pago que no añade funciones (sátira). Opciones reales:
- A: Gratis total, sin ads, sin pago → posicionar como público bien
- B: Cosmetics de pago (temas Fallout, badges premium, sonidos)
- C: Donación opcional tipo "Buy me a coffee"
- D: Modelo freemium futuro (multi-hogar, sincronización cloud)

**¿Cuál es tu visión real?**

---

### 5. Recetas: ¿Base de datos inicial o user-generated?

- Opción A: Seedeamos 20-30 recetas básicas hardcodeadas
- Opción B: Sistema para que usuarios creen sus propias recetas
- Opción C: Ambas (seed inicial + creación custom)

---

### 6. Escaneo de códigos de barras: ¿MVP o V2?

Feature útil para agregar productos rápido. ¿Lo priorizamos en MVP o lo dejamos para segunda iteración?

---

## Próximos pasos propuestos

1. **Decidir backend** (Supabase vs Firebase vs SQLite local)
2. **Definir stack final confirmado**
3. **Crear modelo de datos detallado** (schema SQL/NoSQL según elección)
4. **Wireframes básicos** de pantallas clave:
   - Onboarding
   - Dashboard (progreso de capas)
   - Inventario
   - Agregar producto
   - Minutas/recetas del día
   - Perfil y XP
5. **Setup del proyecto**: inicializar repo, estructura de carpetas, dependencias
6. **Primera feature vertical**: Autenticación + registro de un producto + visualización básica

---

**Esperando tus respuestas a las preguntas abiertas para continuar.**
