# Sync Multi-Device y Multi-Vault - Análisis

> ¿Cambia la decisión de SQLite local? ¿Cómo implementamos sync + permisos?

---

## Nuevos requerimientos

### 1. Multi-device sync
- Usuario puede acceder desde: móvil Android, móvil iOS, web (PC)
- Cambios en un dispositivo se sincronizan a los demás
- **Offline-first sigue siendo mandatorio** — si pierdes conexión, sigues trabajando

### 2. Multi-vault (feature de pago)
- **Gratis:** 1 propietario, 1 vault personal
- **Premium:** Crear múltiples vaults, invitar miembros familiares

**Roles en vault:**
- **Owner/Admin (propietario):** puede leer y escribir
- **Viewer (miembro invitado - gratis):** solo lectura
- **Editor (miembro invitado - premium):** lectura + escritura

**Casos de uso:**
- Papá crea vault familiar, invita a mamá (viewer) y ella ve inventario desde su móvil
- Con premium, mamá puede también agregar/consumir productos
- Cada miembro puede tener su propia vault personal adicional

---

## ¿Cambia la decisión de SQLite local?

### Respuesta: **NO cambia, pero se complejiza**

**SQLite local sigue siendo correcto porque:**
- ✅ Offline-first es mandatorio (requerimiento base)
- ✅ Performance local es crítico para UX fluido
- ✅ Queries complejas (dimensiones) deben correr local sin latencia

**Lo que SÍ necesitamos adicional:**
- Sistema de **sincronización bidireccional** entre dispositivos
- **Backend ligero** para coordinar sync y permisos
- **Resolución de conflictos** cuando dos dispositivos editan offline

---

## Arquitectura revisada

### Opción A: SQLite + Firebase Realtime DB (sync layer) — RECOMENDADO

**Stack:**
- **SQLite local** (cada dispositivo tiene copia completa de sus vaults)
- **Firebase Realtime Database** (capa de sincronización, no de almacenamiento)
- **Firebase Auth** (autenticación + userId)
- **Firebase Cloud Functions** (lógica de permisos + resolución de conflictos)

**Cómo funciona:**

#### 1. Estructura de datos en Firebase Realtime DB
```javascript
{
  // Metadata de vaults (ligera, solo info de coordinación)
  vaults: {
    "vault_uuid_1": {
      ownerId: "user_123",
      name: "Vault Familiar García",
      createdAt: 1698765432,
      members: {
        "user_123": { role: "owner" },
        "user_456": { role: "viewer" },
        "user_789": { role: "editor" } // premium
      },
      lastModifiedAt: 1698765999,
      lastModifiedBy: "user_123"
    }
  },
  
  // Changesets (operaciones delta, no datos completos)
  changesets: {
    "vault_uuid_1": {
      "changeset_001": {
        deviceId: "device_A",
        userId: "user_123",
        timestamp: 1698765500,
        operations: [
          {
            op: "insert",
            table: "products",
            id: "prod_001",
            data: { name: "Arroz", quantity: 5, ... }
          },
          {
            op: "update",
            table: "products",
            id: "prod_002",
            changes: { quantity: 3 } // delta, no objeto completo
          }
        ],
        synced: true
      },
      "changeset_002": { ... }
    }
  },
  
  // Dispositivos conectados (presencia)
  devices: {
    "user_123": {
      "device_A": { lastSeen: 1698765600, platform: "android" },
      "device_B": { lastSeen: 1698765500, platform: "web" }
    }
  }
}
```

#### 2. Flujo de sincronización

**Escritura (device A hace cambio):**
1. Usuario agrega producto en device A (offline o online)
2. Cambio se escribe primero en SQLite local (instantáneo)
3. Cambio se encola en `pending_sync` table (SQLite)
4. Cuando hay conexión:
   - Se genera `changeset` con operaciones delta
   - Se publica en Firebase Realtime DB bajo `changesets/{vaultId}/`
   - Timestamp + deviceId para detección de conflictos

**Lectura (device B recibe cambio):**
1. Device B está subscrito a `changesets/{vaultId}/` (listener)
2. Cuando llega nuevo changeset:
   - Valida permisos (¿el usuario que hizo el cambio tiene rol editor/owner?)
   - Aplica operaciones delta a SQLite local
   - Marca changeset como `synced`
3. UI se actualiza automáticamente (reactive)

