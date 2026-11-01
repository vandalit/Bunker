# Catálogo Maestro de Alimentos - Arquitectura de Sincronización

> Cómo sincronizar el catálogo curado de `food_items` desde servidor hacia todos los dispositivos

---

## Distinción fundamental

### `food_items` — Catálogo maestro (templates curados)
- **Qué es:** Biblioteca de alimentos como objetos de programación
- **Quién lo crea:** Equipo Bunker (curado, validado)
- **Datos incluidos:** Valores nutricionales, peso, volumen, precio referencial, categoría
- **Cantidad inicial:** ~100 items (arroz, atún, papa, legumbres, etc.)
- **Crecimiento:** 1000+ con el tiempo (piña, quinoa, alimentos exóticos)
- **Ubicación canónica:** **Firebase Firestore** (fuente de verdad)

### `products` — Inventario real del usuario
- **Qué es:** Instancias físicas que el usuario tiene en su despensa
- **Quién lo crea:** Usuario final
- **Datos incluidos:** Cantidad, fecha de compra, fecha de caducidad, marca, ubicación
- **Ejemplo:** "Tengo 3 latas de atún Tunny, caducan 2027-05-15"
- **Ubicación:** **SQLite local** en cada dispositivo

---

## Problema a resolver

Cuando creas `food_item` nuevo (ej: "piña"):
1. Lo agregas a Firestore (source of truth)
2. **¿Cómo llega a los dispositivos de todos los usuarios?**
3. **¿Cuándo lo descargan?** (inmediato, lazy, on-demand)
4. **¿Cómo actualizamos items existentes?** (correcciones nutricionales)

---

## Arquitectura propuesta: Catálogo híbrido

### Decisión de DB

**Firebase Firestore para catálogo maestro** (no Realtime DB)

**Por qué Firestore y no Realtime DB:**
- ✅ Mejor para catálogo estático que crece (100 → 1000 items)
- ✅ Queries complejas (filtrar por categoría, buscar por nombre)
- ✅ Offline persistence nativa
- ✅ Cacheo automático
- ✅ Costo por lectura, no por conexión (ideal para catálogo)

**SQLite local se mantiene**
- ✅ Copia local del catálogo descargado
- ✅ Queries sin latencia
- ✅ Funciona offline completo
- ✅ No es PostgreSQL mobile porque no existe (solo SQLite en móvil)

---

## Modelo de datos revisado

### Firestore (servidor) — Source of truth

```javascript
// Colección: foodItems (catálogo maestro)
{
  id: "atun_lata_170g",
  version: 2, // para versionado de cambios
  name: "Atún en lata",
  name_es: "Atún en lata",
  name_en: "Canned Tuna",
  category: "canned",
  
  // Nutricional por 100g
  nutritional: {
    per100g: {
      calories: 116,
      protein: 26,
      carbs: 0,
      sugar: 0,
      fat: 0.8,
      saturated_fat: 0.2,
      fiber: 0,
      sodium: 320
    },
    vitamins: {
      A: 18,   // % RDA
      C: 0,
      D: 44,
      B12: 192
    },
    minerals: {
      iron: 6,
      calcium: 1,
      magnesium: 8,
      zinc: 5
    }
  },
  
  // Ingrediente
  ingredient: {
    baseUnit: "grams",
    minRecipePortion: 170, // 1 lata estándar
    flavorProfile: ["salty", "umami"],
    texture: "flaky"
  },
  
  // Energética
  energetic: {
    caloricDensity: "medium",
    satietyIndex: 82,
    digestionSpeed: "medium"
  },
  
  // Dietas
  dietTags: ["keto", "paleo", "high-protein"],
  allergens: [], // gluten-free por defecto
  
  // Presentaciones comerciales comunes
  commonPackageSizes: [170, 200, 320], // gramos
  averagePricePerKg: 8500, // CLP por kg (referencia)
  
  // Gamificación
  isUnlocked: true, // true = disponible desde inicio
  unlockRequirement: null, // o "layer_2_complete"
  
  // Metadata
  createdAt: timestamp,
  updatedAt: timestamp,
  createdBy: "admin_user_id",
  verifiedBy: "nutritionist_user_id",
  source: "USDA FoodData Central" // fuente de datos nutricionales
}
```

