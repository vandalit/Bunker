# Arquitectura Offline-First - Bunker

> Decisión: SQLite local + Firebase Auth/Storage

---

## Principios fundamentales

### 1. Offline-first por diseño
- La app funciona **sin conexión a internet en todo momento**
- Solo se requiere conexión para:
  - Autenticación inicial con Google (una sola vez)
  - Backup opcional de datos
  - Sincronización entre dispositivos (opcional)

### 2. Coherencia narrativa
- **Cultura prep:** Si se viene el colapso, tu app de preparación no puede depender de internet
- **Escenarios de crisis:** Cortes de luz, internet, infraestructura caída
- **Autonomía real:** Tus datos viven en tu dispositivo, siempre accesibles

---

## Stack técnico

### Base de datos: SQLite (expo-sqlite)

**Por qué SQLite:**
- ✅ SQL completo — queries complejas nativas
- ✅ 100% local — funciona sin conexión
- ✅ Performance óptimo — sin latencia de red
- ✅ Transaccional — ACID compliance
- ✅ Maduro y battle-tested — usado en millones de apps
- ✅ Gratis y sin límites de queries

**Librería:**
```bash
npx expo install expo-sqlite
```

**Alternativa considerada:** Drizzle ORM sobre expo-sqlite
- Type-safety con TypeScript
- Migrations versionadas
- Query builder type-safe
- Opcional, pero recomendado para DX

---

### Autenticación: Firebase Auth

**Por qué Firebase Auth:**
- ✅ Google Sign-In con una línea
- ✅ Ya tienes cuenta activa
- ✅ Tokens JWT para identificar usuario
- ✅ No requiere backend custom

**Flujo:**
1. Usuario abre app por primera vez
2. Requiere conexión para login con Google (una sola vez)
3. Token se guarda localmente
4. Todas las operaciones posteriores offline
5. Token se renueva automáticamente cuando hay conexión (background)

**Importante:** El `userId` de Firebase se usa como clave en SQLite para aislar datos por usuario en el mismo dispositivo.

---

### Backup y Sync: Firebase Storage

**Por qué Firebase Storage:**
- ✅ Backup encriptado de la BD completa
- ✅ Sincronización entre dispositivos (export/import)
- ✅ Control total del usuario (manual o automático)
- ✅ Barato (solo subida/bajada de JSON, no queries)

**Flujo de backup:**
1. Usuario activa backup (manual o automático cada N días)
2. App exporta SQLite a JSON
3. JSON se encripta localmente
4. Se sube a Firebase Storage (`users/{userId}/backup.json.enc`)
5. Si hay conexión: backup automático silencioso

**Flujo de restore/sync:**
1. Usuario instala app en nuevo dispositivo
2. Login con Google
3. App detecta backup en Firebase Storage
4. Descarga y desencripta JSON
5. Importa a SQLite local
6. Ya tiene todos sus datos en nuevo dispositivo

**Ventaja vs Firestore:**
- Firestore cobra por documento leído/escrito (costos escalan)
- Storage cobra por GB almacenado (centavos por usuario)
- Un usuario típico: ~1-5 MB de datos → costo anual <$0.01

---

## Schema SQLite

### Estructura de tablas (SQL definitivo)