**Conflictos:**
- Si dos dispositivos editan mismo producto offline:
  - Timestamp gana (last-write-wins)
  - O merge inteligente (suma de cantidades si ambos agregan stock)
  - Cloud Function arbitrea en casos complejos

#### 3. Permisos

**Validación en Cloud Function:**
```javascript
// functions/src/index.ts
export const validateChangeset = functions.database
  .ref('/changesets/{vaultId}/{changesetId}')
  .onCreate(async (snapshot, context) => {
    const changeset = snapshot.val();
    const { vaultId } = context.params;
    
    // 1. Leer metadata de vault
    const vaultSnap = await admin.database()
      .ref(`/vaults/${vaultId}`)
      .once('value');
    const vault = vaultSnap.val();
    
    // 2. Verificar rol del usuario
    const userRole = vault.members[changeset.userId]?.role;
    
    if (userRole === 'viewer') {
      // Viewer no puede escribir — rechazar changeset
      await snapshot.ref.update({ 
        synced: false, 
        rejected: true,
        reason: 'Permission denied: viewer role'
      });
      return;
    }
    
    if (userRole === 'editor' && !vault.premiumEnabled) {
      // Editor sin premium — rechazar
      await snapshot.ref.update({ 
        synced: false, 
        rejected: true,
        reason: 'Premium required for editor role'
      });
      return;
    }
    
    // 3. Aprobado — marcar como synced
    await snapshot.ref.update({ synced: true });
    
    // 4. Actualizar lastModifiedAt en vault
    await admin.database()
      .ref(`/vaults/${vaultId}`)
      .update({ 
        lastModifiedAt: changeset.timestamp,
        lastModifiedBy: changeset.userId
      });
  });
```

**En cada dispositivo:**
- Antes de escribir, verifica rol localmente (UX)
- Si es viewer, botones de escritura deshabilitados
- Si intenta hackear y escribir igual, Cloud Function rechaza

---

### Opción B: SQLite + Supabase Realtime — ALTERNATIVA

**Diferencias vs Firebase:**
- Supabase tiene Postgres en backend (podríamos migrar data allá)
- Realtime subscriptions con Row Level Security (RLS)
- Postgres triggers para validar permisos

**Ventaja:** SQL en backend también (queries complejas server-side)
**Desventaja:** Más complejo que Firebase Realtime DB para sync puro

**Evaluación:** Viable, pero Firebase es más simple para sync. Si queremos queries server-side, considerarlo.

---

### Opción C: SQLite + CRDTs (Conflict-free Replicated Data Types)

**Qué es:** Estructuras de datos que se sincronizan automáticamente sin conflictos
**Librerías:** Yjs, Automerge, ElectricSQL

**Ventajas:**
- ✅ Sync automático sin backend custom
- ✅ Resolución de conflictos matemáticamente correcta
- ✅ Offline-first perfecto

**Desventajas:**
- ⚠️ Curva de aprendizaje alta
- ⚠️ Menos control sobre permisos (requiere wrapper)
- ⚠️ Overhead de metadata (CRDTs agregan peso)

**Evaluación:** Overkill para MVP, considerar en v2 si sync se vuelve crítico

---

## Cambios en el modelo SQLite

### Nuevas tablas necesarias

