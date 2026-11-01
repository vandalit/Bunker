# Análisis del Modelo de Datos - Bunker

> Documento técnico para decidir entre arquitectura relacional (Supabase) vs NoSQL (Firebase)

---

## El problema: Complejidad de los alimentos

Un **alimento** en Bunker tiene múltiples facetas que no son simples atributos planos:

### Faceta 1: INGREDIENTE (culinario/gastronómico)
- Sabor (dulce, salado, umami, ácido, amargo)
- Textura (crujiente, cremoso, fibroso)
- Grupo alimenticio (granos, proteínas, vegetales, frutas, lácteos, grasas)
- Rol en receta (base, proteína, condimento, guarnición)
- Restricciones (apto vegano, kosher, sin gluten, sin lactosa)

### Faceta 2: PRODUCTO (comercial/logístico)
- Marca
- Presentación (lata, bolsa, frasco, kg suelto)
- Tamaño de empaque (500g, 1kg, 12 unidades)
- Precio por unidad
- Código de barras
- Fecha de compra
- Fecha de caducidad
- Ubicación en bodega

### Faceta 3: NUTRICIONAL (salud/dieta)
- Calorías por 100g
- Macronutrientes:
  - Proteínas (g)
  - Carbohidratos (g)
    - de los cuales azúcares (g)
  - Grasas (g)
    - saturadas, insaturadas, trans
  - Fibra (g)
- Micronutrientes:
  - Vitaminas (A, C, D, E, K, B-complex)
  - Minerales (hierro, calcio, magnesio, zinc, sodio)
- Índice glucémico
- Perfil para dietas:
  - Keto-friendly
  - Low-carb
  - High-protein
  - Diabetic-safe
  - Hipertensión (bajo sodio)

### Faceta 4: ENERGÉTICA (rendimiento/sostenibilidad)
- Densidad calórica (kcal/100g)
- Índice de saciedad
- Velocidad de digestión (rápida/lenta)
- Energía neta post-digestión
- Sostenibilidad de almacenamiento (shelf-stable vs fresco)

---

## Las relaciones complejas

### 1. Alimento → Producto (1:N)
Un mismo alimento (ej: "Arroz blanco") puede tener múltiples productos:
- Arroz Miraflores 1kg bolsa
- Arroz Tucapel 5kg saco
- Arroz a granel mercado

Cada uno con:
- Precio diferente
- Caducidad diferente
- Ubicación diferente

### 2. Alimento ↔ Receta (N:M)
Una receta usa múltiples alimentos, un alimento aparece en múltiples recetas.

**Restricción clave:** No se pueden fraccionar arbitrariamente.
- ✅ Usar 1 lata de atún (170g)
- ✅ Usar 200g de arroz
- ❌ Usar 13.7g de atún (no tiene sentido práctico)

Cada relación receta-ingrediente tiene:
- Cantidad (con unidad lógica)
- Opcional/sustituible
- Notas de preparación

### 3. Producto → Movimientos (1:N)
Cada producto tiene un historial:
- Compra inicial
- Consumos parciales
- Mermas/desperdicios
- Ajustes de inventario

### 4. Usuario ↔ Alimentos favoritos (N:M con metadata)
- Alimentos marcados como "no me gusta" (excluir de recetas)
- Alimentos favoritos (priorizar en sugerencias)
- Alergias/intolerancias (hard exclude)

### 5. Dimensiones ↔ Alimentos (lógica compleja)
Cada dimensión (Hambre, Energética, Nutricional, Variedad) evalúa el inventario con criterios diferentes:

**Dimensión Hambre:**
- Query: Σ calorías disponibles ≥ target
- Solo considera: `alimentos.calorias * productos.cantidad`

**Dimensión Energética:**
- Query compleja: balance macro (carbos/proteínas/grasas) dentro de rangos
- Considera: ratios, distribución, densidad

**Dimensión Nutricional:**
- Query multi-atributo: cobertura de N micronutrientes
- Considera: vitaminas, minerales, RDA (Recommended Daily Allowance)

