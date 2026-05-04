# Module 10 — Storage, Offline & Caching

> Mobile apps must work reliably with no internet. This module covers every storage layer — from in-memory key-value to relational SQLite — and the patterns needed to build a truly offline-first experience.

---

## Table of Contents

1. [Key-Value Storage](#1-key-value-storage)
   - 1.1 [AsyncStorage Basics](#11-asyncstorage-basics)
   - 1.2 [react-native-mmkv (Recommended)](#12-react-native-mmkv-recommended)
   - 1.3 [expo-secure-store (Sensitive Data)](#13-expo-secure-store-sensitive-data)
   - 1.4 [Storage Encryption](#14-storage-encryption)

2. [Database Solutions](#2-database-solutions)
   - 2.1 [SQLite (expo-sqlite)](#21-sqlite-expo-sqlite)
   - 2.2 [WatermelonDB](#22-watermelondb)
   - 2.3 [Realm / MongoDB Realm](#23-realm--mongodb-realm)
   - 2.4 [op-sqlite (Performance)](#24-op-sqlite-performance)

3. [Offline-First Patterns](#3-offline-first-patterns)
   - 3.1 [Cache-Then-Network](#31-cache-then-network)
   - 3.2 [Optimistic UI Updates](#32-optimistic-ui-updates)
   - 3.3 [Queue-Based Sync](#33-queue-based-sync)
   - 3.4 [Conflict Resolution](#34-conflict-resolution)
   - 3.5 [Network State Detection](#35-network-state-detection)

4. [Data Synchronization](#4-data-synchronization)
   - 4.1 [Last-Write-Wins](#41-last-write-wins)
   - 4.2 [Timestamp-Based Sync](#42-timestamp-based-sync)
   - 4.3 [Delta Sync](#43-delta-sync)
   - 4.4 [Background Sync](#44-background-sync)
   - 4.5 [TanStack Query Persistence](#45-tanstack-query-persistence)

5. [Cache Management](#5-cache-management)
   - 5.1 [Image Caching (expo-image)](#51-image-caching-expo-image)
   - 5.2 [API Response Caching](#52-api-response-caching)
   - 5.3 [Cache Invalidation](#53-cache-invalidation)
   - 5.4 [Storage Limits and Cleanup](#54-storage-limits-and-cleanup)

---

## 1. Key-Value Storage

### 1.1 AsyncStorage Basics

`AsyncStorage` is the simplest persistent key-value store. It is asynchronous, unencrypted, and stores plain strings.

```bash
npx expo install @react-native-async-storage/async-storage
```

```tsx
import AsyncStorage from '@react-native-async-storage/async-storage';

// ─── Write ─────────────────────────────────────────────────────────────────
await AsyncStorage.setItem('theme', 'dark');

// Objects must be serialized
await AsyncStorage.setItem('user', JSON.stringify({ id: '1', name: 'Nawaz' }));

// ─── Read ──────────────────────────────────────────────────────────────────
const theme = await AsyncStorage.getItem('theme'); // 'dark' | null

const raw = await AsyncStorage.getItem('user');
const user = raw ? JSON.parse(raw) : null;

// ─── Delete ────────────────────────────────────────────────────────────────
await AsyncStorage.removeItem('theme');

// ─── Batch operations ──────────────────────────────────────────────────────
// multiSet is faster than multiple setItem calls
await AsyncStorage.multiSet([
  ['theme', 'dark'],
  ['language', 'en'],
  ['onboarded', 'true'],
]);

const values = await AsyncStorage.multiGet(['theme', 'language']);
// [['theme', 'dark'], ['language', 'en']]

await AsyncStorage.multiRemove(['theme', 'language']);

// ─── Clear everything ──────────────────────────────────────────────────────
await AsyncStorage.clear(); // wipes all keys — use with caution

// ─── List all keys ─────────────────────────────────────────────────────────
const allKeys = await AsyncStorage.getAllKeys();
```

**Typed wrapper — avoid raw JSON everywhere:**
```tsx
// src/lib/storage.ts
import AsyncStorage from '@react-native-async-storage/async-storage';

export const storage = {
  async get<T>(key: string): Promise<T | null> {
    const raw = await AsyncStorage.getItem(key);
    if (!raw) return null;
    try { return JSON.parse(raw) as T; }
    catch { return raw as unknown as T; }
  },

  async set<T>(key: string, value: T): Promise<void> {
    const serialized = typeof value === 'string' ? value : JSON.stringify(value);
    await AsyncStorage.setItem(key, serialized);
  },

  async remove(key: string): Promise<void> {
    await AsyncStorage.removeItem(key);
  },

  async clear(): Promise<void> {
    await AsyncStorage.clear();
  },
};

// Usage
await storage.set('user', { id: '1', name: 'Nawaz' });
const user = await storage.get<{ id: string; name: string }>('user');
```

**When to use AsyncStorage:**
- Non-sensitive preferences: theme, language, onboarding state
- Zustand/Redux persist adapter
- Feature flags
- When you need a simple, zero-setup solution

**When NOT to use AsyncStorage:**
- Tokens, passwords, PII — use SecureStore
- Performance-critical reads/writes — use MMKV
- Relational or complex queried data — use SQLite

---

### 1.2 react-native-mmkv (Recommended)

MMKV is a **synchronous**, high-performance key-value store backed by Facebook's MMKV library. Up to 30× faster than AsyncStorage.

```bash
npx expo install react-native-mmkv
```

> Requires a Dev Client build — not available in Expo Go.

```tsx
import { MMKV } from 'react-native-mmkv';

// Create a storage instance
export const storage = new MMKV({
  id: 'app-storage',               // unique instance ID
  path: `${appDir}/mmkv`,          // custom path (optional)
  encryptionKey: 'my-secret-key',  // AES encryption (optional)
});
```

**MMKV is synchronous — no await needed:**
```tsx
// Write
storage.set('theme', 'dark');
storage.set('count', 42);
storage.set('isOnboarded', true);
storage.set('user', JSON.stringify({ id: '1', name: 'Nawaz' }));

// Read
const theme = storage.getString('theme');    // string | undefined
const count = storage.getNumber('count');    // number | undefined
const flag  = storage.getBoolean('isOnboarded'); // boolean | undefined
const raw   = storage.getString('user');
const user  = raw ? JSON.parse(raw) : null;

// Delete
storage.delete('theme');

// Check existence
const exists = storage.contains('theme'); // boolean

// All keys
const allKeys = storage.getAllKeys(); // string[]

// Clear instance
storage.clearAll();
```

**MMKV as Zustand storage adapter:**
```tsx
// src/lib/mmkvStorage.ts
import { MMKV } from 'react-native-mmkv';
import { StateStorage } from 'zustand/middleware';

const mmkv = new MMKV({ id: 'zustand-store' });

export const mmkvStorage: StateStorage = {
  getItem: (key) => mmkv.getString(key) ?? null,
  setItem: (key, value) => mmkv.set(key, value),
  removeItem: (key) => mmkv.delete(key),
};

// In your Zustand store — replace createJSONStorage(() => AsyncStorage)
persist(
  (set) => ({ /* store */ }),
  {
    name: 'app-store',
    storage: createJSONStorage(() => mmkvStorage),
  }
)
```

**React hook for reactive MMKV values:**
```tsx
import { useMMKVString, useMMKVNumber, useMMKVBoolean } from 'react-native-mmkv';

function ThemeToggle() {
  const [theme, setTheme] = useMMKVString('theme', storage);
  // Re-renders when 'theme' changes — like useState but persisted

  return (
    <Switch
      value={theme === 'dark'}
      onValueChange={v => setTheme(v ? 'dark' : 'light')}
    />
  );
}
```

---

### 1.3 expo-secure-store (Sensitive Data)

Covered in detail in Module 9. Quick reference for storage:

```tsx
import * as SecureStore from 'expo-secure-store';

// Auth tokens, API keys, PII — anything sensitive
await SecureStore.setItemAsync('accessToken', token, {
  keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
});

const token = await SecureStore.getItemAsync('accessToken');
await SecureStore.deleteItemAsync('accessToken');
```

**Storage layer decision matrix:**

| Data Type | Store | Why |
|---|---|---|
| Auth tokens, passwords | `expo-secure-store` | Keychain/Keystore encryption |
| App preferences, settings | `react-native-mmkv` | Fast synchronous reads |
| Large structured data | `expo-sqlite` | Queryable, relational |
| Zustand/Redux state | `mmkv` + `persist` | Fast, synchronous adapter |
| TanStack Query cache | `AsyncStorage` + persister | Simple, no sync needed |
| Offline queue | `mmkv` or `sqlite` | Depends on complexity |

---

### 1.4 Storage Encryption

**MMKV encryption (AES-256):**
```tsx
import { MMKV } from 'react-native-mmkv';
import * as SecureStore from 'expo-secure-store';
import * as Crypto from 'expo-crypto';

async function createEncryptedStorage() {
  // Retrieve or generate a persistent encryption key
  let encryptionKey = await SecureStore.getItemAsync('mmkv-key');

  if (!encryptionKey) {
    // Generate a cryptographically secure random key
    const bytes = await Crypto.getRandomBytesAsync(32);
    encryptionKey = Buffer.from(bytes).toString('hex');
    await SecureStore.setItemAsync('mmkv-key', encryptionKey, {
      keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
    });
  }

  return new MMKV({
    id: 'encrypted-storage',
    encryptionKey, // stored securely in Keychain
  });
}
```

**SQLite encryption with SQLCipher:**
```tsx
// expo-sqlite v14+ supports encryption via SQLCipher
import * as SQLite from 'expo-sqlite';
import * as SecureStore from 'expo-secure-store';

async function openEncryptedDatabase() {
  let dbKey = await SecureStore.getItemAsync('sqlite-key');
  if (!dbKey) {
    const bytes = await Crypto.getRandomBytesAsync(32);
    dbKey = Buffer.from(bytes).toString('hex');
    await SecureStore.setItemAsync('sqlite-key', dbKey);
  }

  const db = await SQLite.openDatabaseAsync('app.db', {
    enableChangeListener: true,
  });

  await db.execAsync(`PRAGMA key = '${dbKey}';`); // SQLCipher key
  return db;
}
```

---

## 2. Database Solutions

### 2.1 SQLite (expo-sqlite)

SQLite is a full relational database embedded in the app. Ideal for structured, queryable data that needs to persist.

```bash
npx expo install expo-sqlite
```

**Setup and migrations:**
```tsx
// src/db/database.ts
import * as SQLite from 'expo-sqlite';

let db: SQLite.SQLiteDatabase;

export async function getDatabase(): Promise<SQLite.SQLiteDatabase> {
  if (db) return db;

  db = await SQLite.openDatabaseAsync('app.db');

  // Run migrations on open
  await migrate(db);

  return db;
}

async function migrate(db: SQLite.SQLiteDatabase) {
  // Version-based migrations
  await db.execAsync(`PRAGMA journal_mode = WAL;`); // better write performance

  const version = await db.getFirstAsync<{ user_version: number }>(
    'PRAGMA user_version'
  );
  const currentVersion = version?.user_version ?? 0;

  if (currentVersion < 1) {
    await db.execAsync(`
      CREATE TABLE IF NOT EXISTS products (
        id TEXT PRIMARY KEY NOT NULL,
        name TEXT NOT NULL,
        price REAL NOT NULL,
        category TEXT,
        imageUrl TEXT,
        syncedAt INTEGER,
        createdAt INTEGER NOT NULL DEFAULT (strftime('%s','now'))
      );
      CREATE INDEX IF NOT EXISTS idx_products_category ON products(category);
      PRAGMA user_version = 1;
    `);
  }

  if (currentVersion < 2) {
    await db.execAsync(`
      ALTER TABLE products ADD COLUMN isFavorite INTEGER NOT NULL DEFAULT 0;
      PRAGMA user_version = 2;
    `);
  }
}
```

**CRUD operations:**
```tsx
// src/db/productsRepository.ts
import { getDatabase } from './database';

type Product = { id: string; name: string; price: number; category: string };

export const productsRepo = {
  // Insert / upsert
  upsert: async (product: Product) => {
    const db = await getDatabase();
    await db.runAsync(
      `INSERT INTO products (id, name, price, category)
       VALUES (?, ?, ?, ?)
       ON CONFLICT(id) DO UPDATE SET
         name = excluded.name,
         price = excluded.price,
         category = excluded.category`,
      [product.id, product.name, product.price, product.category]
    );
  },

  // Batch upsert — fast for syncing many records
  upsertMany: async (products: Product[]) => {
    const db = await getDatabase();
    await db.withTransactionAsync(async () => {
      for (const p of products) {
        await db.runAsync(
          `INSERT OR REPLACE INTO products (id, name, price, category) VALUES (?, ?, ?, ?)`,
          [p.id, p.name, p.price, p.category]
        );
      }
    });
  },

  // Query
  getAll: async (): Promise<Product[]> => {
    const db = await getDatabase();
    return db.getAllAsync<Product>('SELECT * FROM products ORDER BY name ASC');
  },

  getByCategory: async (category: string): Promise<Product[]> => {
    const db = await getDatabase();
    return db.getAllAsync<Product>(
      'SELECT * FROM products WHERE category = ? ORDER BY price ASC',
      [category]
    );
  },

  search: async (query: string): Promise<Product[]> => {
    const db = await getDatabase();
    return db.getAllAsync<Product>(
      `SELECT * FROM products WHERE name LIKE ? OR category LIKE ?`,
      [`%${query}%`, `%${query}%`]
    );
  },

  getById: async (id: string): Promise<Product | null> => {
    const db = await getDatabase();
    return db.getFirstAsync<Product>('SELECT * FROM products WHERE id = ?', [id]);
  },

  delete: async (id: string) => {
    const db = await getDatabase();
    await db.runAsync('DELETE FROM products WHERE id = ?', [id]);
  },

  count: async (): Promise<number> => {
    const db = await getDatabase();
    const row = await db.getFirstAsync<{ count: number }>('SELECT COUNT(*) as count FROM products');
    return row?.count ?? 0;
  },
};
```

**Reactive queries with `useSQLiteQuery`:**
```tsx
import { useSQLiteContext } from 'expo-sqlite';

// Wrap app with SQLiteProvider
<SQLiteProvider databaseName="app.db" onInit={migrate}>
  <App />
</SQLiteProvider>

// In a component — reactive query updates on DB change
function ProductList() {
  const db = useSQLiteContext();
  const [products, setProducts] = useState<Product[]>([]);

  useEffect(() => {
    const subscription = db.addListener('change', async ({ tableName }) => {
      if (tableName === 'products') {
        const updated = await db.getAllAsync<Product>('SELECT * FROM products');
        setProducts(updated);
      }
    });

    // Initial load
    db.getAllAsync<Product>('SELECT * FROM products').then(setProducts);

    return () => subscription.remove();
  }, [db]);

  return <FlatList data={products} renderItem={({ item }) => <ProductCard product={item} />} />;
}
```

---

### 2.2 WatermelonDB

WatermelonDB is a high-performance reactive database built on SQLite. Designed for apps with thousands of records and complex queries.

```bash
npx expo install @nozbe/watermelondb
npx expo install @nozbe/with-observables
```

**Schema definition:**
```tsx
// src/db/schema.ts
import { appSchema, tableSchema } from '@nozbe/watermelondb';

export const schema = appSchema({
  version: 1,
  tables: [
    tableSchema({
      name: 'products',
      columns: [
        { name: 'name', type: 'string' },
        { name: 'price', type: 'number' },
        { name: 'category', type: 'string' },
        { name: 'is_favorite', type: 'boolean' },
        { name: 'synced_at', type: 'number', isOptional: true },
        { name: 'created_at', type: 'number' },
        { name: 'updated_at', type: 'number' },
      ],
    }),
    tableSchema({
      name: 'orders',
      columns: [
        { name: 'product_id', type: 'string', isIndexed: true },
        { name: 'qty', type: 'number' },
        { name: 'total', type: 'number' },
        { name: 'created_at', type: 'number' },
        { name: 'updated_at', type: 'number' },
      ],
    }),
  ],
});
```

**Model definition:**
```tsx
// src/db/models/Product.ts
import { Model } from '@nozbe/watermelondb';
import { field, date, readonly, relation } from '@nozbe/watermelondb/decorators';

export class Product extends Model {
  static table = 'products';

  @field('name')       name!: string;
  @field('price')      price!: number;
  @field('category')   category!: string;
  @field('is_favorite') isFavorite!: boolean;

  @readonly @date('created_at') createdAt!: Date;
  @readonly @date('updated_at') updatedAt!: Date;
}
```

**Database setup:**
```tsx
// src/db/index.ts
import { Database } from '@nozbe/watermelondb';
import SQLiteAdapter from '@nozbe/watermelondb/adapters/sqlite';
import { schema } from './schema';
import { Product } from './models/Product';

const adapter = new SQLiteAdapter({
  schema,
  migrations: undefined, // add migrations when schema changes
  jsi: true,             // use JSI for faster JS-to-native bridge
  onSetUpError: (error) => console.error('DB setup failed:', error),
});

export const database = new Database({
  adapter,
  modelClasses: [Product],
});
```

**Reactive component with `withObservables`:**
```tsx
import { withObservables } from '@nozbe/with-observables';
import { Q } from '@nozbe/watermelondb';
import { database } from '@/db';

// Enhance component with observable props
const enhance = withObservables([], () => ({
  products: database.collections.get<Product>('products').query(
    Q.sortBy('name', Q.asc)
  ),
}));

function ProductListBase({ products }: { products: Product[] }) {
  return (
    <FlatList
      data={products}
      keyExtractor={p => p.id}
      renderItem={({ item }) => <ProductCard product={item} />}
    />
  );
}

// ProductList automatically re-renders when DB changes
export const ProductList = enhance(ProductListBase);
```

---

### 2.3 Realm / MongoDB Realm

Realm is an object-oriented mobile database that can sync with MongoDB Atlas.

```bash
npx expo install realm @realm/react
```

```tsx
// src/db/realmConfig.ts
import Realm, { createRealmContext } from '@realm/react';

class Product extends Realm.Object<Product> {
  _id!: Realm.BSON.ObjectId;
  name!: string;
  price!: number;
  category!: string;
  isFavorite!: boolean;

  static schema: Realm.ObjectSchema = {
    name: 'Product',
    primaryKey: '_id',
    properties: {
      _id: { type: 'objectId', default: () => new Realm.BSON.ObjectId() },
      name: 'string',
      price: 'float',
      category: 'string',
      isFavorite: { type: 'bool', default: false },
    },
  };
}

export const { RealmProvider, useRealm, useQuery, useObject } = createRealmContext({
  schema: [Product],
  schemaVersion: 1,
});
```

```tsx
// Wrap app
<RealmProvider>
  <App />
</RealmProvider>

// In component — reactive, auto-updates on change
function ProductList() {
  const products = useQuery(Product, (col) => col.sorted('name'));

  return (
    <FlatList
      data={[...products]}
      renderItem={({ item }) => <ProductCard product={item} />}
    />
  );
}

// Write — must be inside a transaction
function AddProduct() {
  const realm = useRealm();

  const add = () => {
    realm.write(() => {
      realm.create(Product, { name: 'Nike Air Max', price: 120, category: 'shoes' });
    });
  };
}

// Atlas Device Sync (cloud sync)
export const { RealmProvider } = createRealmContext({
  schema: [Product],
  sync: {
    flexible: true,
    user: app.currentUser,   // Realm App user
    onError: (_, error) => console.error(error),
  },
});
```

---

### 2.4 op-sqlite (Performance)

`op-sqlite` is the fastest SQLite binding for React Native using JSI (JavaScript Interface), bypassing the bridge entirely.

```bash
npx expo install @op-engineering/op-sqlite
```

```tsx
import { open } from '@op-engineering/op-sqlite';

const db = open({
  name: 'app.db',
  location: '...',   // optional custom path
});

// Synchronous execution — no await
const result = db.executeSync('SELECT * FROM products WHERE id = ?', ['123']);
// result.rows — array of row objects

// Async execution
const { rows } = await db.execute('SELECT * FROM products', []);

// Prepared statements — best for repeated queries
const stmt = db.prepareStatement('SELECT * FROM products WHERE category = ?');
const { rows } = stmt.execute(['shoes']);
stmt.dispose(); // free native memory

// Transactions
await db.transaction(async (tx) => {
  await tx.execute('INSERT INTO products (id, name) VALUES (?, ?)', ['1', 'Shoe']);
  await tx.execute('UPDATE inventory SET count = count - 1 WHERE id = ?', ['shoe-1']);
});

// Blob support
const blob = db.executeSync('SELECT image FROM products WHERE id = ?', ['1']);

// React hook for reactive queries
import { useLiveQuery } from '@op-engineering/op-sqlite';

function ProductList() {
  const { data: products } = useLiveQuery(
    db,
    'SELECT * FROM products ORDER BY name'
  );
  // Re-runs when the products table changes
  return <FlatList data={products} renderItem={({ item }) => <Text>{item.name}</Text>} />;
}
```

**Performance comparison:**

| Library | Read Speed | Write Speed | Reactive | Sync |
|---|---|---|---|---|
| `expo-sqlite` | Medium | Medium | Via listener | No |
| `WatermelonDB` | Fast | Fast | Built-in | Via custom |
| `Realm` | Fast | Fast | Built-in | Atlas Sync |
| `op-sqlite` | Fastest (JSI) | Fastest (JSI) | Via hook | No |

---

## 3. Offline-First Patterns

### 3.1 Cache-Then-Network

Show cached data immediately, then update from the network in the background.

```tsx
// src/hooks/useCacheThenNetwork.ts
import { useQuery } from '@tanstack/react-query';
import { productsRepo } from '@/db/productsRepository';
import { productsApi } from '@/api/productsApi';

export function useProducts() {
  return useQuery({
    queryKey: ['products'],
    queryFn: async () => {
      // 1. Fetch from network
      const fresh = await productsApi.getAll();

      // 2. Persist to local DB for next offline session
      await productsRepo.upsertMany(fresh);

      return fresh;
    },

    // 3. Use local DB as initial data while network request is in-flight
    initialData: () => {
      // This runs synchronously before the queryFn
      // Return undefined if cache is not available yet
      return undefined;
    },

    // 4. Show stale data while fetching
    staleTime: 1000 * 60 * 5, // 5 minutes before considering stale
    networkMode: 'offlineFirst', // use cache even when offline
  });
}
```

**Manual cache-then-network hook:**
```tsx
export function useCacheThenNetwork<T>(
  cacheLoader: () => Promise<T[] | null>,
  networkLoader: () => Promise<T[]>,
  onNetworkSuccess: (data: T[]) => Promise<void>
) {
  const [data, setData] = useState<T[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [isFetching, setIsFetching] = useState(false);

  useEffect(() => {
    let cancelled = false;

    const load = async () => {
      // Step 1 — show cache immediately
      const cached = await cacheLoader();
      if (cached && !cancelled) {
        setData(cached);
        setIsLoading(false);
      }

      // Step 2 — fetch fresh data in background
      setIsFetching(true);
      try {
        const fresh = await networkLoader();
        if (!cancelled) {
          setData(fresh);
          await onNetworkSuccess(fresh); // persist to DB
        }
      } catch {
        // Network failed — keep showing cached data
      } finally {
        if (!cancelled) {
          setIsLoading(false);
          setIsFetching(false);
        }
      }
    };

    load();
    return () => { cancelled = true; };
  }, []);

  return { data, isLoading, isFetching };
}

// Usage
const { data, isLoading, isFetching } = useCacheThenNetwork(
  () => productsRepo.getAll(),
  () => productsApi.getAll(),
  (fresh) => productsRepo.upsertMany(fresh)
);
```

---

### 3.2 Optimistic UI Updates

Update the UI immediately before the server confirms — roll back on failure.

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { productsRepo } from '@/db/productsRepository';

export function useToggleFavorite() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (productId: string) => productsApi.toggleFavorite(productId),

    onMutate: async (productId) => {
      // 1. Cancel in-flight refetches
      await queryClient.cancelQueries({ queryKey: ['products'] });

      // 2. Snapshot current cache
      const previous = queryClient.getQueryData<Product[]>(['products']);

      // 3. Optimistically update the cache
      queryClient.setQueryData<Product[]>(['products'], (old = []) =>
        old.map(p =>
          p.id === productId ? { ...p, isFavorite: !p.isFavorite } : p
        )
      );

      // 4. Also update local DB optimistically
      const product = await productsRepo.getById(productId);
      if (product) {
        await productsRepo.upsert({ ...product, isFavorite: !product.isFavorite });
      }

      return { previous };
    },

    onError: (error, productId, context) => {
      // 5. Roll back cache
      if (context?.previous) {
        queryClient.setQueryData(['products'], context.previous);
      }
      // Roll back DB
      productsRepo.getById(productId).then(p => {
        if (p) productsRepo.upsert({ ...p, isFavorite: !p.isFavorite });
      });
    },

    onSettled: () => {
      // 6. Always re-sync with server after
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },
  });
}
```

---

### 3.3 Queue-Based Sync

When offline, queue mutations locally and replay them when connectivity returns.

```tsx
// src/lib/syncQueue.ts
import { MMKV } from 'react-native-mmkv';

type QueuedOperation = {
  id: string;
  type: 'CREATE' | 'UPDATE' | 'DELETE';
  entity: string;
  payload: unknown;
  timestamp: number;
  retryCount: number;
};

const queueStorage = new MMKV({ id: 'sync-queue' });
const QUEUE_KEY = 'pending-operations';

export const syncQueue = {
  getAll: (): QueuedOperation[] => {
    const raw = queueStorage.getString(QUEUE_KEY);
    return raw ? JSON.parse(raw) : [];
  },

  enqueue: (op: Omit<QueuedOperation, 'id' | 'timestamp' | 'retryCount'>) => {
    const ops = syncQueue.getAll();
    ops.push({
      ...op,
      id: `${Date.now()}-${Math.random().toString(36).slice(2)}`,
      timestamp: Date.now(),
      retryCount: 0,
    });
    queueStorage.set(QUEUE_KEY, JSON.stringify(ops));
  },

  remove: (id: string) => {
    const ops = syncQueue.getAll().filter(op => op.id !== id);
    queueStorage.set(QUEUE_KEY, JSON.stringify(ops));
  },

  incrementRetry: (id: string) => {
    const ops = syncQueue.getAll().map(op =>
      op.id === id ? { ...op, retryCount: op.retryCount + 1 } : op
    );
    queueStorage.set(QUEUE_KEY, JSON.stringify(ops));
  },

  clear: () => queueStorage.delete(QUEUE_KEY),
};

// ─── Sync processor ────────────────────────────────────────────────────────
const MAX_RETRIES = 3;

export async function processSyncQueue(): Promise<void> {
  const ops = syncQueue.getAll();
  if (ops.length === 0) return;

  console.log(`Processing ${ops.length} queued operations`);

  for (const op of ops) {
    if (op.retryCount >= MAX_RETRIES) {
      console.error(`Operation ${op.id} exceeded max retries — dropping`);
      syncQueue.remove(op.id);
      continue;
    }

    try {
      await executeOperation(op);
      syncQueue.remove(op.id);
    } catch (error) {
      console.warn(`Operation ${op.id} failed (attempt ${op.retryCount + 1})`);
      syncQueue.incrementRetry(op.id);
      // Don't break — try remaining operations
    }
  }
}

async function executeOperation(op: QueuedOperation): Promise<void> {
  switch (`${op.entity}.${op.type}`) {
    case 'products.CREATE': return productsApi.create(op.payload as any);
    case 'products.UPDATE': return productsApi.update((op.payload as any).id, op.payload as any);
    case 'products.DELETE': return productsApi.delete(op.payload as string);
    case 'orders.CREATE':   return ordersApi.create(op.payload as any);
    // add more entity handlers
  }
}
```

**Trigger sync on network recovery:**
```tsx
import NetInfo from '@react-native-community/netinfo';
import { useEffect } from 'react';

export function useSyncOnReconnect() {
  useEffect(() => {
    let wasOffline = false;

    const unsubscribe = NetInfo.addEventListener(async (state) => {
      const isOnline = state.isConnected ?? false;

      if (wasOffline && isOnline) {
        // Just reconnected — drain the queue
        await processSyncQueue();
      }

      wasOffline = !isOnline;
    });

    return unsubscribe;
  }, []);
}
```

**Using the queue in mutations:**
```tsx
async function createProduct(product: CreateProduct) {
  const isOnline = (await NetInfo.fetch()).isConnected;

  if (isOnline) {
    // Direct API call
    return productsApi.create(product);
  } else {
    // Queue for later — also write to local DB immediately
    const tempId = `temp-${Date.now()}`;
    await productsRepo.upsert({ ...product, id: tempId, isSynced: false });
    syncQueue.enqueue({ type: 'CREATE', entity: 'products', payload: product });
    return { ...product, id: tempId };
  }
}
```

---

### 3.4 Conflict Resolution

When the same record is modified both offline (locally) and on the server, you need a strategy to resolve the conflict.

**Strategy 1 — Last Write Wins (simple):**
```tsx
// Compare updatedAt timestamps — server wins if newer
async function resolveProductConflict(local: Product, server: Product): Promise<Product> {
  return local.updatedAt > server.updatedAt ? local : server;
}
```

**Strategy 2 — Server Wins (simplest, suitable for most apps):**
```tsx
async function syncProducts() {
  const serverProducts = await productsApi.getAll();
  // Server data always overwrites local — no conflict logic needed
  await productsRepo.upsertMany(serverProducts);
}
```

**Strategy 3 — Field-level merge (complex, highest fidelity):**
```tsx
type VersionedProduct = Product & {
  serverUpdatedAt: number;
  localUpdatedAt: number;
  localChanges: Partial<Product>; // which fields were changed locally
};

async function mergeProduct(local: VersionedProduct, server: Product): Promise<Product> {
  const merged = { ...server }; // start with server version

  // Apply local changes only if they happened AFTER the last sync
  for (const [field, value] of Object.entries(local.localChanges)) {
    const localChangeTime = local.localUpdatedAt;
    const serverChangeTime = server.updatedAt;

    if (localChangeTime > serverChangeTime) {
      // Local change is newer — keep it
      (merged as any)[field] = value;
    }
    // else: server change is newer — server value already in merged
  }

  return merged;
}
```

**Strategy 4 — Operational Transform / CRDT (for collaborative apps):**
```
Use Yjs or Automerge for true conflict-free merging.
These are heavy libraries — only justify them for collaborative editors.
```

---

### 3.5 Network State Detection

```bash
npx expo install @react-native-community/netinfo
```

```tsx
// src/hooks/useNetworkStatus.ts
import NetInfo, { NetInfoState } from '@react-native-community/netinfo';
import { useState, useEffect, useCallback } from 'react';

export type NetworkStatus = {
  isOnline: boolean;
  isWifi: boolean;
  isCellular: boolean;
  isExpensive: boolean;   // true for cellular — good for deciding whether to sync
  type: string | null;
  wasOffline: boolean;    // true on the first render after coming back online
};

export function useNetworkStatus(): NetworkStatus {
  const [state, setState] = useState<NetInfoState | null>(null);
  const [wasOffline, setWasOffline] = useState(false);

  useEffect(() => {
    // Initialize with current state
    NetInfo.fetch().then(setState);

    const unsubscribe = NetInfo.addEventListener((newState) => {
      setState(prev => {
        // Detect reconnection
        if (prev && !prev.isConnected && newState.isConnected) {
          setWasOffline(true);
          setTimeout(() => setWasOffline(false), 100); // reset after one render
        }
        return newState;
      });
    });

    return unsubscribe;
  }, []);

  return {
    isOnline: state?.isConnected ?? true,
    isWifi: state?.type === 'wifi',
    isCellular: state?.type === 'cellular',
    isExpensive: state?.details?.isConnectionExpensive ?? false,
    type: state?.type ?? null,
    wasOffline,
  };
}

// Offline banner component
export function OfflineBanner() {
  const { isOnline, wasOffline } = useNetworkStatus();

  if (isOnline && !wasOffline) return null;

  return (
    <View style={{
      backgroundColor: isOnline ? '#22c55e' : '#ef4444',
      padding: 8,
      alignItems: 'center',
    }}>
      <Text style={{ color: '#fff', fontWeight: '600' }}>
        {isOnline ? 'Back Online' : 'No Internet Connection'}
      </Text>
    </View>
  );
}
```

---

## 4. Data Synchronization

### 4.1 Last-Write-Wins

The simplest conflict strategy — the record with the latest timestamp wins.

```tsx
// src/db/syncStrategies.ts

type SyncRecord = {
  id: string;
  updatedAt: number; // Unix timestamp in ms
  [key: string]: unknown;
};

export function lastWriteWins<T extends SyncRecord>(local: T, server: T): T {
  return local.updatedAt > server.updatedAt ? local : server;
}

// Apply to a batch sync
async function syncWithLWW(serverRecords: SyncRecord[]) {
  for (const serverRecord of serverRecords) {
    const local = await productsRepo.getById(serverRecord.id);

    if (!local) {
      // New record from server — just insert
      await productsRepo.upsert(serverRecord as Product);
    } else {
      // Conflict — apply LWW
      const winner = lastWriteWins(local as SyncRecord, serverRecord);
      await productsRepo.upsert(winner as Product);
    }
  }
}
```

---

### 4.2 Timestamp-Based Sync

Only sync records that changed since the last sync — efficient for large datasets.

```tsx
// src/lib/syncService.ts
import { MMKV } from 'react-native-mmkv';
import { productsRepo } from '@/db/productsRepository';
import { productsApi } from '@/api/productsApi';

const syncStorage = new MMKV({ id: 'sync-state' });

export const syncService = {
  getLastSyncTime: (entity: string): number => {
    return syncStorage.getNumber(`lastSync.${entity}`) ?? 0;
  },

  setLastSyncTime: (entity: string, timestamp: number) => {
    syncStorage.set(`lastSync.${entity}`, timestamp);
  },

  syncProducts: async () => {
    const lastSync = syncService.getLastSyncTime('products');
    const syncStarted = Date.now();

    // Ask server for only records changed since lastSync
    const { updated, deleted } = await productsApi.getDelta(lastSync);

    // Apply updates
    if (updated.length > 0) {
      await productsRepo.upsertMany(updated);
    }

    // Apply deletions
    for (const id of deleted) {
      await productsRepo.delete(id);
    }

    // Mark sync complete
    syncService.setLastSyncTime('products', syncStarted);

    return { updated: updated.length, deleted: deleted.length };
  },
};

// Server endpoint should support:
// GET /products/delta?since=1713996400000
// → { updated: [...], deleted: ['id1', 'id2'] }
```

---

### 4.3 Delta Sync

Only transfer what changed — saves bandwidth on metered connections.

```tsx
// src/api/productsApi.ts — delta sync endpoint
export const productsApi = {
  getDelta: async (since: number): Promise<{ updated: Product[]; deleted: string[] }> => {
    const { data } = await apiClient.get('/products/delta', {
      params: { since: Math.floor(since / 1000) }, // convert to Unix seconds
    });
    return data;
  },
};

// Full sync service with delta awareness
export class DeltaSyncManager {
  private readonly FULL_SYNC_INTERVAL = 24 * 60 * 60 * 1000; // 24 hours

  async sync(entity: string): Promise<void> {
    const lastSync = syncService.getLastSyncTime(entity);
    const elapsed = Date.now() - lastSync;

    if (lastSync === 0 || elapsed > this.FULL_SYNC_INTERVAL) {
      // First sync or been too long — do a full sync
      await this.fullSync(entity);
    } else {
      // Incremental delta sync
      await this.deltaSync(entity, lastSync);
    }
  }

  private async fullSync(entity: string) {
    const all = await productsApi.getAll();
    await productsRepo.upsertMany(all);
    syncService.setLastSyncTime(entity, Date.now());
  }

  private async deltaSync(entity: string, since: number) {
    const { updated, deleted } = await productsApi.getDelta(since);
    await productsRepo.upsertMany(updated);
    for (const id of deleted) await productsRepo.delete(id);
    syncService.setLastSyncTime(entity, Date.now());
  }
}

export const syncManager = new DeltaSyncManager();
```

---

### 4.4 Background Sync

Sync data while the app is in the background using Expo's background tasks.

```bash
npx expo install expo-background-fetch expo-task-manager
```

```tsx
// src/tasks/syncTask.ts
import * as BackgroundFetch from 'expo-background-fetch';
import * as TaskManager from 'expo-task-manager';
import { syncManager } from '@/lib/syncService';

const SYNC_TASK = 'background-sync';

// Define the task — must be called at module level, not inside a component
TaskManager.defineTask(SYNC_TASK, async () => {
  try {
    const network = await NetInfo.fetch();
    if (!network.isConnected) {
      return BackgroundFetch.BackgroundFetchResult.NoData;
    }

    // Don't sync over expensive cellular connections (optional)
    if (network.details?.isConnectionExpensive) {
      return BackgroundFetch.BackgroundFetchResult.NoData;
    }

    await syncManager.sync('products');
    await syncManager.sync('orders');
    await processSyncQueue(); // flush any queued offline mutations

    return BackgroundFetch.BackgroundFetchResult.NewData;
  } catch (error) {
    console.error('[BackgroundSync] Failed:', error);
    return BackgroundFetch.BackgroundFetchResult.Failed;
  }
});

// Register the task (call once, e.g., after login)
export async function registerBackgroundSync() {
  const status = await BackgroundFetch.getStatusAsync();
  if (status === BackgroundFetch.BackgroundFetchStatus.Restricted ||
      status === BackgroundFetch.BackgroundFetchStatus.Denied) {
    console.warn('Background fetch is not available');
    return;
  }

  await BackgroundFetch.registerTaskAsync(SYNC_TASK, {
    minimumInterval: 15 * 60,  // minimum 15 minutes (iOS enforces this)
    stopOnTerminate: false,    // continue after app is killed
    startOnBoot: true,         // run after device reboots
  });
}

// Unregister (e.g., on logout)
export async function unregisterBackgroundSync() {
  await BackgroundFetch.unregisterTaskAsync(SYNC_TASK);
}
```

---

### 4.5 TanStack Query Persistence

Persist TanStack Query's cache to AsyncStorage so data is available on next app launch.

```bash
npx expo install @tanstack/query-async-storage-persister @tanstack/react-query-persist-client
```

```tsx
// src/lib/queryClient.ts
import { QueryClient } from '@tanstack/react-query';
import { createAsyncStoragePersister } from '@tanstack/query-async-storage-persister';
import { PersistQueryClientProvider } from '@tanstack/react-query-persist-client';
import AsyncStorage from '@react-native-async-storage/async-storage';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      gcTime: 1000 * 60 * 60 * 24, // keep cache for 24 hours
      staleTime: 1000 * 60 * 5,    // fresh for 5 minutes
      networkMode: 'offlineFirst',
    },
  },
});

export const asyncStoragePersister = createAsyncStoragePersister({
  storage: AsyncStorage,
  key: 'REACT_QUERY_OFFLINE_CACHE',
  throttleTime: 1000,  // debounce writes by 1s (avoid thrashing)
  serialize: JSON.stringify,
  deserialize: JSON.parse,
});
```

```tsx
// app/_layout.tsx
import { PersistQueryClientProvider } from '@tanstack/react-query-persist-client';
import { queryClient, asyncStoragePersister } from '@/lib/queryClient';

export default function RootLayout() {
  return (
    <PersistQueryClientProvider
      client={queryClient}
      persistOptions={{
        persister: asyncStoragePersister,
        maxAge: 1000 * 60 * 60 * 24, // cache valid for 24 hours
        buster: APP_VERSION,          // invalidate cache on new app version
      }}
      onSuccess={() => {
        // Cache has been restored — resume any paused mutations
        queryClient.resumePausedMutations().then(() => {
          queryClient.invalidateQueries();
        });
      }}
    >
      <Stack />
    </PersistQueryClientProvider>
  );
}
```

**Persisted mutations** — mutations queued while offline automatically replay on reconnect:
```tsx
const { mutate: createProduct } = useMutation({
  mutationFn: productsApi.create,
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['products'] }),
  // Mutations are paused when offline and replayed when online
  // (PersistQueryClientProvider handles this via onSuccess → resumePausedMutations)
});
```

---

## 5. Cache Management

### 5.1 Image Caching (expo-image)

`expo-image` is a drop-in `Image` replacement with aggressive caching, blurhash placeholders, and priority loading.

```bash
npx expo install expo-image
```

```tsx
import { Image } from 'expo-image';

// Basic usage — cached automatically
<Image
  source={{ uri: 'https://example.com/photo.jpg' }}
  style={{ width: 200, height: 200 }}
  contentFit="cover"
  placeholder={{ blurhash: 'L6PZfSi_.AyE_3t7t7R**0o#DgR4' }}
  transition={300}  // ms fade-in after load
/>
```

**Caching policies:**
```tsx
<Image
  source={{ uri: imageUrl }}
  cachePolicy="memory-disk"   // default — memory first, then disk
/>

// Policy options:
// 'none'          — no caching
// 'disk'          — disk only
// 'memory'        — memory only (cleared when app closes)
// 'memory-disk'   — memory first, fallback to disk (default)
```

**Priority loading (critical above-the-fold images):**
```tsx
<Image
  source={{ uri: heroImageUrl }}
  priority="high"    // 'low' | 'normal' | 'high'
  style={{ width: '100%', height: 300 }}
/>
```

**Prefetching images before they appear:**
```tsx
import { Image } from 'expo-image';

// Prefetch a list of images in the background
async function prefetchProductImages(products: Product[]) {
  await Image.prefetch(
    products.map(p => p.imageUrl).filter(Boolean) as string[]
  );
}

// Prefetch with priority
await Image.prefetch(['url1', 'url2'], { cachePolicy: 'disk' });
```

**Clear image cache:**
```tsx
// Clear all cached images
await Image.clearDiskCache();
await Image.clearMemoryCache();

// Check if a URL is cached
const isCached = await Image.getCachePathAsync('https://example.com/photo.jpg');
```

---

### 5.2 API Response Caching

Layer 1: TanStack Query in-memory cache → Layer 2: Persisted cache → Layer 3: SQLite

```tsx
// src/hooks/useProducts.ts — three-layer caching
import { useQuery } from '@tanstack/react-query';
import { productsRepo } from '@/db/productsRepository';
import { productsApi } from '@/api/productsApi';

export function useProducts() {
  return useQuery({
    queryKey: ['products'],

    queryFn: async ({ signal }) => {
      try {
        // Layer 1: network (with abort support)
        const fresh = await productsApi.getAll({ signal });
        // Layer 3: persist to SQLite for offline use
        await productsRepo.upsertMany(fresh);
        return fresh;
      } catch (error) {
        // Network failed — try SQLite
        const cached = await productsRepo.getAll();
        if (cached.length > 0) return cached;
        throw error; // nothing available
      }
    },

    // Layer 2: TanStack Query persisted cache (from 4.5)
    staleTime: 1000 * 60 * 5,
    gcTime: 1000 * 60 * 60 * 24,
    networkMode: 'offlineFirst',

    // Placeholder from SQLite while query runs
    placeholderData: () => {
      // Note: this is synchronous — only works with MMKV, not SQLite
      return undefined;
    },
  });
}
```

**HTTP cache headers — respect server cache directives:**
```tsx
// If your API sends Cache-Control headers, respect them in your staleTime
apiClient.interceptors.response.use(response => {
  const cacheControl = response.headers['cache-control'];
  const maxAge = cacheControl?.match(/max-age=(\d+)/)?.[1];

  if (maxAge) {
    // Store alongside the data for TanStack Query staleTime
    response.data._cacheMaxAge = parseInt(maxAge) * 1000;
  }

  return response;
});
```

---

### 5.3 Cache Invalidation

Know exactly when to invalidate — over-invalidating defeats the purpose of caching.

```tsx
// src/lib/cacheManager.ts
import { queryClient } from './queryClient';
import { productsRepo } from '@/db/productsRepository';
import { Image } from 'expo-image';

export const cacheManager = {
  // Invalidate after a mutation
  afterProductMutation: async (productId?: string) => {
    if (productId) {
      // Targeted — only invalidate the specific product
      queryClient.invalidateQueries({ queryKey: ['products', productId] });
    }
    // Also invalidate the list
    queryClient.invalidateQueries({ queryKey: ['products'] });
  },

  // Invalidate stale data (older than a threshold)
  invalidateStaleQueries: async (maxAgeMs: number) => {
    const cache = queryClient.getQueryCache();
    cache.getAll().forEach(query => {
      const age = Date.now() - (query.state.dataUpdatedAt ?? 0);
      if (age > maxAgeMs) {
        queryClient.invalidateQueries({ queryKey: query.queryKey });
      }
    });
  },

  // Full reset — on logout or major data change
  clearAll: async () => {
    queryClient.clear();                    // TanStack Query memory cache
    await Image.clearDiskCache();           // Image disk cache
    await Image.clearMemoryCache();         // Image memory cache
    // SQLite data kept intentionally for offline use
    // Clear it only if needed: await productsRepo.deleteAll()
  },

  // Cache stats
  getStats: () => {
    const queries = queryClient.getQueryCache().getAll();
    return {
      totalQueries: queries.length,
      staleCount: queries.filter(q => q.isStale()).length,
      fetchingCount: queries.filter(q => q.state.status === 'pending').length,
    };
  },
};
```

**Invalidation strategies by trigger:**
```tsx
// After login — don't invalidate (user likely wants their cached data)

// After logout — clear everything
const handleLogout = async () => {
  await cacheManager.clearAll();
  router.replace('/(auth)/login');
};

// After pull-to-refresh — invalidate specific screen's queries
const onRefresh = useCallback(() => {
  queryClient.invalidateQueries({ queryKey: ['products'] });
}, []);

// After background sync — don't auto-invalidate, let staleTime handle it
// unless you know data changed: queryClient.invalidateQueries(...)

// App version bump — buster string in PersistQueryClientProvider handles it
```

---

### 5.4 Storage Limits and Cleanup

Mobile devices have limited storage. Implement a cleanup strategy.

```tsx
// src/lib/storageManager.ts
import * as FileSystem from 'expo-file-system';
import { Image } from 'expo-image';
import { productsRepo } from '@/db/productsRepository';

const LIMITS = {
  cacheMaxSizeBytes: 100 * 1024 * 1024, // 100 MB
  cacheMaxAgeMs: 7 * 24 * 60 * 60 * 1000, // 7 days
  dbMaxRecords: 10_000,                 // max rows per table
};

export const storageManager = {
  // Get current storage usage
  getStorageInfo: async () => {
    const info = await FileSystem.getFreeDiskStorageAsync();
    const total = await FileSystem.getTotalDiskCapacityAsync();
    const cacheDir = FileSystem.cacheDirectory!;
    const files = await FileSystem.readDirectoryAsync(cacheDir).catch(() => []);

    let cacheSize = 0;
    for (const file of files) {
      const fileInfo = await FileSystem.getInfoAsync(cacheDir + file);
      if (fileInfo.exists && 'size' in fileInfo) cacheSize += fileInfo.size;
    }

    return {
      freeDisk: info,
      totalDisk: total,
      usedPercent: Math.round(((total - info) / total) * 100),
      cacheSize,
    };
  },

  // Clean up old cache files
  pruneCache: async () => {
    const cacheDir = FileSystem.cacheDirectory!;
    const files = await FileSystem.readDirectoryAsync(cacheDir).catch(() => []);
    const now = Date.now();
    let freedBytes = 0;

    for (const file of files) {
      const path = cacheDir + file;
      const info = await FileSystem.getInfoAsync(path);
      if (!info.exists) continue;

      const age = now - info.modificationTime * 1000;
      if (age > LIMITS.cacheMaxAgeMs) {
        const size = 'size' in info ? info.size : 0;
        await FileSystem.deleteAsync(path, { idempotent: true });
        freedBytes += size;
      }
    }

    return freedBytes;
  },

  // Prune old DB records
  pruneDatabase: async () => {
    const count = await productsRepo.count();
    if (count > LIMITS.dbMaxRecords) {
      // Delete oldest records beyond limit
      const db = await getDatabase();
      await db.runAsync(`
        DELETE FROM products WHERE id IN (
          SELECT id FROM products
          ORDER BY createdAt ASC
          LIMIT ?
        )
      `, [count - LIMITS.dbMaxRecords]);
    }
  },

  // Run full cleanup
  runCleanup: async () => {
    const { cacheSize } = await storageManager.getStorageInfo();

    if (cacheSize > LIMITS.cacheMaxSizeBytes) {
      const freed = await storageManager.pruneCache();
      await Image.clearDiskCache();
      console.log(`Freed ${Math.round(freed / 1024 / 1024)} MB of cache`);
    }

    await storageManager.pruneDatabase();
  },
};

// Run cleanup on app launch (not blocking)
export function useStorageCleanup() {
  useEffect(() => {
    // Run in background after app is fully loaded
    const timer = setTimeout(() => {
      storageManager.runCleanup().catch(console.error);
    }, 5000); // wait 5s after launch

    return () => clearTimeout(timer);
  }, []);
}
```

**App size monitoring:**
```tsx
import * as Application from 'expo-application';

async function logStorageUsage() {
  const info = await storageManager.getStorageInfo();

  // Warn if device is critically low
  if (info.usedPercent > 90) {
    Alert.alert(
      'Storage Almost Full',
      'Your device storage is almost full. Some features may not work correctly.',
      [
        { text: 'Ignore' },
        { text: 'Free Up Space', onPress: storageManager.runCleanup },
      ]
    );
  }
}
```

---

*End of Module 10*