```sql
-- Vaults (bóvedas)
CREATE TABLE IF NOT EXISTS vaults (
  id TEXT PRIMARY KEY,
  owner_id TEXT NOT NULL, -- Firebase UID del creador
  name TEXT NOT NULL,
  description TEXT,
  
  is_shared INTEGER DEFAULT 0, -- boolean
  premium_enabled INTEGER DEFAULT 0,
  
  created_at INTEGER NOT NULL,
  last_modified_at INTEGER NOT NULL,
  last_modified_by TEXT, -- userId que hizo último cambio
  
  FOREIGN KEY (owner_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Miembros de vault (relación N:M)
CREATE TABLE IF NOT EXISTS vault_members (
  id TEXT PRIMARY KEY,
  vault_id TEXT NOT NULL,
  user_id TEXT NOT NULL,
  role TEXT NOT NULL, -- 'owner', 'editor', 'viewer'
  
  invited_at INTEGER NOT NULL,
  invited_by TEXT NOT NULL, -- userId del que invitó
  accepted_at INTEGER, -- NULL si invitación pendiente
  
  UNIQUE(vault_id, user_id),
  FOREIGN KEY (vault_id) REFERENCES vaults(id) ON DELETE CASCADE,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_vault_members_vault ON vault_members(vault_id);
CREATE INDEX IF NOT EXISTS idx_vault_members_user ON vault_members(user_id);

-- Modificar tabla products para asociar a vault
ALTER TABLE products ADD COLUMN vault_id TEXT REFERENCES vaults(id) ON DELETE CASCADE;
CREATE INDEX IF NOT EXISTS idx_products_vault ON products(vault_id);

-- Modificar otras tablas relevantes
ALTER TABLE stock_movements ADD COLUMN vault_id TEXT REFERENCES vaults(id);
ALTER TABLE layer_progress ADD COLUMN vault_id TEXT REFERENCES vaults(id);
ALTER TABLE alerts ADD COLUMN vault_id TEXT REFERENCES vaults(id);

-- Cola de sincronización pendiente
CREATE TABLE IF NOT EXISTS pending_sync (
  id TEXT PRIMARY KEY,
  vault_id TEXT NOT NULL,
  device_id TEXT NOT NULL,
  
  operation TEXT NOT NULL, -- 'insert', 'update', 'delete'
  table_name TEXT NOT NULL,
  record_id TEXT NOT NULL,
  data TEXT, -- JSON con datos completos (insert) o delta (update)
  
  created_at INTEGER NOT NULL,
  synced_at INTEGER, -- NULL si pendiente
  attempts INTEGER DEFAULT 0,
  last_error TEXT,
  
  FOREIGN KEY (vault_id) REFERENCES vaults(id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_pending_sync_vault ON pending_sync(vault_id);
CREATE INDEX IF NOT EXISTS idx_pending_sync_status ON pending_sync(synced_at) WHERE synced_at IS NULL;

-- Metadata de dispositivo
CREATE TABLE IF NOT EXISTS devices (
  id TEXT PRIMARY KEY, -- UUID generado en instalación
  user_id TEXT NOT NULL,
  
  platform TEXT NOT NULL, -- 'android', 'ios', 'web'
  device_name TEXT, -- "iPhone de Juan", "PC Casa"
  
  last_sync_at INTEGER,
  last_seen_at INTEGER,
  
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

---

## Flujos de usuario actualizados

### Setup inicial (primera vez)
1. Usuario instala app
2. Login con Google → Firebase Auth
3. App crea:
   - Entrada en `users`
   - Primera vault personal (nombre: "Mi Bunker")
   - Entrada en `vault_members` (role: 'owner')
   - Device entry
4. Seed alimentos básicos
5. Usuario empieza a usar

### Invitar miembro a vault (premium)
1. Owner abre vault settings
2. Ingresa email del familiar
3. Selecciona rol:
   - **Viewer (gratis):** solo lectura
   - **Editor (requiere premium):** lectura + escritura
4. App crea entrada en `vault_members` con `accepted_at = NULL`
5. Sistema envía invitación (email o notificación push)
6. Invitado acepta:
   - Su app descarga changeset inicial (toda la vault)
   - Crea copia local en SQLite
   - Empieza a escuchar cambios

### Editar desde múltiples dispositivos
**Escenario:** Papá agrega arroz en móvil, mamá ve cambio en PC

1. **Móvil (Papá, offline):**
   - Agrega producto → INSERT en SQLite local
   - INSERT en `pending_sync`
   
2. **Móvil (Papá, vuelve online):**
   - Background task detecta pending_sync
   - Publica changeset a Firebase
   - Marca como synced
   
3. **PC (Mamá, online):**
   - Listener de Firebase detecta nuevo changeset
   - Valida permisos (papá es owner → OK)
   - Aplica INSERT a SQLite local
   - UI se actualiza → mamá ve nuevo producto
   
4. **Móvil (Mamá, viewer role):**
   - Ve cambio en tiempo real
   - Botones de "Agregar/Editar" están deshabilitados
   - Solo puede ver inventario y estadísticas

---

## Web app (acceso desde PC)

### Stack para web
- **React (Vite)** — misma lógica de negocio que mobile
- **SQLite WASM** (sql.js o wa-sqlite) — SQLite en browser
- **Firebase SDK Web** — Auth + Realtime DB
- **Zustand** — estado compartido con mobile (mismo código)

**Ventajas:**
- Reutilizas 80% del código (lógica de negocio, queries, sync)
- Mismo paradigma offline-first (SQLite en IndexedDB del browser)
- UI puede ser más rica (pantallas grandes, teclado)

**Caso de uso:** Usuario revisa inventario desde PC, planifica compras, imprime lista

---

## Monetización actualizada

### Free (gratis total)
- 1 vault personal
- Solo el propietario usa la app
- Puede invitar viewers (solo lectura) — GRATIS para invitados también
- Backup manual a Firebase Storage
- Acceso mobile (Android/iOS)

### Premium (pago mensual/anual)
- Múltiples vaults (familiar, secundaria, etc.)
- Invitar **editors** (lectura + escritura) — requiere premium del owner
- Sincronización automática multi-device
- Acceso web desde PC/laptop
- Backup automático
- Temas premium (Fallout deluxe, minimalista, etc.)
- Soporte prioritario

**Precio sugerido:** $2-3 USD/mes o $20/año

**Justificación del precio:**
- Feature real y valiosa (multi-vault + sync + web)
- No es cosmético, es funcional
- Familias lo justifican fácil ($3/mes para toda la familia)

---

## ¿Cambia la decisión? — Veredicto

### ✅ SQLite local SE MANTIENE

**Razones:**
- Offline-first sigue siendo mandatorio (cultura prep)
- Performance local es crítico
- Queries complejas deben correr sin latencia
- Web app puede usar SQLite WASM

### ✅ Agregamos capa de sync con Firebase Realtime DB

**No es reemplazo, es complemento:**
- Firebase solo coordina changesets (ligero)
- No almacena datos completos (solo deltas)
- Permisos validados por Cloud Functions
- Costo mínimo (free tier cubre MVP fácil)

### Stack final confirmado

| Capa | Tecnología |
|---|---|
| **DB local** | SQLite (expo-sqlite mobile, sql.js web) |
| **Sync** | Firebase Realtime DB (changesets) |
| **Auth** | Firebase Auth |
| **Permisos** | Cloud Functions (validación server-side) |
| **Backup** | Firebase Storage |
| **Frontend Mobile** | React Native + Expo |
| **Frontend Web** | React + Vite |
| **Estado** | Zustand |
| **Queries** | SQL directo (Drizzle opcional) |

---

## Complejidad agregada

### Nuevo (vs arquitectura inicial):
- ✅ Sistema de changesets (delta sync)
- ✅ Tabla `pending_sync` para queue
- ✅ Cloud Functions para validación de permisos
- ✅ Listeners de Firebase en cada dispositivo
- ✅ Resolución de conflictos (simple: last-write-wins)
- ✅ Web app adicional (React)

### Estimación de esfuerzo:
- MVP sin sync: **3-4 semanas**
- MVP con sync multi-device: **5-7 semanas**
- Web app adicional: **+2 semanas**

**Total MVP completo:** ~8-9 semanas de desarrollo full-time

---

## Riesgos y mitigaciones

### Riesgo 1: Sincronización fallando bajo conexión inestable
**Mitigación:**
- Cola de pending_sync con retry automático
- Exponential backoff
- UI muestra "Sincronizando..." mientras está pendiente
- Funciona offline siempre, sync es transparente

### Riesgo 2: Conflictos complejos
**Mitigación:**
- Empezar con last-write-wins (simple)
- Para productos: merge inteligente (suma cantidades si ambos agregan)
- Marcar conflictos para revisión manual del owner (raro)

### Riesgo 3: Costos de Firebase
**Mitigación:**
- Changesets son ligeros (solo deltas, no objetos completos)
- Free tier: 10GB storage, 1GB transfer/día — suficiente para miles de usuarios
- Costo estimado con 1000 usuarios activos: ~$5-10/mes

### Riesgo 4: Permisos burlados desde cliente
**Mitigación:**
- Cloud Functions validan todo (server-side, no puede bypassearse)
- Cliente solo bloquea UI (comodidad)
- Si usuario manipula SQLite local, changesets rechazados en backend

---

## Próximos pasos revisados

1. ✅ Decisión tomada: SQLite + Firebase Realtime DB
2. Inicializar proyecto React Native + Expo
3. Setup SQLite con schema completo (incluyendo vaults, vault_members, pending_sync)
4. Integrar Firebase Auth
5. Crear Cloud Functions para validación de permisos
6. Implementar sistema de changesets
7. Primera feature: Login + Crear vault + Agregar producto
8. Segunda feature: Invitar miembro + Sync básico
9. Web app (fase 2)

---

**Decisión final: ✅ SQLite local + Firebase Realtime DB para sync.**

La arquitectura es más compleja pero sigue siendo offline-first y ahora habilita features premium valiosas.