**Dimensión Variedad:**
- Query de diversidad: COUNT(DISTINCT categorías), rotación temporal
- Considera: categorías únicas, frecuencia de consumo

---

## Comparativa: Relacional vs NoSQL

### Escenario A: Modelo Relacional (Supabase / PostgreSQL)

#### Schema propuesto

```sql
-- Catálogo maestro
CREATE TABLE food_items (
  id UUID PRIMARY KEY,
  name TEXT NOT NULL,
  category TEXT, -- grains, proteins, vegetables, etc.
  
  -- Nutricional
  calories_per_100g FLOAT,
  protein_per_100g FLOAT,
  carbs_per_100g FLOAT,
  sugar_per_100g FLOAT,
  fat_per_100g FLOAT,
  saturated_fat_per_100g FLOAT,
  fiber_per_100g FLOAT,
  sodium_per_100mg FLOAT,
  
  -- Micronutrientes (normalized)
  vitamins JSONB, -- {A: 500, C: 20, D: 0, ...}
  minerals JSONB, -- {iron: 2.5, calcium: 100, ...}
  
  -- Ingrediente
  base_unit TEXT, -- 'grams', 'ml', 'units'
  min_recipe_portion FLOAT, -- porción mínima lógica
  flavor_profile JSONB, -- ['salty', 'umami']
  texture TEXT,
  
  -- Energética
  caloric_density TEXT, -- 'low', 'medium', 'high'
  satiety_index FLOAT,
  
  -- Dietas
  diet_tags JSONB, -- ['keto', 'vegan', 'gluten-free']
  
  -- Metadata
  created_at TIMESTAMP DEFAULT NOW()
);

-- Productos (instancias comerciales)
CREATE TABLE products (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  food_item_id UUID REFERENCES food_items(id),
  
  brand TEXT,
  package_size FLOAT,
  package_unit TEXT,
  barcode TEXT,
  
  purchase_date DATE,
  expiry_date DATE,
  quantity_available INTEGER DEFAULT 1,
  cost_per_unit FLOAT,
  
  location TEXT, -- 'despensa', 'refrigerador'
  
  created_at TIMESTAMP DEFAULT NOW()
);

-- Recetas
CREATE TABLE recipes (
  id UUID PRIMARY KEY,
  name TEXT NOT NULL,
  difficulty TEXT, -- 'war_mode', 'basic', 'comfort'
  servings INTEGER,
  prep_time_minutes INTEGER,
  instructions TEXT,
  
  -- Nutricional calculado
  total_calories FLOAT,
  protein_total FLOAT,
  carbs_total FLOAT,
  
  created_at TIMESTAMP DEFAULT NOW()
);

-- Ingredientes de recetas (relación N:M)
CREATE TABLE recipe_ingredients (
  id UUID PRIMARY KEY,
  recipe_id UUID REFERENCES recipes(id) ON DELETE CASCADE,
  food_item_id UUID REFERENCES food_items(id),
  
  quantity FLOAT,
  unit TEXT,
  optional BOOLEAN DEFAULT FALSE,
  notes TEXT
);

-- Movimientos de stock
CREATE TABLE stock_movements (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  product_id UUID REFERENCES products(id),
  
  movement_type TEXT, -- 'purchase', 'consumption', 'waste'
  quantity INTEGER,
  recipe_id UUID REFERENCES recipes(id), -- nullable
  
  xp_earned INTEGER DEFAULT 0,
  notes TEXT,
  
  created_at TIMESTAMP DEFAULT NOW()
);

-- Progreso de capas/dimensiones
CREATE TABLE layer_progress (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  
  layer_number INTEGER, -- 1, 2, 3
  dimension TEXT, -- 'hunger', 'energy', 'nutrition', 'variety'
  target_days INTEGER, -- 30, 60, 90
  
  is_unlocked BOOLEAN DEFAULT FALSE,
  progress_percentage FLOAT DEFAULT 0,
  
  last_calculated_at TIMESTAMP,
  
  UNIQUE(user_id, layer_number, dimension)
);
```

