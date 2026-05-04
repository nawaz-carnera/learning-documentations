# Module 5 — State Management

> State management in React Native follows the same mental model as React JS. This module covers local state, global client state (Zustand), server state (TanStack Query), and how to decide which tool belongs where.

---

## Table of Contents

1. [Local State](#1-local-state)
   - 1.1 [useState Patterns](#11-usestate-patterns)
   - 1.2 [useReducer for Complex State](#12-usereducer-for-complex-state)
   - 1.3 [Context API with useContext](#13-context-api-with-usecontext)

2. [Global State — Zustand](#2-global-state--zustand-recommended)
   - 2.1 [Store Creation](#21-store-creation)
   - 2.2 [Selectors and Shallow Comparison](#22-selectors-and-shallow-comparison)
   - 2.3 [Slices Pattern](#23-slices-pattern)
   - 2.4 [Persist Middleware](#24-persist-middleware)
   - 2.5 [DevTools Middleware](#25-devtools-middleware)

3. [Global State — Redux Toolkit](#3-global-state--redux-toolkit-if-required)
   - 3.1 [Store Setup](#31-store-setup)
   - 3.2 [createSlice](#32-createslice)
   - 3.3 [createAsyncThunk](#33-createasyncthunk)
   - 3.4 [RTK Query Basics](#34-rtk-query-basics)
   - 3.5 [Redux Persist](#35-redux-persist)
   - 3.6 [useSelector and useDispatch](#36-useselector-and-usedispatch)

4. [Server State — TanStack Query](#4-server-state--tanstack-query)
   - 4.1 [QueryClient Setup](#41-queryclient-setup)
   - 4.2 [useQuery](#42-usequery)
   - 4.3 [useMutation](#43-usemutation)
   - 4.4 [Query Keys and Caching](#44-query-keys-and-caching)
   - 4.5 [Invalidation](#45-invalidation)
   - 4.6 [Optimistic Updates](#46-optimistic-updates)
   - 4.7 [Infinite Queries](#47-infinite-queries)
   - 4.8 [Offline-First Patterns](#48-offline-first-patterns)

5. [State Architecture Decisions](#5-state-architecture-decisions)
   - 5.1 [Server State vs Client State Separation](#51-server-state-vs-client-state-separation)
   - 5.2 [When to Lift State](#52-when-to-lift-state)
   - 5.3 [Avoiding Prop Drilling](#53-avoiding-prop-drilling)
   - 5.4 [Atomic State with Jotai](#54-atomic-state-with-jotai-optional)

---

## 1. Local State

### 1.1 useState Patterns

You already know `useState` from React JS. Here are the patterns that matter in practice:

**Derived state — compute from existing state, don't duplicate:**
```tsx
const [items, setItems] = useState<Item[]>([]);

// Don't add a separate: const [count, setCount] = useState(0)
const count = items.length;           // derived — always in sync
const total = items.reduce((s, i) => s + i.price, 0); // derived
```

**Functional update — safe for async / batched updates:**
```tsx
// BAD — stale closure risk if setCount is called multiple times quickly
setCount(count + 1);

// GOOD — always gets the latest value
setCount(prev => prev + 1);
```

**Object state — spread to update partially:**
```tsx
const [form, setForm] = useState({ name: '', email: '', age: 0 });

// Update one field without losing the others
setForm(prev => ({ ...prev, email: 'nawaz@example.com' }));
```

**Lazy initialization — run expensive setup only once:**
```tsx
// This function runs on every render (bad)
const [data, setData] = useState(parseHeavyData());

// This function runs only on mount (good)
const [data, setData] = useState(() => parseHeavyData());
```

**Toggling boolean state:**
```tsx
const [isVisible, setIsVisible] = useState(false);

// Clean toggle — no need to read current value
<Pressable onPress={() => setIsVisible(prev => !prev)}>
```

---

### 1.2 useReducer for Complex State

Use `useReducer` when:
- Multiple pieces of state update together
- Next state depends on previous state in non-trivial ways
- You have many `setState` calls that are hard to reason about

```tsx
import { useReducer } from 'react';
import { View, TextInput, Pressable, Text, ActivityIndicator } from 'react-native';

type State = {
  name: string;
  email: string;
  isSubmitting: boolean;
  error: string | null;
};

type Action =
  | { type: 'SET_FIELD'; field: keyof Pick<State, 'name' | 'email'>; value: string }
  | { type: 'SUBMIT_START' }
  | { type: 'SUBMIT_SUCCESS' }
  | { type: 'SUBMIT_ERROR'; message: string };

const initialState: State = {
  name: '',
  email: '',
  isSubmitting: false,
  error: null,
};

function formReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'SET_FIELD':
      return { ...state, [action.field]: action.value, error: null };
    case 'SUBMIT_START':
      return { ...state, isSubmitting: true, error: null };
    case 'SUBMIT_SUCCESS':
      return initialState; // reset form on success
    case 'SUBMIT_ERROR':
      return { ...state, isSubmitting: false, error: action.message };
  }
}

export default function SignupForm() {
  const [state, dispatch] = useReducer(formReducer, initialState);

  const handleSubmit = async () => {
    dispatch({ type: 'SUBMIT_START' });
    try {
      await api.signup({ name: state.name, email: state.email });
      dispatch({ type: 'SUBMIT_SUCCESS' });
    } catch (e) {
      dispatch({ type: 'SUBMIT_ERROR', message: 'Signup failed. Try again.' });
    }
  };

  return (
    <View>
      <TextInput
        value={state.name}
        onChangeText={v => dispatch({ type: 'SET_FIELD', field: 'name', value: v })}
        placeholder="Name"
      />
      <TextInput
        value={state.email}
        onChangeText={v => dispatch({ type: 'SET_FIELD', field: 'email', value: v })}
        placeholder="Email"
      />
      {state.error && <Text style={{ color: 'red' }}>{state.error}</Text>}
      <Pressable onPress={handleSubmit} disabled={state.isSubmitting}>
        {state.isSubmitting ? <ActivityIndicator /> : <Text>Sign Up</Text>}
      </Pressable>
    </View>
  );
}
```

---

### 1.3 Context API with useContext

Best for low-frequency global data: theme, auth user, locale, feature flags.

**Step 1 — Create context with a custom hook:**
```tsx
// src/context/AuthContext.tsx
import { createContext, useContext, useState, useEffect } from 'react';

type User = { id: string; name: string; email: string };

type AuthContextType = {
  user: User | null;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
};

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    // Rehydrate session on app launch
    restoreSession().then(setUser).finally(() => setIsLoading(false));
  }, []);

  const login = async (email: string, password: string) => {
    const u = await api.login(email, password);
    setUser(u);
  };

  const logout = () => {
    api.logout();
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, isLoading, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

// Custom hook — throws if used outside the provider
export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used within AuthProvider');
  return ctx;
}
```

**Step 2 — Add provider to root layout:**
```tsx
// app/_layout.tsx
import { AuthProvider } from '@/context/AuthContext';

export default function RootLayout() {
  return (
    <AuthProvider>
      <Stack />
    </AuthProvider>
  );
}
```

**Step 3 — Consume anywhere:**
```tsx
import { useAuth } from '@/context/AuthContext';

export default function ProfileScreen() {
  const { user, logout } = useAuth();
  return <Text>Welcome, {user?.name}</Text>;
}
```

> **Context re-renders every consumer when the value changes.** For high-frequency state (form inputs, animations, lists), use Zustand or split contexts into smaller pieces.

---

## 2. Global State — Zustand (Recommended)

Zustand is a minimal, unopinionated state library. No providers, no boilerplate.

```bash
npx expo install zustand
```

### 2.1 Store Creation

```tsx
// src/store/useCartStore.ts
import { create } from 'zustand';

type CartItem = { id: string; name: string; price: number; qty: number };

type CartStore = {
  items: CartItem[];
  addItem: (item: Omit<CartItem, 'qty'>) => void;
  removeItem: (id: string) => void;
  updateQty: (id: string, qty: number) => void;
  clearCart: () => void;
  // Derived — computed getter
  total: () => number;
  itemCount: () => number;
};

export const useCartStore = create<CartStore>((set, get) => ({
  items: [],

  addItem: (item) =>
    set((state) => {
      const existing = state.items.find(i => i.id === item.id);
      if (existing) {
        return {
          items: state.items.map(i =>
            i.id === item.id ? { ...i, qty: i.qty + 1 } : i
          ),
        };
      }
      return { items: [...state.items, { ...item, qty: 1 }] };
    }),

  removeItem: (id) =>
    set((state) => ({ items: state.items.filter(i => i.id !== id) })),

  updateQty: (id, qty) =>
    set((state) => ({
      items: qty <= 0
        ? state.items.filter(i => i.id !== id)
        : state.items.map(i => i.id === id ? { ...i, qty } : i),
    })),

  clearCart: () => set({ items: [] }),

  total: () => get().items.reduce((sum, i) => sum + i.price * i.qty, 0),
  itemCount: () => get().items.reduce((sum, i) => sum + i.qty, 0),
}));
```

**Usage in components:**
```tsx
import { useCartStore } from '@/store/useCartStore';

export default function CartIcon() {
  // Subscribe to only the itemCount function — avoids re-render on unrelated changes
  const itemCount = useCartStore(state => state.itemCount());

  return (
    <View>
      <Ionicons name="cart" size={24} />
      {itemCount > 0 && <Text style={styles.badge}>{itemCount}</Text>}
    </View>
  );
}

export default function ProductCard({ product }: { product: Product }) {
  const addItem = useCartStore(state => state.addItem);

  return (
    <Pressable onPress={() => addItem(product)}>
      <Text>Add to Cart</Text>
    </Pressable>
  );
}
```

**Accessing state outside components (in service files, event handlers):**
```tsx
// No hook needed — call getState() directly
const total = useCartStore.getState().total();
useCartStore.getState().clearCart();
```

---

### 2.2 Selectors and Shallow Comparison

By default, a component re-renders whenever **any** part of the store changes. Use selectors to subscribe to specific slices.

```tsx
// Re-renders on every store change — BAD for large stores
const store = useCartStore();

// Re-renders only when items changes — GOOD
const items = useCartStore(state => state.items);

// Re-renders only when addItem reference changes (never, since it's stable) — GOOD
const addItem = useCartStore(state => state.addItem);
```

**Selecting multiple values — use `shallow` to avoid unnecessary re-renders:**
```tsx
import { useShallow } from 'zustand/react/shallow';

// Without shallow: re-renders whenever ANY part of store changes
// (because a new object is created each render)
const { items, total } = useCartStore(state => ({ items: state.items, total: state.total() }));

// With shallow: only re-renders when items or total actually changes
const { items, total } = useCartStore(
  useShallow(state => ({ items: state.items, total: state.total() }))
);
```

---

### 2.3 Slices Pattern

For larger apps, split your store into logical slices and combine them into one store.

```tsx
// src/store/slices/cartSlice.ts
import { StateCreator } from 'zustand';
import { RootStore } from '../useStore';

type CartItem = { id: string; name: string; price: number; qty: number };

export type CartSlice = {
  cart: {
    items: CartItem[];
    addItem: (item: Omit<CartItem, 'qty'>) => void;
    clearCart: () => void;
  };
};

export const createCartSlice: StateCreator<RootStore, [], [], CartSlice> = (set) => ({
  cart: {
    items: [],
    addItem: (item) =>
      set(state => ({
        cart: {
          ...state.cart,
          items: [...state.cart.items, { ...item, qty: 1 }],
        },
      })),
    clearCart: () => set(state => ({ cart: { ...state.cart, items: [] } })),
  },
});
```

```tsx
// src/store/slices/userSlice.ts
import { StateCreator } from 'zustand';
import { RootStore } from '../useStore';

type User = { id: string; name: string };

export type UserSlice = {
  user: {
    data: User | null;
    setUser: (user: User | null) => void;
  };
};

export const createUserSlice: StateCreator<RootStore, [], [], UserSlice> = (set) => ({
  user: {
    data: null,
    setUser: (data) => set(state => ({ user: { ...state.user, data } })),
  },
});
```

```tsx
// src/store/useStore.ts
import { create } from 'zustand';
import { CartSlice, createCartSlice } from './slices/cartSlice';
import { UserSlice, createUserSlice } from './slices/userSlice';

export type RootStore = CartSlice & UserSlice;

export const useStore = create<RootStore>((...args) => ({
  ...createCartSlice(...args),
  ...createUserSlice(...args),
}));

// Per-slice hooks for convenience
export const useCart = () => useStore(state => state.cart);
export const useUser = () => useStore(state => state.user);
```

---

### 2.4 Persist Middleware

Persist store state to storage so it survives app restarts. Uses `AsyncStorage` in React Native.

```bash
npx expo install @react-native-async-storage/async-storage
```

```tsx
// src/store/useCartStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

type CartStore = {
  items: CartItem[];
  addItem: (item: Omit<CartItem, 'qty'>) => void;
  clearCart: () => void;
};

export const useCartStore = create<CartStore>()(
  persist(
    (set) => ({
      items: [],
      addItem: (item) => set(state => ({ items: [...state.items, { ...item, qty: 1 }] })),
      clearCart: () => set({ items: [] }),
    }),
    {
      name: 'cart-storage',                          // AsyncStorage key
      storage: createJSONStorage(() => AsyncStorage), // storage adapter
      // Persist only selected fields — omit sensitive data
      partialize: (state) => ({ items: state.items }),
    }
  )
);
```

**Handling hydration** — the store is async on first load:
```tsx
import { useCartStore } from '@/store/useCartStore';

export default function App() {
  const hasHydrated = useCartStore.persist.hasHydrated();

  if (!hasHydrated) return <SplashScreen />;

  return <RootLayout />;
}
```

**Manual rehydration:**
```tsx
useEffect(() => {
  useCartStore.persist.rehydrate();
}, []);
```

---

### 2.5 DevTools Middleware

Connect Zustand to Redux DevTools browser extension for time-travel debugging.

```tsx
import { create } from 'zustand';
import { devtools } from 'zustand/middleware';

export const useCartStore = create<CartStore>()(
  devtools(
    (set) => ({
      items: [],
      addItem: (item) =>
        set(
          state => ({ items: [...state.items, { ...item, qty: 1 }] }),
          false,         // false = merge, true = replace state entirely
          'cart/addItem' // action name shown in DevTools
        ),
      clearCart: () => set({ items: [] }, false, 'cart/clearCart'),
    }),
    { name: 'CartStore' } // store name in DevTools
  )
);
```

**Combining persist + devtools:**
```tsx
export const useCartStore = create<CartStore>()(
  devtools(
    persist(
      (set) => ({ /* store impl */ }),
      { name: 'cart-storage', storage: createJSONStorage(() => AsyncStorage) }
    ),
    { name: 'CartStore' }
  )
);
```

---

## 3. Global State — Redux Toolkit (If Required)

Use Redux Toolkit when: the team already uses Redux, you need the full Redux ecosystem (Saga, RTK Query), or the app has very complex state interactions across many features.

```bash
npx expo install @reduxjs/toolkit react-redux
```

### 3.1 Store Setup

```tsx
// src/store/index.ts
import { configureStore } from '@reduxjs/toolkit';
import { cartReducer } from './slices/cartSlice';
import { userReducer } from './slices/userSlice';
import { apiSlice } from './api/apiSlice';

export const store = configureStore({
  reducer: {
    cart: cartReducer,
    user: userReducer,
    [apiSlice.reducerPath]: apiSlice.reducer, // RTK Query
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(apiSlice.middleware),
});

// Infer types from store
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

// Typed hooks — use these everywhere instead of plain useSelector/useDispatch
import { useDispatch, useSelector, TypedUseSelectorHook } from 'react-redux';
export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

**Wrap app in Provider:**
```tsx
// app/_layout.tsx
import { Provider } from 'react-redux';
import { store } from '@/store';

export default function RootLayout() {
  return (
    <Provider store={store}>
      <Stack />
    </Provider>
  );
}
```

---

### 3.2 createSlice

```tsx
// src/store/slices/cartSlice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

type CartItem = { id: string; name: string; price: number; qty: number };
type CartState = { items: CartItem[] };

const cartSlice = createSlice({
  name: 'cart',
  initialState: { items: [] } as CartState,
  reducers: {
    addItem(state, action: PayloadAction<Omit<CartItem, 'qty'>>) {
      const existing = state.items.find(i => i.id === action.payload.id);
      if (existing) {
        existing.qty += 1; // Immer allows direct mutation
      } else {
        state.items.push({ ...action.payload, qty: 1 });
      }
    },
    removeItem(state, action: PayloadAction<string>) {
      state.items = state.items.filter(i => i.id !== action.payload);
    },
    clearCart(state) {
      state.items = [];
    },
  },
});

export const { addItem, removeItem, clearCart } = cartSlice.actions;
export const cartReducer = cartSlice.reducer;

// Selectors — colocate with the slice
export const selectCartItems = (state: RootState) => state.cart.items;
export const selectCartTotal = (state: RootState) =>
  state.cart.items.reduce((sum, i) => sum + i.price * i.qty, 0);
```

---

### 3.3 createAsyncThunk

Handle async operations (API calls) with automatic `pending` / `fulfilled` / `rejected` action types.

```tsx
// src/store/slices/userSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

type User = { id: string; name: string; email: string };
type UserState = { data: User | null; isLoading: boolean; error: string | null };

// Thunk — async action creator
export const fetchUser = createAsyncThunk(
  'user/fetchUser',
  async (userId: string, { rejectWithValue }) => {
    try {
      const response = await api.getUser(userId);
      return response.data as User;
    } catch (error: any) {
      return rejectWithValue(error.message);
    }
  }
);

const userSlice = createSlice({
  name: 'user',
  initialState: { data: null, isLoading: false, error: null } as UserState,
  reducers: {
    clearUser: (state) => { state.data = null; },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchUser.pending, (state) => {
        state.isLoading = true;
        state.error = null;
      })
      .addCase(fetchUser.fulfilled, (state, action) => {
        state.isLoading = false;
        state.data = action.payload;
      })
      .addCase(fetchUser.rejected, (state, action) => {
        state.isLoading = false;
        state.error = action.payload as string;
      });
  },
});

export const { clearUser } = userSlice.actions;
export const userReducer = userSlice.reducer;
```

---

### 3.4 RTK Query Basics

RTK Query is a data-fetching and caching layer built into Redux Toolkit. It replaces manual `createAsyncThunk` for API calls.

```tsx
// src/store/api/apiSlice.ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

type Product = { id: string; name: string; price: number };

export const apiSlice = createApi({
  reducerPath: 'api',
  baseQuery: fetchBaseQuery({
    baseUrl: 'https://api.example.com',
    prepareHeaders: (headers) => {
      const token = getAuthToken();
      if (token) headers.set('Authorization', `Bearer ${token}`);
      return headers;
    },
  }),
  tagTypes: ['Product', 'Cart'],
  endpoints: (builder) => ({
    getProducts: builder.query<Product[], void>({
      query: () => '/products',
      providesTags: ['Product'],
    }),
    getProduct: builder.query<Product, string>({
      query: (id) => `/products/${id}`,
      providesTags: (result, error, id) => [{ type: 'Product', id }],
    }),
    addToCart: builder.mutation<void, string>({
      query: (productId) => ({
        url: '/cart',
        method: 'POST',
        body: { productId },
      }),
      invalidatesTags: ['Cart'],
    }),
  }),
});

// Auto-generated hooks
export const { useGetProductsQuery, useGetProductQuery, useAddToCartMutation } = apiSlice;
```

**Usage:**
```tsx
import { useGetProductsQuery, useAddToCartMutation } from '@/store/api/apiSlice';

export default function ProductList() {
  const { data: products, isLoading, error } = useGetProductsQuery();
  const [addToCart, { isLoading: isAdding }] = useAddToCartMutation();

  if (isLoading) return <ActivityIndicator />;
  if (error) return <Text>Error loading products</Text>;

  return (
    <FlatList
      data={products}
      renderItem={({ item }) => (
        <Pressable onPress={() => addToCart(item.id)}>
          <Text>{item.name}</Text>
        </Pressable>
      )}
    />
  );
}
```

---

### 3.5 Redux Persist

```bash
npx expo install redux-persist @react-native-async-storage/async-storage
```

```tsx
// src/store/index.ts
import { configureStore, combineReducers } from '@reduxjs/toolkit';
import { persistStore, persistReducer, FLUSH, REHYDRATE, PAUSE, PERSIST, PURGE, REGISTER } from 'redux-persist';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { cartReducer } from './slices/cartSlice';
import { userReducer } from './slices/userSlice';

const persistConfig = {
  key: 'root',
  storage: AsyncStorage,
  whitelist: ['cart'],  // only persist cart — user session managed separately
};

const rootReducer = combineReducers({
  cart: cartReducer,
  user: userReducer,
});

const persistedReducer = persistReducer(persistConfig, rootReducer);

export const store = configureStore({
  reducer: persistedReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        // Required: ignore redux-persist action types
        ignoredActions: [FLUSH, REHYDRATE, PAUSE, PERSIST, PURGE, REGISTER],
      },
    }),
});

export const persistor = persistStore(store);
```

```tsx
// app/_layout.tsx
import { Provider } from 'react-redux';
import { PersistGate } from 'redux-persist/integration/react';
import { store, persistor } from '@/store';

export default function RootLayout() {
  return (
    <Provider store={store}>
      <PersistGate loading={null} persistor={persistor}>
        <Stack />
      </PersistGate>
    </Provider>
  );
}
```

---

### 3.6 useSelector and useDispatch

Always use the **typed versions** created in the store setup:

```tsx
import { useAppSelector, useAppDispatch } from '@/store';
import { addItem, selectCartItems, selectCartTotal } from '@/store/slices/cartSlice';
import { fetchUser } from '@/store/slices/userSlice';

export default function CartScreen() {
  const dispatch = useAppDispatch();
  const items = useAppSelector(selectCartItems);
  const total = useAppSelector(selectCartTotal);

  useEffect(() => {
    dispatch(fetchUser('user-123')); // async thunk
  }, []);

  return (
    <View>
      {items.map(item => (
        <Text key={item.id}>{item.name} × {item.qty}</Text>
      ))}
      <Text>Total: ${total}</Text>
    </View>
  );
}
```

---

## 4. Server State — TanStack Query

TanStack Query manages async server data: fetching, caching, background refetching, loading/error states. It removes the need to manually manage `isLoading`, `error`, and `data` state for API calls.

```bash
npx expo install @tanstack/react-query
```

### 4.1 QueryClient Setup

```tsx
// src/lib/queryClient.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,    // data is fresh for 5 minutes
      gcTime: 1000 * 60 * 10,      // keep unused data in cache for 10 minutes
      retry: 2,                     // retry failed requests twice
      refetchOnWindowFocus: false,  // disable web behavior (not relevant in RN)
    },
    mutations: {
      retry: 0,                     // don't retry failed mutations
    },
  },
});
```

```tsx
// app/_layout.tsx
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from '@/lib/queryClient';

export default function RootLayout() {
  return (
    <QueryClientProvider client={queryClient}>
      <Stack />
    </QueryClientProvider>
  );
}
```

---

### 4.2 useQuery

Fetch and cache data from an API.

```tsx
// src/api/products.ts — keep API functions separate
export const productsApi = {
  getAll: async (): Promise<Product[]> => {
    const res = await fetch('https://api.example.com/products');
    if (!res.ok) throw new Error('Failed to fetch products');
    return res.json();
  },
  getById: async (id: string): Promise<Product> => {
    const res = await fetch(`https://api.example.com/products/${id}`);
    if (!res.ok) throw new Error('Product not found');
    return res.json();
  },
};
```

```tsx
// src/hooks/useProducts.ts — colocate queries with their API
import { useQuery } from '@tanstack/react-query';
import { productsApi } from '@/api/products';

export function useProducts() {
  return useQuery({
    queryKey: ['products'],
    queryFn: productsApi.getAll,
  });
}

export function useProduct(id: string) {
  return useQuery({
    queryKey: ['products', id],
    queryFn: () => productsApi.getById(id),
    enabled: !!id,   // don't run if id is empty/undefined
  });
}
```

```tsx
// Usage in a component
import { useProducts } from '@/hooks/useProducts';
import { FlatList, ActivityIndicator, Text, View } from 'react-native';

export default function ProductListScreen() {
  const { data: products, isLoading, isError, error, refetch } = useProducts();

  if (isLoading) return <ActivityIndicator style={{ flex: 1 }} />;
  if (isError) return <Text>Error: {error.message}</Text>;

  return (
    <FlatList
      data={products}
      keyExtractor={item => item.id}
      renderItem={({ item }) => <ProductCard product={item} />}
      onRefresh={refetch}            // pull-to-refresh
      refreshing={isLoading}
    />
  );
}
```

**All `useQuery` states:**
```tsx
const {
  data,          // the fetched data (undefined while loading)
  isLoading,     // true on first load with no cached data
  isFetching,    // true whenever a request is in-flight (including background refetch)
  isError,       // true if query failed
  error,         // the error object
  isSuccess,     // true if query succeeded
  refetch,       // manually trigger a refetch
  status,        // 'pending' | 'error' | 'success'
} = useQuery({ queryKey: ['products'], queryFn: productsApi.getAll });
```

---

### 4.3 useMutation

For creating, updating, or deleting data (POST / PUT / DELETE).

```tsx
// src/hooks/useProducts.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { productsApi } from '@/api/products';

export function useCreateProduct() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: productsApi.create,

    onSuccess: () => {
      // Invalidate the products list so it refetches with the new item
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },

    onError: (error) => {
      Alert.alert('Error', error.message);
    },
  });
}
```

```tsx
// Usage
import { useCreateProduct } from '@/hooks/useProducts';

export default function AddProductScreen() {
  const [name, setName] = useState('');
  const { mutate: createProduct, isPending } = useCreateProduct();

  return (
    <View>
      <TextInput value={name} onChangeText={setName} placeholder="Product name" />
      <Pressable
        onPress={() => createProduct({ name })}
        disabled={isPending}
      >
        {isPending ? <ActivityIndicator /> : <Text>Add Product</Text>}
      </Pressable>
    </View>
  );
}
```

---

### 4.4 Query Keys and Caching

Query keys are the cache keys — they uniquely identify a query. TanStack Query uses them to:
- Find existing cached data
- Determine when to refetch
- Allow targeted cache invalidation

```tsx
// Simple key
useQuery({ queryKey: ['products'], queryFn: ... })

// Key with variable — different cache entry per id
useQuery({ queryKey: ['products', id], queryFn: () => getProduct(id) })

// Key with filters — different cache entry per filter combo
useQuery({ queryKey: ['products', { category, page, sort }], queryFn: ... })
```

**Key hierarchy matters for invalidation:**
```tsx
// This invalidates ALL of these:
queryClient.invalidateQueries({ queryKey: ['products'] })
// ['products']
// ['products', '123']
// ['products', { category: 'shoes' }]

// This invalidates only the specific product:
queryClient.invalidateQueries({ queryKey: ['products', '123'] })
```

**Convention — centralize query keys:**
```tsx
// src/api/queryKeys.ts
export const queryKeys = {
  products: {
    all: ['products'] as const,
    lists: () => [...queryKeys.products.all, 'list'] as const,
    list: (filters: object) => [...queryKeys.products.lists(), filters] as const,
    details: () => [...queryKeys.products.all, 'detail'] as const,
    detail: (id: string) => [...queryKeys.products.details(), id] as const,
  },
};

// Usage
useQuery({ queryKey: queryKeys.products.detail(id), queryFn: ... })
queryClient.invalidateQueries({ queryKey: queryKeys.products.all })
```

---

### 4.5 Invalidation

Force a query to refetch by marking it stale:

```tsx
const queryClient = useQueryClient();

// After deleting a product — invalidate the list
const { mutate: deleteProduct } = useMutation({
  mutationFn: productsApi.delete,
  onSuccess: (_, deletedId) => {
    // Invalidate the products list
    queryClient.invalidateQueries({ queryKey: ['products'] });
    // Remove the specific product from cache immediately
    queryClient.removeQueries({ queryKey: ['products', deletedId] });
  },
});

// Manual invalidation — e.g., on pull-to-refresh
const handleRefresh = () => {
  queryClient.invalidateQueries({ queryKey: ['products'] });
};
```

**`invalidateQueries` vs `refetchQueries`:**

| Method | Behavior |
|---|---|
| `invalidateQueries` | Marks as stale — refetches only if component is mounted |
| `refetchQueries` | Forces immediate refetch regardless of mount state |
| `removeQueries` | Deletes from cache — next access fetches fresh |

---

### 4.6 Optimistic Updates

Update the UI immediately before the server confirms — makes the app feel instant. Rollback on error.

```tsx
const queryClient = useQueryClient();

const { mutate: toggleLike } = useMutation({
  mutationFn: (postId: string) => api.toggleLike(postId),

  onMutate: async (postId) => {
    // 1. Cancel any in-flight refetches for this query
    await queryClient.cancelQueries({ queryKey: ['posts', postId] });

    // 2. Snapshot current value for rollback
    const previousPost = queryClient.getQueryData<Post>(['posts', postId]);

    // 3. Optimistically update the cache
    queryClient.setQueryData<Post>(['posts', postId], (old) => {
      if (!old) return old;
      return { ...old, isLiked: !old.isLiked, likeCount: old.isLiked ? old.likeCount - 1 : old.likeCount + 1 };
    });

    // 4. Return snapshot — available in onError as context
    return { previousPost };
  },

  onError: (error, postId, context) => {
    // 5. Rollback to the snapshot on failure
    if (context?.previousPost) {
      queryClient.setQueryData(['posts', postId], context.previousPost);
    }
  },

  onSettled: (data, error, postId) => {
    // 6. Always refetch after success or error to sync with server
    queryClient.invalidateQueries({ queryKey: ['posts', postId] });
  },
});
```

---

### 4.7 Infinite Queries

For paginated lists with a "load more" button or infinite scroll.

```tsx
// src/hooks/useFeed.ts
import { useInfiniteQuery } from '@tanstack/react-query';

type FeedPage = { posts: Post[]; nextCursor: string | null };

export function useFeed() {
  return useInfiniteQuery({
    queryKey: ['feed'],
    queryFn: ({ pageParam }) =>
      api.getFeed({ cursor: pageParam, limit: 20 }),
    initialPageParam: null as string | null,
    getNextPageParam: (lastPage) => lastPage.nextCursor, // null = no more pages
  });
}
```

```tsx
import { useFeed } from '@/hooks/useFeed';
import { FlatList, ActivityIndicator, Pressable, Text } from 'react-native';

export default function FeedScreen() {
  const {
    data,
    isLoading,
    isFetchingNextPage,
    hasNextPage,
    fetchNextPage,
  } = useFeed();

  // Flatten pages into a single array for FlatList
  const posts = data?.pages.flatMap(page => page.posts) ?? [];

  return (
    <FlatList
      data={posts}
      keyExtractor={item => item.id}
      renderItem={({ item }) => <PostCard post={item} />}

      // Load more when user reaches the end
      onEndReached={() => {
        if (hasNextPage && !isFetchingNextPage) fetchNextPage();
      }}
      onEndReachedThreshold={0.5} // trigger when 50% from bottom

      ListFooterComponent={
        isFetchingNextPage ? <ActivityIndicator style={{ padding: 16 }} /> : null
      }
    />
  );
}
```

---

### 4.8 Offline-First Patterns

React Native apps need to handle intermittent connectivity. TanStack Query helps with its cache; pair it with `NetInfo` for a complete solution.

**Keep showing stale data when offline:**
```tsx
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,
      gcTime: 1000 * 60 * 60 * 24, // keep cache for 24 hours
      networkMode: 'offlineFirst',  // use cache even when offline
    },
  },
});
```

**`networkMode` options:**

| Value | Behavior |
|---|---|
| `'online'` (default) | Pauses queries when offline |
| `'offlineFirst'` | Always runs query — uses cache when offline |
| `'always'` | Always runs regardless of network (for non-network queries) |

**Show offline banner:**
```tsx
import NetInfo from '@react-native-community/netinfo';
import { useEffect, useState } from 'react';
import { onlineManager } from '@tanstack/react-query';

// Sync TanStack Query's online state with actual network state
export function useNetworkSync() {
  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener(state => {
      onlineManager.setOnline(state.isConnected ?? undefined);
    });
    return unsubscribe;
  }, []);
}

// Offline banner component
export function OfflineBanner() {
  const [isOnline, setIsOnline] = useState(true);

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener(state => {
      setIsOnline(state.isConnected ?? true);
    });
    return unsubscribe;
  }, []);

  if (isOnline) return null;

  return (
    <View style={{ backgroundColor: '#f00', padding: 8, alignItems: 'center' }}>
      <Text style={{ color: '#fff' }}>You are offline</Text>
    </View>
  );
}
```

**Retry on reconnect:**
```tsx
// TanStack Query automatically retries paused queries when back online.
// Ensure onlineManager is synced (see useNetworkSync above).
// No extra code needed — it's built in.
```

---

## 5. State Architecture Decisions

### 5.1 Server State vs Client State Separation

The single most important architectural rule:

| Type | Examples | Best Tool |
|---|---|---|
| **Server state** | Products list, user profile, feed, orders | TanStack Query |
| **Client UI state** | Modal open/closed, selected tab, form input, active filter | useState / useReducer |
| **Global client state** | Cart, auth session, user preferences, theme | Zustand |
| **URL/Navigation state** | Current route, route params | Expo Router |

**Anti-pattern — storing server data in Zustand:**
```tsx
// BAD — you're duplicating TanStack Query's job
const useProductStore = create((set) => ({
  products: [],
  isLoading: false,
  fetchProducts: async () => {
    set({ isLoading: true });
    const data = await api.getProducts();
    set({ products: data, isLoading: false });
  },
}));

// GOOD — TanStack Query handles server data
const { data: products, isLoading } = useQuery({
  queryKey: ['products'],
  queryFn: api.getProducts,
});
// Zustand holds only client-side state: cart, UI preferences, etc.
```

**Decision flowchart:**
```
Is the data from a server / API?
  Yes → TanStack Query
  No  → Does it need to persist across app restarts?
          Yes → Zustand + persist middleware
          No  → Does more than one screen need it?
                  Yes → Zustand (or Context for simple cases)
                  No  → useState / useReducer
```

---

### 5.2 When to Lift State

Lift state to the **closest common ancestor** of all components that need it.

```
Before lifting (broken — sibling can't read sibling's state):

  SearchBar (owns searchText)
  ProductList (needs searchText) ← can't reach SearchBar's state

After lifting (correct):

  SearchScreen (owns searchText)
  ├── SearchBar (receives searchText, onChange)
  └── ProductList (receives searchText)
```

**Signs you need to lift state:**
- Component A needs to update state that Component B reads
- Two siblings are trying to stay "in sync"
- You find yourself duplicating state across components

**Signs you've lifted too high:**
- State is in a root layout but only two nested screens use it
- Unrelated components re-render when the state changes
- Solution: drop it into a Zustand store instead — avoids re-render propagation

---

### 5.3 Avoiding Prop Drilling

Prop drilling = passing props through multiple intermediate components that don't use the data themselves.

**Identify it:**
```
App → Screen → Section → Row → Cell → Text (uses color)

If every component in the chain receives `color` just to pass it down → prop drilling.
```

**Solutions in order of preference:**

1. **Component composition** — pass the final component as a child instead of its data:
```tsx
// Instead of drilling `color` down 4 levels:
<Row color={color} />

// Pass the already-styled component:
<Row>
  <ColoredText color={color} />
</Row>
```

2. **Zustand** — for global state accessed in many unrelated components:
```tsx
const color = useThemeStore(state => state.primaryColor);
// Any component reads directly — no drilling
```

3. **Context** — for subtree-scoped state (e.g., a form's validation context):
```tsx
const FormContext = createContext(null);
// Wrap the form, all inputs read directly
```

---

### 5.4 Atomic State with Jotai (Optional)

Jotai manages state as individual **atoms** — small, independent pieces of state. Components subscribe to only the atoms they need.

```bash
npx expo install jotai
```

```tsx
// src/atoms/filterAtoms.ts
import { atom } from 'jotai';

export const searchAtom = atom('');
export const categoryAtom = atom<string | null>(null);
export const sortAtom = atom<'price' | 'rating' | 'newest'>('newest');

// Derived atom — computed from other atoms
export const activeFiltersCountAtom = atom((get) => {
  let count = 0;
  if (get(searchAtom)) count++;
  if (get(categoryAtom)) count++;
  return count;
});
```

```tsx
import { useAtom, useAtomValue, useSetAtom } from 'jotai';
import { searchAtom, categoryAtom, activeFiltersCountAtom } from '@/atoms/filterAtoms';

// Read + write
function SearchBar() {
  const [search, setSearch] = useAtom(searchAtom);
  return <TextInput value={search} onChangeText={setSearch} />;
}

// Read only
function FilterBadge() {
  const count = useAtomValue(activeFiltersCountAtom);
  return count > 0 ? <Text>{count} active filters</Text> : null;
}

// Write only (avoids re-render on read)
function ClearButton() {
  const setSearch = useSetAtom(searchAtom);
  return <Pressable onPress={() => setSearch('')}><Text>Clear</Text></Pressable>;
}
```

**When to choose Jotai over Zustand:**
- State is naturally atomic (many small independent pieces)
- Different components need different combinations of state slices
- You want React Suspense integration for async atoms
- Zustand store is growing unwieldy with many unrelated slices

---

*End of Module 5*