```sql
-- Configuración inicial
PRAGMA foreign_keys = ON;
PRAGMA journal_mode = WAL; -- Write-Ahead Logging para concurrencia

-- Usuario local (sincronizado con Firebase Auth)
CREATE TABLE IF NOT EXISTS users (
  id TEXT PRIMARY KEY, -- Firebase UID
  email TEXT NOT NULL,
  display_name TEXT,
  household_size INTEGER DEFAULT 1,
  xp_points INTEGER DEFAULT 0,
  current_level INTEGER DEFAULT 1,
  created_at INTEGER NOT NULL, -- Unix timestamp
  last_synced_at INTEGER -- Unix timestamp
);

-- Catálogo maestro de alimentos
CREATE TABLE IF NOT EXISTS food_items (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  category TEXT NOT NULL, -- 'grains', 'proteins', 'vegetables', 'fruits', 'dairy', 'fats', 'canned', 'legumes'
  
  -- Nutricional por 100g
  calories_per_100g REAL,
  protein_per_100g REAL,
  carbs_per_100g REAL,
  sugar_per_100g REAL,
  fat_per_100g REAL,
  saturated_fat_per_100g REAL,
  fiber_per_100g REAL,
  sodium_per_100mg REAL,
  
  -- Micronutrientes (JSON serializado)
  vitamins TEXT, -- {"A": 500, "C": 20, "D": 0, ...}
  minerals TEXT, -- {"iron": 2.5, "calcium": 100, ...}
  
  -- Ingrediente
  base_unit TEXT NOT NULL, -- 'grams', 'ml', 'units', 'cans'
  min_recipe_portion REAL, -- porción mínima lógica (ej: 170 para 1 lata)
  flavor_profile TEXT, -- JSON array: ["salty", "umami"]
  texture TEXT, -- 'soft', 'crunchy', 'creamy', 'chewy'
  
  -- Energética
  caloric_density TEXT, -- 'low', 'medium', 'high'
  satiety_index REAL, -- 0-100
  
  -- Dietas (JSON array)
  diet_tags TEXT, -- ["keto", "vegan", "gluten-free"]
  
  -- Metadata
  is_unlocked INTEGER DEFAULT 1, -- gamificación: items se desbloquean
  unlock_requirement TEXT, -- "layer_1_hunger_complete"
  created_at INTEGER NOT NULL
);

-- Índices para búsqueda eficiente
CREATE INDEX IF NOT EXISTS idx_food_items_category ON food_items(category);
CREATE INDEX IF NOT EXISTS idx_food_items_unlocked ON food_items(is_unlocked);

-- Productos (instancias comerciales de alimentos)
CREATE TABLE IF NOT EXISTS products (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  food_item_id TEXT NOT NULL,
  
  brand TEXT,
  package_size REAL NOT NULL, -- ej: 1000 para 1kg
  package_unit TEXT NOT NULL, -- debe coincidir con food_item.base_unit
  barcode TEXT,
  
  purchase_date INTEGER, -- Unix timestamp
  expiry_date INTEGER, -- Unix timestamp
  quantity_available INTEGER DEFAULT 1, -- cuántos paquetes
  cost_per_unit REAL, -- precio por paquete
  
  location TEXT, -- 'despensa', 'refrigerador', 'congelador'
  
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (food_item_id) REFERENCES food_items(id) ON DELETE RESTRICT
);

CREATE INDEX IF NOT EXISTS idx_products_user ON products(user_id);
CREATE INDEX IF NOT EXISTS idx_products_food_item ON products(food_item_id);
CREATE INDEX IF NOT EXISTS idx_products_expiry ON products(expiry_date);

-- Recetas
CREATE TABLE IF NOT EXISTS recipes (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  difficulty TEXT NOT NULL, -- 'war_mode', 'basic', 'normal', 'comfort'
  servings INTEGER DEFAULT 1,
  prep_time_minutes INTEGER,
  instructions TEXT,
  
  -- Nutricional calculado (desnormalizado para performance)
  total_calories REAL,
  total_protein REAL,
  total_carbs REAL,
  total_fat REAL,
  
  -- Metadata
  is_unlocked INTEGER DEFAULT 1,
  unlock_requirement TEXT,
  created_at INTEGER NOT NULL
);

-- Ingredientes de recetas (relación N:M)
CREATE TABLE IF NOT EXISTS recipe_ingredients (
  id TEXT PRIMARY KEY,
  recipe_id TEXT NOT NULL,
  food_item_id TEXT NOT NULL,
  
  quantity REAL NOT NULL,
  unit TEXT NOT NULL,
  optional INTEGER DEFAULT 0, -- boolean
  notes TEXT,
  
  FOREIGN KEY (recipe_id) REFERENCES recipes(id) ON DELETE CASCADE,
  FOREIGN KEY (food_item_id) REFERENCES food_items(id) ON DELETE RESTRICT
);

CREATE INDEX IF NOT EXISTS idx_recipe_ingredients_recipe ON recipe_ingredients(recipe_id);

-- Movimientos de stock (log inmutable)
CREATE TABLE IF NOT EXISTS stock_movements (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  product_id TEXT NOT NULL,
  
  movement_type TEXT NOT NULL, -- 'purchase', 'consumption', 'waste', 'adjustment'
  quantity INTEGER NOT NULL, -- positivo o negativo
  recipe_id TEXT, -- nullable, solo si es consumption por receta
  
  xp_earned INTEGER DEFAULT 0,
  notes TEXT,
  
  created_at INTEGER NOT NULL,
  
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
  FOREIGN KEY (recipe_id) REFERENCES recipes(id) ON DELETE SET NULL
);

CREATE INDEX IF NOT EXISTS idx_stock_movements_user ON stock_movements(user_id);
CREATE INDEX IF NOT EXISTS idx_stock_movements_product ON stock_movements(product_id);
CREATE INDEX IF NOT EXISTS idx_stock_movements_date ON stock_movements(created_at);

-- Progreso de capas/dimensiones
CREATE TABLE IF NOT EXISTS layer_progress (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  
  layer_number INTEGER NOT NULL, -- 1, 2, 3
  dimension TEXT NOT NULL, -- 'hunger', 'energy', 'nutrition', 'variety'
  target_days INTEGER NOT NULL, -- 30, 60, 90
  
  is_unlocked INTEGER DEFAULT 0,
  progress_percentage REAL DEFAULT 0.0,
  
  last_calculated_at INTEGER,
  
  UNIQUE(user_id, layer_number, dimension),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_layer_progress_user ON layer_progress(user_id);

-- Alertas
CREATE TABLE IF NOT EXISTS alerts (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  product_id TEXT, -- nullable si alerta es general
  
  alert_type TEXT NOT NULL, -- 'expiring_soon', 'expired', 'low_stock', 'dimension_incomplete'
  severity TEXT NOT NULL, -- 'info', 'warning', 'critical'
  message TEXT NOT NULL,
  
  created_at INTEGER NOT NULL,
  dismissed_at INTEGER, -- NULL si no está dismissed
  
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_alerts_user ON alerts(user_id);
CREATE INDEX IF NOT EXISTS idx_alerts_dismissed ON alerts(dismissed_at);

-- Configuración del usuario
CREATE TABLE IF NOT EXISTS user_settings (
  user_id TEXT PRIMARY KEY,
  
  -- Notificaciones
  notify_expiry_days INTEGER DEFAULT 7, -- alertar N días antes
  notify_low_stock_threshold REAL DEFAULT 0.2, -- 20% del target
  
  -- Backup
  auto_backup_enabled INTEGER DEFAULT 0,
  auto_backup_frequency_days INTEGER DEFAULT 7,
  last_backup_at INTEGER,
  
  -- Preferencias
  theme TEXT DEFAULT 'fallout', -- 'fallout', 'minimal', 'dark'
  language TEXT DEFAULT 'es',
  
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Metadata de sincronización
CREATE TABLE IF NOT EXISTS sync_metadata (
  user_id TEXT PRIMARY KEY,
  last_export_at INTEGER,
  last_import_at INTEGER,
  db_version INTEGER DEFAULT 1,
  
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

---

## Queries críticas optimizadas

### 1. Calcular dimensión "Hambre"
```sql
-- Calorías totales disponibles
SELECT 
  SUM(
    fi.calories_per_100g * 
    p.package_size * 
    p.quantity_available / 100
  ) as total_calories