#### Ventajas relacional

✅ **Queries complejas naturales**
```sql
-- Calcular dimensión nutricional (cobertura de micronutrientes)
SELECT 
  COUNT(DISTINCT nutrient) as nutrients_covered
FROM (
  SELECT jsonb_object_keys(vitamins) as nutrient
  FROM food_items fi
  JOIN products p ON p.food_item_id = fi.id
  WHERE p.user_id = :user_id 
    AND p.quantity_available > 0
    AND jsonb_extract_path(vitamins, nutrient)::float > 0.7 * rda_value
) nutrients;
```

✅ **Joins eficientes para reporting**
```sql
-- Productos por expirar con info nutricional completa
SELECT 
  p.*, 
  fi.name, 
  fi.calories_per_100g,
  fi.category
FROM products p
JOIN food_items fi ON p.food_item_id = fi.id
WHERE p.expiry_date BETWEEN NOW() AND NOW() + INTERVAL '7 days'
ORDER BY p.expiry_date ASC;
```

✅ **Integridad referencial**
- No puedes borrar un food_item si tiene productos asociados
- Cascadas automáticas bien definidas

✅ **Aggregaciones complejas**
```sql
-- Calcular balance macro del inventario
SELECT 
  SUM(fi.protein_per_100g * p.package_size * p.quantity_available / 100) as total_protein,
  SUM(fi.carbs_per_100g * p.package_size * p.quantity_available / 100) as total_carbs,
  SUM(fi.fat_per_100g * p.package_size * p.quantity_available / 100) as total_fat
FROM products p
JOIN food_items fi ON p.food_item_id = fi.id
WHERE p.user_id = :user_id;
```

✅ **JSONB para flexibilidad híbrida**
- Micronutrientes en JSONB (no todos los alimentos tienen todos)
- Indexable y queryable con GIN indexes
- Best of both worlds

#### Desventajas relacional

⚠️ **Queries pueden ser pesadas**
- Joins múltiples en móvil pueden ser lentos si mal optimizados
- Requiere pensar en índices desde el inicio

⚠️ **Migraciones de schema más rígidas**
- Agregar columna = ALTER TABLE
- Más ceremonia que NoSQL

⚠️ **Requiere conexión para queries complejas**
- Si queremos offline-first puro, necesitamos SQLite local + sync

---

### Escenario B: Modelo NoSQL (Firebase Firestore)

#### Schema propuesto

```javascript
// Colección: foodItems
{
  id: "uuid",
  name: "Arroz blanco",
  category: "grains",
  
  // Nutricional
  nutritional: {
    per100g: {
      calories: 130,
      protein: 2.7,
      carbs: 28,
      sugar: 0,
      fat: 0.3,
      fiber: 0.4,
      sodium: 1
    },
    vitamins: {
      A: 0, C: 0, D: 0, B12: 0, ...
    },
    minerals: {
      iron: 0.2, calcium: 10, ...
    }
  },
  
  // Ingrediente
  ingredient: {
    baseUnit: "grams",
    minRecipePortion: 100,
    flavorProfile: ["neutral", "starchy"],
    texture: "soft"
  },
  
  // Energética
  energetic: {
    caloricDensity: "medium",
    satietyIndex: 75,
    digestionSpeed: "medium"
  },
  
  // Dietas
  dietTags: ["vegan", "gluten-free", "keto-unfriendly"],
  
  createdAt: timestamp
}

// Colección: users/{userId}/products
{
  id: "uuid",
  foodItemId: "ref_to_foodItems",
  foodItemSnapshot: { name: "Arroz blanco", category: "grains" }, // denormalized
  
  brand: "Miraflores",
  packageSize: 1000,
  packageUnit: "grams",
  barcode: "1234567890",
  
  purchaseDate: "2026-10-01",
  expiryDate: "2027-10-01",
  quantityAvailable: 2,
  costPerUnit: 1200,
  location: "despensa",
  
  createdAt: timestamp
}

// Colección: recipes
{
  id: "uuid",
  name: "Arroz con atún",
  difficulty: "basic",
  servings: 4,
  prepTimeMinutes: 20,
  instructions: "...",
  
  ingredients: [
    {
      foodItemId: "uuid",
      foodItemName: "Arroz blanco", // denormalized
      quantity: 400,
      unit: "grams",
      optional: false
    },
    {
      foodItemId: "uuid",
      foodItemName: "Atún en lata",
      quantity: 1,
      unit: "can",
      optional: false
    }
  ],
  
  nutritionalSummary: {
    totalCalories: 600,
    proteinTotal: 30,
    carbsTotal: 80
  },
  
  createdAt: timestamp
}

// Colección: users/{userId}/stockMovements
{
  id: "uuid",
  productId: "uuid",
  productSnapshot: { name: "Arroz Miraflores", ... }, // denormalized
  
  movementType: "consumption",
  quantity: 1,
  recipeId: "uuid", // optional
  
  xpEarned: 10,
  notes: "",
  
  createdAt: timestamp
}

// Colección: users/{userId}/layerProgress
{
  id: "layer_1_hunger",
  layerNumber: 1,
  dimension: "hunger",
  targetDays: 30,
  
  isUnlocked: true,
  progressPercentage: 67.5,
  
  lastCalculatedAt: timestamp
}
```