**Estructura de colección:**
```
foodItems/
  atun_lata_170g/
  arroz_blanco/
  papa/
  piña/  ← nuevo item cuando se agregue
  ...
```

---

### SQLite local (dispositivo) — Cache + extensiones

```sql
-- Catálogo descargado (copia de Firestore)
CREATE TABLE IF NOT EXISTS food_items (
  id TEXT PRIMARY KEY,
  version INTEGER NOT NULL,
  
  name TEXT NOT NULL,
  name_es TEXT,
  name_en TEXT,
  category TEXT NOT NULL,
  
  -- Nutricional (JSON serializado para flexibilidad)
  nutritional TEXT NOT NULL, -- JSON completo
  
  -- Ingrediente
  base_unit TEXT NOT NULL,
  min_recipe_portion REAL,
  flavor_profile TEXT, -- JSON array
  texture TEXT,
  
  -- Energética
  caloric_density TEXT,
  satiety_index REAL,
  
  -- Dietas
  diet_tags TEXT, -- JSON array
  allergens TEXT, -- JSON array
  
  -- Comercial
  common_package_sizes TEXT, -- JSON array
  average_price_per_kg REAL,
  
  -- Gamificación
  is_unlocked INTEGER DEFAULT 1,
  unlock_requirement TEXT,
  
  -- Metadata
  created_at INTEGER,
  updated_at INTEGER,
  
  -- Sync control
  synced_from_server INTEGER NOT NULL, -- timestamp de última sync
  local_only INTEGER DEFAULT 0 -- 0 = del servidor, 1 = creado localmente
);

CREATE INDEX IF NOT EXISTS idx_food_items_category ON food_items(category);
CREATE INDEX IF NOT EXISTS idx_food_items_unlocked ON food_items(is_unlocked);
CREATE INDEX IF NOT EXISTS idx_food_items_local ON food_items(local_only);

-- Tabla de versiones de catálogo (para sync incremental)
CREATE TABLE IF NOT EXISTS catalog_sync_metadata (
  id INTEGER PRIMARY KEY CHECK (id = 1), -- solo 1 fila
  last_sync_at INTEGER,
  catalog_version INTEGER DEFAULT 1,
  total_items INTEGER DEFAULT 0
);
```

---

## Estrategia de sincronización

### Fase 1: Descarga inicial (primera instalación)

**Cuándo:** Primera vez que usuario abre app (después de login)

**Flujo:**
1. Usuario hace login con Google
2. App verifica si `food_items` local está vacío
3. Si vacío → descarga catálogo completo desde Firestore:
   ```typescript
   const foodItemsSnapshot = await db
     .collection('foodItems')
     .where('isUnlocked', '==', true) // solo items desbloqueados
     .get();
   
   // Batch insert a SQLite
   await sqliteDb.transaction(tx => {
     foodItemsSnapshot.docs.forEach(doc => {
       const item = doc.data();
       tx.executeSql(
         `INSERT INTO food_items (id, version, name, ...) VALUES (?, ?, ?, ...)`,
         [doc.id, item.version, item.name, ...]
       );
     });
   });
   
   // Marcar sync completado
   await sqliteDb.executeSql(
     `INSERT INTO catalog_sync_metadata (id, last_sync_at, total_items) VALUES (1, ?, ?)`,
     [Date.now(), foodItemsSnapshot.size]
   );
   ```

**Costo:**
- 100 items × 1 lectura = 100 reads
- Firebase free tier: 50k reads/día → costo: $0
- Usuario solo paga esto una vez (en instalación)

**Tiempo:** ~2-5 segundos (background con loading spinner)

**Caché:** Firestore cachea automáticamente → instalaciones posteriores del mismo usuario son instantáneas

---

### Fase 2: Sync incremental (actualizaciones periódicas)

**Cuándo:** 
- Cada vez que app arranca (background silencioso)
- Cada 24-48 horas
- On-demand si usuario va a "Catálogo" y hace pull-to-refresh