FROM products p
JOIN food_items fi ON p.food_item_id = fi.id
WHERE p.user_id = ?
  AND p.quantity_available > 0;

-- Target para 30 días, 1 persona: 60,000 kcal
-- Progress: (total_calories / 60000) * 100
```

### 2. Calcular dimensión "Energética" (balance macro)
```sql
-- Macronutrientes totales
SELECT 
  SUM(fi.protein_per_100g * p.package_size * p.quantity_available / 100) as total_protein,
  SUM(fi.carbs_per_100g * p.package_size * p.quantity_available / 100) as total_carbs,
  SUM(fi.fat_per_100g * p.package_size * p.quantity_available / 100) as total_fat,
  SUM(fi.calories_per_100g * p.package_size * p.quantity_available / 100) as total_calories
FROM products p
JOIN food_items fi ON p.food_item_id = fi.id
WHERE p.user_id = ?
  AND p.quantity_available > 0;

-- Calcular ratios en código:
-- protein_cals = total_protein * 4
-- carbs_cals = total_carbs * 4
-- fat_cals = total_fat * 9
-- Evaluar si están en rangos (50-60% carbos, 15-25% proteínas, 20-30% grasas)
```

### 3. Productos por expirar
```sql
SELECT 
  p.*,
  fi.name as food_name,
  fi.category
FROM products p
JOIN food_items fi ON p.food_item_id = fi.id
WHERE p.user_id = ?
  AND p.expiry_date IS NOT NULL
  AND p.expiry_date BETWEEN ? AND ? -- hoy y +7 días
ORDER BY p.expiry_date ASC;
```

### 4. Diversidad de categorías (dimensión "Variedad")
```sql
SELECT COUNT(DISTINCT fi.category) as unique_categories
FROM products p
JOIN food_items fi ON p.food_item_id = fi.id
WHERE p.user_id = ?
  AND p.quantity_available > 0;