#### Ventajas NoSQL

✅ **Offline-first nativo**
- Firestore caché local automático
- Sincronización transparente
- Perfecto para móvil

✅ **Queries simples muy rápidas**
```javascript
// Obtener productos por expirar
db.collection('users').doc(userId).collection('products')
  .where('expiryDate', '>=', today)
  .where('expiryDate', '<=', weekFromNow)
  .orderBy('expiryDate')
  .get()
```

✅ **Schema flexible**
- Agregar campos sin migración
- Cada documento puede tener estructura ligeramente diferente

✅ **Realtime fácil**
```javascript
// Escuchar cambios en inventario
db.collection('users').doc(userId).collection('products')
  .onSnapshot(snapshot => {
    // UI se actualiza automáticamente
  })
```

✅ **Denormalización estratégica**
- Guardar nombre del alimento en el producto → evita joins
- Snapshot de datos en movimientos → historial inmutable

#### Desventajas NoSQL

❌ **Queries complejas son difíciles o imposibles**

**Problema 1:** Calcular balance macro del inventario completo
```javascript
// En SQL: un JOIN + SUM
// En Firestore: 
// 1. Leer TODOS los productos del usuario (puede ser cientos)
// 2. Para cada producto, leer el foodItem asociado
// 3. Calcular en cliente
// = N+1 queries o lectura masiva
```

**Problema 2:** Cobertura de micronutrientes (dimensión nutricional)
```javascript
// En SQL: query con jsonb_object_keys + aggregation
// En Firestore:
// 1. Leer todos los productos
// 2. Leer todos los foodItems
// 3. Extraer vitaminas/minerales de cada uno
// 4. Calcular cobertura en cliente
// = Operación costosa en cliente móvil
```

**Problema 3:** Queries con múltiples filtros
```javascript
// Imposible:
db.collection('foodItems')
  .where('category', '==', 'proteins')
  .where('nutritional.per100g.protein', '>', 15)
  .where('dietTags', 'array-contains', 'keto')
  .get()
// Firestore solo permite 1 inequality o 1 array-contains por query
```

❌ **Joins manuales o denormalización**
- Guardar datos redundantes (nombre en producto + en foodItem)
- Mantener consistencia es responsabilidad del código
- Bugs potenciales si olvidas actualizar snapshots

❌ **Aggregaciones complejas → Cloud Functions**
- Calcular dimensiones requiere lógica en backend
- Más latencia, más costo

❌ **Costos por lectura**
- Firestore cobra por documento leído
- Queries que leen 100+ documentos = costoso en escala

---

## Análisis de casos de uso críticos

### Caso 1: Calcular progreso de "Dimensión Nutricional"