**Flujo:**
1. App consulta última sync desde `catalog_sync_metadata`
2. Query incremental a Firestore:
   ```typescript
   const lastSyncTimestamp = await getLastSyncTimestamp();
   
   const updatedItems = await db
     .collection('foodItems')
     .where('updatedAt', '>', new Date(lastSyncTimestamp))
     .get();
   
   if (updatedItems.empty) {
     console.log('Catálogo al día');
     return;
   }
   
   // Actualizar items modificados
   await sqliteDb.transaction(tx => {
     updatedItems.docs.forEach(doc => {
       const item = doc.data();
       
       // UPSERT (insert or replace)
       tx.executeSql(
         `INSERT OR REPLACE INTO food_items (id, version, ...) VALUES (?, ?, ...)`,
         [doc.id, item.version, ...]
       );
     });
   });
   
   // Actualizar metadata
   await sqliteDb.executeSql(
     `UPDATE catalog_sync_metadata SET last_sync_at = ?, total_items = ?`,
     [Date.now(), totalItems]
   );
   ```

**Costo:**
- Solo items nuevos o modificados
- Ej: agregaste "piña" → 1 read por usuario (cuando sincronicen)
- Con 10,000 usuarios activos: 10k reads = ~$0.03

**Offline:** Si no hay conexión, sync se saltea. Funciona con catálogo local.

---

### Fase 3: On-demand (búsqueda de items no descargados)

**Caso:** Usuario busca "piña" pero aún no la tiene localmente

**Flujo:**
1. Usuario busca en UI: "piña"
2. App busca primero en SQLite local → no encuentra
3. App busca en Firestore (si hay conexión):
   ```typescript
   const searchResults = await db
     .collection('foodItems')
     .where('name', '>=', 'piña')
     .where('name', '<=', 'piña\uf8ff') // prefix search
     .limit(10)
     .get();
   
   if (!searchResults.empty) {
     // Guardar resultados en SQLite para futuro uso
     await cacheSearchResults(searchResults);
     return searchResults;
   }
   ```

**Ventaja:** Lazy loading — solo descargas items cuando usuario los necesita

---

## Versionado de items (correcciones nutricionales)

**Problema:** Descubres que el valor de proteína del atún estaba mal, lo corriges en Firestore.

**Solución:** Campo `version` en cada item

```typescript
// Firestore: actualizas el item
await db.collection('foodItems').doc('atun_lata_170g').update({
  'nutritional.per100g.protein': 28, // corregido de 26 a 28
  version: 3, // incrementas versión
  updatedAt: firebase.firestore.FieldValue.serverTimestamp()
});

// Dispositivos:
// En próximo sync incremental, detectan version > local
// Descargan cambio y reemplazan localmente
```

**Impacto en usuarios:**
- Si usuario tiene productos de ese item, se recalculan dimensiones automáticamente
- Transparente para el usuario

---

## Catálogo custom del usuario (opcional)

**Caso:** Usuario quiere agregar "Mermelada de frambuesa casera" que no está en catálogo oficial

**Solución:** Campo `local_only = 1` en SQLite

```typescript
// Usuario crea item custom
await sqliteDb.executeSql(
  `INSERT INTO food_items (id, name, category, ..., local_only, synced_from_server)
   VALUES (?, ?, ?, ..., 1, ?)`,
  [uuidv4(), 'Mermelada frambuesa casera', 'spreads', ..., Date.now()]
);
```

**Características:**
- Solo vive en dispositivo del usuario
- No se sincroniza a Firestore (es personal)
- Si cambia de dispositivo, puede exportar/importar en backup
- Opcional para MVP, puede ser feature v2

---

## PostgreSQL local vs SQLite

### ¿Por qué NO PostgreSQL mobile?

**Técnicamente no existe:**
- ❌ No hay PostgreSQL embebido para Android/iOS
- ❌ PostgreSQL requiere servidor (proceso separado)
- ❌ Overhead de ~20-50 MB solo para el binario

**SQLite es el estándar para mobile:**
- ✅ Embebido en OS (Android, iOS, todos los browsers)
- ✅ Overhead de ~1 MB
- ✅ Usado por: WhatsApp, Instagram, Dropbox, Apple Health
- ✅ SQL completo (ACID, triggers, views, CTEs)
- ✅ Limitaciones solo para casos extremos (millones de writes concurrentes)