-- Total de categorías posibles: 8 (grains, proteins, vegetables, fruits, dairy, fats, canned, legumes)
-- Progress: (unique_categories / 8) * 100
```

---

## Flujo de datos

### Primer uso
1. Usuario instala app
2. Login con Google → Firebase Auth devuelve `userId`
3. App crea entrada en tabla `users` con ese `userId`
4. Seed inicial: inserta alimentos básicos en `food_items` (arroz, atún, papa, etc.)
5. Inicializa `layer_progress` con capa 1 desbloqueada
6. Usuario empieza a agregar productos

### Agregar producto
1. Usuario selecciona alimento de catálogo (query a `food_items`)
2. Ingresa detalles de producto (marca, tamaño, precio, fecha compra/caducidad)
3. INSERT en tabla `products`
4. INSERT en tabla `stock_movements` (tipo: 'purchase')
5. Trigger recalcular dimensiones (async)

### Consumir producto (por receta)
1. Usuario selecciona receta del día
2. App verifica si tiene ingredientes (JOIN products + recipe_ingredients)
3. Usuario confirma consumo
4. UPDATE `products` (decrementa `quantity_available`)
5. INSERT en `stock_movements` (tipo: 'consumption', con `recipe_id`)
6. UPDATE `users` (incrementa `xp_points`)
7. Trigger recalcular dimensiones

### Recalcular dimensiones (background)
1. Para cada capa × dimensión:
   - Query específico según dimensión (hambre, energética, etc.)
   - Calcula progress_percentage
   - UPDATE `layer_progress`
2. Si dimensión alcanza 100% → check si desbloquear siguiente
3. Si progreso cambió → generar alerta si aplica

---

## Backup y sincronización

### Export (backup)
```typescript
async function exportDatabase(userId: string): Promise<string> {
  // 1. Leer todas las tablas relevantes
  const users = await db.getAllAsync('SELECT * FROM users WHERE id = ?', [userId]);
  const products = await db.getAllAsync('SELECT * FROM products WHERE user_id = ?', [userId]);
  const stockMovements = await db.getAllAsync('SELECT * FROM stock_movements WHERE user_id = ?', [userId]);
  const layerProgress = await db.getAllAsync('SELECT * FROM layer_progress WHERE user_id = ?', [userId]);
  // ... etc
  
  // 2. Serializar a JSON
  const backup = {
    version: 1,
    exportedAt: Date.now(),
    userId,
    data: {
      users,
      products,
      stockMovements,
      layerProgress,
      // ... etc
    }
  };
  
  // 3. Encriptar (opcional pero recomendado)
  const encrypted = await encryptBackup(JSON.stringify(backup), userId);
  
  return encrypted;
}

async function uploadBackup(userId: string, encryptedData: string) {
  const storageRef = ref(storage, `users/${userId}/backup_${Date.now()}.json.enc`);
  await uploadString(storageRef, encryptedData);
}
```

### Import (restore)
```typescript
async function downloadAndRestoreBackup(userId: string) {
  // 1. Descargar último backup de Firebase Storage
  const listRef = ref(storage, `users/${userId}/`);
  const list = await listAll(listRef);
  const latestBackup = list.items.sort((a, b) => b.name.localeCompare(a.name))[0];
  const encryptedData = await getDownloadURL(latestBackup);
  
  // 2. Desencriptar
  const decrypted = await decryptBackup(encryptedData, userId);
  const backup = JSON.parse(decrypted);
  
  // 3. Limpiar BD local (opcional, o merge)
  await db.execAsync('DELETE FROM products WHERE user_id = ?', [userId]);
  // ... etc
  
  // 4. Insertar datos
  for (const product of backup.data.products) {
    await db.runAsync(
      'INSERT INTO products (...) VALUES (...)',
      [...valores]
    );
  }
  // ... etc
}
```

---

## Ventajas de esta arquitectura

✅ **Narrativa coherente** — app funciona en apocalipsis  
✅ **Performance máximo** — cero latencia, todo local  
✅ **Queries complejas** — SQL completo sin restricciones  
✅ **Cero costos** — no pagas por queries  
✅ **Control total** — usuario es dueño de sus datos  
✅ **Backup opcional** — sincronización cuando el usuario quiere  
✅ **Privacidad** — datos no viven en servidor, solo backup encriptado opcional  

---

## Próximos pasos

1. Inicializar proyecto React Native + Expo
2. Configurar expo-sqlite
3. Crear migrations para schema inicial
4. Seed de alimentos básicos
5. Integrar Firebase Auth
6. Primera pantalla: Login + Dashboard