**Requiere:**
1. Listar todos los productos del usuario con cantidad > 0
2. Join con foodItems para obtener vitaminas/minerales
3. Sumar cobertura de cada nutriente
4. Comparar contra RDA (Recommended Daily Allowance)
5. Calcular % de nutrientes cubiertos

**En Relacional (Supabase):**
```sql
-- Una query, server-side, <100ms
WITH user_nutrients AS (
  SELECT 
    jsonb_object_keys(fi.vitamins) as nutrient,
    SUM((fi.vitamins->>jsonb_object_keys(fi.vitamins))::float * 
        p.package_size * p.quantity_available / 100) as total_amount
  FROM products p
  JOIN food_items fi ON p.food_item_id = fi.id
  WHERE p.user_id = :user_id AND p.quantity_available > 0
  GROUP BY nutrient
)
SELECT 
  COUNT(*) FILTER (WHERE total_amount >= rda_threshold) * 100.0 / COUNT(*) as coverage_pct
FROM user_nutrients
JOIN nutrient_rda ON user_nutrients.nutrient = nutrient_rda.name;
```

**En NoSQL (Firebase):**
```javascript
// Múltiples queries, client-side
// 1. Leer productos (1 query, N documentos)
const products = await db.collection(`users/${userId}/products`)
  .where('quantityAvailable', '>', 0)
  .get();

// 2. Leer foodItems para cada producto (N queries o batch)
const foodItemIds = products.docs.map(p => p.data().foodItemId);
const foodItems = await Promise.all(
  foodItemIds.map(id => db.collection('foodItems').doc(id).get())
);

// 3. Calcular en memoria (JavaScript en móvil)
const nutrients = {};
products.forEach((prodDoc, i) => {
  const prod = prodDoc.data();
  const food = foodItems[i].data();
  Object.entries(food.nutritional.vitamins).forEach(([nut, val]) => {
    nutrients[nut] = (nutrients[nut] || 0) + 
      (val * prod.packageSize * prod.quantityAvailable / 100);
  });
});

const covered = Object.values(nutrients).filter(v => v >= rdaThreshold).length;
const coveragePct = (covered / Object.keys(nutrients).length) * 100;
```

**Comparación:**
- **Relacional:** 1 query server-side, ~50ms, sin procesamiento en cliente
- **NoSQL:** 1 + N queries, procesamiento pesado en cliente móvil, ~500ms+, consume batería

**Ganador:** Relacional

---

### Caso 2: Listar productos por expirar (próximos 7 días)

**En Relacional:**
```sql
SELECT p.*, fi.name, fi.category
FROM products p
JOIN food_items fi ON p.food_item_id = fi.id
WHERE p.user_id = :user_id
  AND p.expiry_date BETWEEN NOW() AND NOW() + INTERVAL '7 days'
ORDER BY p.expiry_date;
```

**En NoSQL:**
```javascript
// Query simple
const expiring = await db.collection(`users/${userId}/products`)
  .where('expiryDate', '>=', today)
  .where('expiryDate', '<=', weekFromNow)
  .orderBy('expiryDate')
  .get();

// Si denormalizamos nombre en producto, no need de más queries
```

**Comparación:**
- **Relacional:** 1 query con JOIN
- **NoSQL:** 1 query (con denormalización) o 1+N queries (sin denormalizar)

**Ganador:** Empate (si NoSQL denormaliza correctamente)

---

### Caso 3: Guardar un nuevo producto

**En Relacional:**
```sql
INSERT INTO products (user_id, food_item_id, brand, package_size, ...)
VALUES (:user_id, :food_item_id, :brand, :package_size, ...);
```

**En NoSQL:**
```javascript
await db.collection(`users/${userId}/products`).add({
  foodItemId: foodItemId,
  foodItemSnapshot: { name: "...", category: "..." }, // denormalizar
  brand, packageSize, ...
});
```

**Comparación:**
- Ambos simples y rápidos

**Ganador:** Empate

---