**Para web (React):**
- sql.js o wa-sqlite (SQLite compilado a WASM)
- Funciona en browser, mismo API que mobile

**Conclusión:** SQLite es la elección correcta. PostgreSQL no es opción en mobile.

---

## Flujo completo: Agregar "piña" al catálogo

### 1. Equipo Bunker crea el item

```typescript
// Admin panel (web app interna)
await db.collection('foodItems').doc('piña').set({
  version: 1,
  name: "Piña",
  name_es: "Piña",
  name_en: "Pineapple",
  category: "fruits",
  nutritional: {
    per100g: {
      calories: 50,
      protein: 0.5,
      carbs: 13,
      sugar: 10,
      fat: 0.1,
      fiber: 1.4,
      sodium: 1
    },
    vitamins: { C: 80, A: 1, B6: 6 },
    minerals: { calcium: 1, magnesium: 3, iron: 2 }
  },
  ingredient: {
    baseUnit: "grams",
    minRecipePortion: 100, // 1 rodaja
    flavorProfile: ["sweet", "acidic"],
    texture: "juicy"
  },
  energetic: {
    caloricDensity: "low",
    satietyIndex: 65
  },
  dietTags: ["vegan", "low-fat"],
  commonPackageSizes: [500, 1000, 1500], // peso promedio
  averagePricePerKg: 2000, // CLP
  isUnlocked: true,
  createdAt: firebase.firestore.FieldValue.serverTimestamp(),
  updatedAt: firebase.firestore.FieldValue.serverTimestamp(),
  source: "USDA FoodData Central"
});
```

### 2. Usuarios sincronizan (próxima apertura de app)

```typescript
// Background sync al abrir app
const lastSync = await getLastSyncTimestamp(); // ej: hace 3 días

const newItems = await db.collection('foodItems')
  .where('createdAt', '>', new Date(lastSync))
  .get();

// newItems incluye "piña"
newItems.forEach(doc => {
  await insertOrUpdateFoodItem(doc.id, doc.data());
});

// UI notifica (opcional): "3 nuevos alimentos disponibles"
```

### 3. Usuario puede usar "piña"

```typescript
// Usuario va a "Agregar producto"
// Busca "piña" → ahora aparece en catálogo local
// Crea producto: "Piña Golden, 1.2 kg, comprada hoy"
// Se asocia a food_item_id = "piña"
```

### 4. Costos

- Creación en Firestore: 1 write = $0.0000018
- Sync a 10,000 usuarios: 10k reads = $0.036
- **Total para agregar 1 item nuevo: ~$0.04**

---

## Escalabilidad

### Catálogo de 100 → 1000 items

**Descarga inicial:**
- 100 items: ~2 segundos
- 1000 items: ~15-20 segundos (primera vez solamente)
- Mitigación: descarga en background mientras usuario ve onboarding

**Búsqueda:**
- SQLite maneja millones de filas sin problema
- Con 1000 items + índices: búsqueda <10ms

**Almacenamiento:**
- 1 food_item serializado: ~2-3 KB
- 1000 items: ~2.5 MB
- Despreciable en dispositivos modernos (mínimo 32 GB)

**Sync incremental:**
- Solo items nuevos/modificados desde última sync
- Con 10 items nuevos/semana: 10 reads por usuario
- Costo mensual con 10k usuarios: ~$1.50

---

## Resumen de decisiones

| Aspecto | Decisión |
|---|---|
| **Catálogo maestro** | Firestore (source of truth) |
| **DB local mobile** | SQLite (expo-sqlite) |
| **DB local web** | SQLite WASM (sql.js) |
| **Sync strategy** | Incremental: solo items nuevos/modificados |
| **First install** | Descarga completa en background |
| **Updates** | Cada 24-48h o on-demand |
| **Versionado** | Campo `version` para correcciones |
| **Custom items** | Opcional con `local_only = 1` |

**Stack confirmado:**
- ✅ Firestore para catálogo maestro
- ✅ SQLite local para cache + inventario
- ✅ Sync incremental basado en timestamps
- ✅ Offline-first sigue intacto

---

**Arquitectura bien diseñada:** Escala de 100 a 10,000 items sin problema, costos predecibles, y paradigma offline-first preservado.