### Caso 4: Recalcular todas las dimensiones (background job)

**Requiere:**
- Leer inventario completo
- Calcular 4 dimensiones × 3 capas = 12 valores
- Actualizar progreso

**En Relacional:**
- Stored procedures o backend con queries SQL eficientes
- Puede correr server-side sin overhead de red

**En NoSQL:**
- Cloud Function que lee todo y calcula
- Overhead de red para leer documentos
- O calcular client-side (pesado en móvil)

**Ganador:** Relacional

---

## Híbrido: SQLite local + Firebase Auth/Storage

### Propuesta

- **SQLite local:** toda la lógica de datos, queries complejas client-side
- **Firebase Auth:** autenticación Google
- **Firebase Storage:** backup de BD, sincronización entre dispositivos (export/import manual o automático)

#### Ventajas híbrido

✅ **Queries complejas sin compromiso** (SQLite es SQL completo)
✅ **Offline-first total** (sin depender de conexión)
✅ **Performance máximo** (local, sin latencia de red)
✅ **Sin costos por query** (todo local)
✅ **Aprovechas Firebase** que ya tienes (solo para Auth/Storage)

#### Desventajas híbrido

⚠️ **Sincronización manual** (no hay realtime automático entre dispositivos)
⚠️ **Backup/restore** es responsabilidad del código
⚠️ **Más complejo** que usar Firebase directamente

---

## Recomendación

### Opción 1: **Supabase (PostgreSQL)** — RECOMENDADO

**Por qué:**
- El modelo de datos ES inherentemente relacional
- Las queries críticas (dimensiones, balance macro, cobertura nutricional) son naturales en SQL
- JSONB permite flexibilidad donde se necesita (micronutrientes, tags)
- RLS (Row Level Security) protege datos por usuario automáticamente
- Auth incluida (puede usar Google OAuth)
- Realtime subscriptions para UI reactiva
- Free tier generoso para MVP

**Trade-off:**
- Requiere conexión para queries server-side
- Offline: podemos usar caché Apollo/React Query + queue de mutaciones

**Stack resultante:**
- Backend: Supabase (Postgres + Auth + Storage + Realtime)
- Frontend: React Native + Zustand (estado local) + React Query (caché + sync)
- Offline: React Query persistent cache + mutation queue

---

### Opción 2: **SQLite local + Firebase Auth/Storage**

**Por qué:**
- Offline-first total
- Queries complejas sin compromiso
- Aprovechas Firebase que ya tienes (solo para Auth)
- Cero costos por query

**Trade-off:**
- Sincronización entre dispositivos manual
- Más código de infraestructura

**Stack resultante:**
- Backend: Firebase Auth + Storage (solo backup)
- DB: expo-sqlite (SQLite)
- Frontend: React Native + Zustand
- Sync: export/import JSON a Firebase Storage

---

### Opción 3: **Firebase Firestore puro** — NO RECOMENDADO

**Por qué NO:**
- Las queries complejas son dolorosas o imposibles
- Calcular dimensiones requiere leer muchos documentos + procesar en cliente
- Costos escalan mal (lecturas por documento)
- Denormalización introduce bugs potenciales

**Solo viable si:**
- Simplificamos radicalmente el modelo (quitar dimensiones nutricionales complejas)
- Aceptamos calcular todo client-side con performance limitado

---

## Pregunta decisiva

**¿Qué tan crítico es el offline-first total (sin conexión nunca) vs. queries complejas eficientes?**

- **Si offline total es crítico:** SQLite local + Firebase Auth/Storage
- **Si podemos asumir conexión ocasional:** Supabase (mejor DX, menos código de infraestructura)

**Mi recomendación personal:** Supabase. El modelo de datos grita "relacional", y Supabase hace todo más simple que manejar sincronización manual con SQLite.

---

## Siguiente paso

Define tu prioridad:
1. **Offline-first absoluto** → SQLite + Firebase
2. **Queries eficientes + menos código** → Supabase

Con eso decidido, armamos el schema definitivo y empezamos el setup.
