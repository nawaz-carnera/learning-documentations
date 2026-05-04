# Module 6 — Networking & API Integration

> React Native runs on a JS engine with full `fetch` API support. This module covers HTTP clients, data fetching patterns with TanStack Query, error handling, runtime validation with Zod, and advanced networking concepts.

---

## Table of Contents

1. [HTTP Clients](#1-http-clients)
   - 1.1 [fetch API](#11-fetch-api)
   - 1.2 [Axios Setup and Instance](#12-axios-setup-and-instance)
   - 1.3 [Request/Response Interceptors](#13-requestresponse-interceptors)
   - 1.4 [Base URL Configuration](#14-base-url-configuration)
   - 1.5 [Timeout Handling](#15-timeout-handling)

2. [Data Fetching Patterns](#2-data-fetching-patterns)
   - 2.1 [TanStack Query Integration](#21-tanstack-query-integration)
   - 2.2 [Loading States](#22-loading-states)
   - 2.3 [Error States](#23-error-states)
   - 2.4 [Empty States](#24-empty-states)
   - 2.5 [Pagination](#25-pagination)
   - 2.6 [Infinite Scroll](#26-infinite-scroll)

3. [Error Handling](#3-error-handling)
   - 3.1 [Try-Catch Patterns](#31-try-catch-patterns)
   - 3.2 [HTTP Status Code Handling](#32-http-status-code-handling)
   - 3.3 [Network Error Detection](#33-network-error-detection)
   - 3.4 [Retry Logic](#34-retry-logic)
   - 3.5 [Error Boundaries](#35-error-boundaries)
   - 3.6 [Global Error Handlers](#36-global-error-handlers)
   - 3.7 [User-Friendly Error Messages](#37-user-friendly-error-messages)

4. [Data Validation](#4-data-validation)
   - 4.1 [Zod Schemas for API Responses](#41-zod-schemas-for-api-responses)
   - 4.2 [Runtime Type Checking](#42-runtime-type-checking)
   - 4.3 [TypeScript Types from Zod](#43-typescript-types-from-zod)

5. [Advanced Concepts](#5-advanced-concepts)
   - 5.1 [Request Cancellation (AbortController)](#51-request-cancellation-abortcontroller)
   - 5.2 [Debouncing and Throttling](#52-debouncing-and-throttling)
   - 5.3 [Race Condition Handling](#53-race-condition-handling)
   - 5.4 [Token Refresh Flow](#54-token-refresh-flow)
   - 5.5 [API Versioning](#55-api-versioning)

---

## 1. HTTP Clients

### 1.1 fetch API

`fetch` is available globally in React Native — no import needed. It is Promise-based and works identically to the browser.

```tsx
// Basic GET
const response = await fetch('https://api.example.com/products');
const data = await response.json();

// POST with JSON body
const response = await fetch('https://api.example.com/products', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  },
  body: JSON.stringify({ name: 'Nike Air Max', price: 120 }),
});

if (!response.ok) {
  throw new Error(`HTTP ${response.status}: ${response.statusText}`);
}

const product = await response.json();
```

**Critical `fetch` gotcha — it does NOT throw on HTTP errors (4xx/5xx):**
```tsx
// fetch only rejects on network failure, NOT on 404 or 500
const res = await fetch('/api/user');

// You MUST check res.ok manually
if (!res.ok) {
  const errorBody = await res.json();
  throw new Error(errorBody.message ?? `Error ${res.status}`);
}
```

**Reusable fetch wrapper:**
```tsx
// src/lib/fetchClient.ts
async function request<T>(url: string, options?: RequestInit): Promise<T> {
  const response = await fetch(url, options);

  if (!response.ok) {
    let message = `HTTP error ${response.status}`;
    try {
      const body = await response.json();
      message = body.message ?? message;
    } catch {}
    throw new ApiError(message, response.status);
  }

  return response.json() as Promise<T>;
}

export const fetchClient = {
  get: <T>(url: string, options?: RequestInit) =>
    request<T>(url, { ...options, method: 'GET' }),

  post: <T>(url: string, body: unknown, options?: RequestInit) =>
    request<T>(url, {
      ...options,
      method: 'POST',
      headers: { 'Content-Type': 'application/json', ...options?.headers },
      body: JSON.stringify(body),
    }),

  put: <T>(url: string, body: unknown, options?: RequestInit) =>
    request<T>(url, {
      ...options,
      method: 'PUT',
      headers: { 'Content-Type': 'application/json', ...options?.headers },
      body: JSON.stringify(body),
    }),

  delete: <T>(url: string, options?: RequestInit) =>
    request<T>(url, { ...options, method: 'DELETE' }),
};
```

---

### 1.2 Axios Setup and Instance

Axios throws on 4xx/5xx automatically, has built-in JSON parsing, and supports interceptors cleanly. These are the main reasons to prefer it over raw `fetch`.

```bash
npx expo install axios
```

**Create a configured instance:**
```tsx
// src/lib/apiClient.ts
import axios from 'axios';
import { getAuthToken } from '@/store/authStore';

const apiClient = axios.create({
  baseURL: process.env.EXPO_PUBLIC_API_URL ?? 'https://api.example.com',
  timeout: 10_000,  // 10 seconds
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  },
});

export default apiClient;
```

**Usage:**
```tsx
import apiClient from '@/lib/apiClient';

// GET — response.data is already parsed JSON
const { data: products } = await apiClient.get<Product[]>('/products');

// POST
const { data: product } = await apiClient.post<Product>('/products', {
  name: 'Nike Air Max',
  price: 120,
});

// PUT
await apiClient.put(`/products/${id}`, updatedProduct);

// DELETE
await apiClient.delete(`/products/${id}`);

// With query params
const { data } = await apiClient.get('/products', {
  params: { category: 'shoes', page: 1, limit: 20 },
});
// → GET /products?category=shoes&page=1&limit=20
```

---

### 1.3 Request/Response Interceptors

Interceptors run before every request is sent and after every response is received. Perfect for auth headers, token refresh, and global error logging.

```tsx
// src/lib/apiClient.ts
import axios, { AxiosError, InternalAxiosRequestConfig } from 'axios';
import { getAuthToken, refreshToken, clearSession } from '@/store/authStore';
import { router } from 'expo-router';

const apiClient = axios.create({
  baseURL: process.env.EXPO_PUBLIC_API_URL,
  timeout: 10_000,
});

// ─── Request Interceptor ───────────────────────────────────────────────────
apiClient.interceptors.request.use(
  (config: InternalAxiosRequestConfig) => {
    const token = getAuthToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// ─── Response Interceptor ─────────────────────────────────────────────────
apiClient.interceptors.response.use(
  (response) => response,  // pass through successful responses unchanged
  async (error: AxiosError) => {
    const originalRequest = error.config as InternalAxiosRequestConfig & { _retry?: boolean };

    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;  // prevent infinite retry loop

      try {
        await refreshToken();         // get a new access token
        return apiClient(originalRequest); // replay the original request
      } catch {
        clearSession();               // refresh failed — log user out
        router.replace('/(auth)/login');
        return Promise.reject(error);
      }
    }

    return Promise.reject(error);
  }
);

export default apiClient;
```

**Eject interceptors when no longer needed:**
```tsx
const id = apiClient.interceptors.request.use(handler);
apiClient.interceptors.request.eject(id);
```

---

### 1.4 Base URL Configuration

Store environment-specific base URLs in Expo's environment variable system.

**`.env` files:**
```bash
# .env.development
EXPO_PUBLIC_API_URL=https://dev-api.example.com

# .env.production
EXPO_PUBLIC_API_URL=https://api.example.com

# .env.staging
EXPO_PUBLIC_API_URL=https://staging-api.example.com
```

> All variables exposed to client code **must** be prefixed with `EXPO_PUBLIC_`. Variables without this prefix are not available at runtime.

**Access in code:**
```tsx
// Access anywhere — no import needed
const baseURL = process.env.EXPO_PUBLIC_API_URL;
```

**`app.config.ts` for dynamic config:**
```tsx
// app.config.ts
import { ExpoConfig } from 'expo/config';

const config: ExpoConfig = {
  name: 'MyApp',
  slug: 'myapp',
  extra: {
    apiUrl: process.env.EXPO_PUBLIC_API_URL,
    environment: process.env.APP_ENV ?? 'development',
  },
};

export default config;
```

```tsx
// Read in code via expo-constants
import Constants from 'expo-constants';
const apiUrl = Constants.expoConfig?.extra?.apiUrl;
```

---

### 1.5 Timeout Handling

**Axios — built-in timeout:**
```tsx
const apiClient = axios.create({ timeout: 10_000 }); // throws AxiosError with code ECONNABORTED

// Detect timeout specifically
try {
  await apiClient.get('/products');
} catch (error) {
  if (axios.isAxiosError(error) && error.code === 'ECONNABORTED') {
    Alert.alert('Request timed out', 'Please check your connection and try again.');
  }
}
```

**fetch — manual timeout with AbortController:**
```tsx
async function fetchWithTimeout<T>(url: string, timeoutMs = 10_000): Promise<T> {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const response = await fetch(url, { signal: controller.signal });
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return response.json();
  } catch (error) {
    if (error instanceof DOMException && error.name === 'AbortError') {
      throw new Error('Request timed out');
    }
    throw error;
  } finally {
    clearTimeout(timeoutId);
  }
}
```

**Per-request timeout override in Axios:**
```tsx
// Override default timeout for a specific long-running request
await apiClient.post('/reports/generate', payload, { timeout: 60_000 });
```

---

## 2. Data Fetching Patterns

### 2.1 TanStack Query Integration

Structure your API layer so queries stay thin and logic lives in dedicated API files.

```tsx
// src/api/productsApi.ts
import apiClient from '@/lib/apiClient';
import { ProductSchema, ProductListSchema } from '@/schemas/productSchema';

export const productsApi = {
  getAll: async (params?: { category?: string; page?: number }) => {
    const { data } = await apiClient.get('/products', { params });
    return ProductListSchema.parse(data); // runtime validation with Zod
  },

  getById: async (id: string) => {
    const { data } = await apiClient.get(`/products/${id}`);
    return ProductSchema.parse(data);
  },

  create: async (payload: CreateProductPayload) => {
    const { data } = await apiClient.post('/products', payload);
    return ProductSchema.parse(data);
  },

  update: async (id: string, payload: Partial<CreateProductPayload>) => {
    const { data } = await apiClient.put(`/products/${id}`, payload);
    return ProductSchema.parse(data);
  },

  delete: async (id: string) => {
    await apiClient.delete(`/products/${id}`);
  },
};
```

```tsx
// src/hooks/useProducts.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { productsApi } from '@/api/productsApi';
import { queryKeys } from '@/api/queryKeys';

export function useProducts(params?: { category?: string }) {
  return useQuery({
    queryKey: queryKeys.products.list(params ?? {}),
    queryFn: () => productsApi.getAll(params),
    staleTime: 1000 * 60 * 2, // 2 minutes
  });
}

export function useProduct(id: string) {
  return useQuery({
    queryKey: queryKeys.products.detail(id),
    queryFn: () => productsApi.getById(id),
    enabled: !!id,
  });
}

export function useDeleteProduct() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: productsApi.delete,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: queryKeys.products.all });
    },
  });
}
```

---

### 2.2 Loading States

Show appropriate feedback during data fetch — distinguish between first load and background refetch.

```tsx
import { useProducts } from '@/hooks/useProducts';
import { ActivityIndicator, View, Text, FlatList } from 'react-native';

export default function ProductListScreen() {
  const { data, isLoading, isFetching, isError } = useProducts();

  // First load — no cached data
  if (isLoading) {
    return (
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
        <ActivityIndicator size="large" />
        <Text style={{ marginTop: 8, color: '#666' }}>Loading products...</Text>
      </View>
    );
  }

  return (
    <View style={{ flex: 1 }}>
      {/* Subtle indicator for background refetch — don't block UI */}
      {isFetching && (
        <View style={{ position: 'absolute', top: 0, right: 16, zIndex: 10 }}>
          <ActivityIndicator size="small" />
        </View>
      )}
      <FlatList data={data} renderItem={({ item }) => <ProductCard product={item} />} />
    </View>
  );
}
```

**Skeleton loading — better UX than a spinner for content-heavy screens:**
```tsx
function ProductCardSkeleton() {
  return (
    <View style={styles.skeleton}>
      <View style={styles.skeletonImage} />
      <View style={{ flex: 1, gap: 8 }}>
        <View style={[styles.skeletonLine, { width: '60%' }]} />
        <View style={[styles.skeletonLine, { width: '40%' }]} />
      </View>
    </View>
  );
}

// While loading, show skeletons instead of spinner
if (isLoading) {
  return (
    <View>
      {Array.from({ length: 5 }).map((_, i) => <ProductCardSkeleton key={i} />)}
    </View>
  );
}

const styles = StyleSheet.create({
  skeleton: { flexDirection: 'row', padding: 16, gap: 12 },
  skeletonImage: { width: 80, height: 80, borderRadius: 8, backgroundColor: '#e5e5e5' },
  skeletonLine: { height: 14, borderRadius: 4, backgroundColor: '#e5e5e5' },
});
```

---

### 2.3 Error States

```tsx
import { useProducts } from '@/hooks/useProducts';
import { View, Text, Pressable } from 'react-native';
import { isAxiosError } from 'axios';

export default function ProductListScreen() {
  const { data, isError, error, refetch } = useProducts();

  if (isError) {
    const message = isAxiosError(error)
      ? error.response?.data?.message ?? error.message
      : 'Something went wrong';

    return (
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center', padding: 24 }}>
        <Text style={{ fontSize: 48 }}>⚠️</Text>
        <Text style={{ fontSize: 18, fontWeight: '600', marginTop: 12 }}>
          Failed to load products
        </Text>
        <Text style={{ color: '#666', textAlign: 'center', marginTop: 4 }}>
          {message}
        </Text>
        <Pressable
          onPress={() => refetch()}
          style={{ marginTop: 20, backgroundColor: '#007AFF', padding: 14, borderRadius: 8 }}
        >
          <Text style={{ color: '#fff', fontWeight: '600' }}>Try Again</Text>
        </Pressable>
      </View>
    );
  }

  return (/* list UI */);
}
```

---

### 2.4 Empty States

Always handle the case where the API succeeds but returns no data.

```tsx
function EmptyState({ message, onAction, actionLabel }: {
  message: string;
  onAction?: () => void;
  actionLabel?: string;
}) {
  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center', padding: 32 }}>
      <Text style={{ fontSize: 64 }}>📭</Text>
      <Text style={{ fontSize: 16, color: '#666', textAlign: 'center', marginTop: 12 }}>
        {message}
      </Text>
      {onAction && actionLabel && (
        <Pressable
          onPress={onAction}
          style={{ marginTop: 16, backgroundColor: '#007AFF', padding: 12, borderRadius: 8 }}
        >
          <Text style={{ color: '#fff' }}>{actionLabel}</Text>
        </Pressable>
      )}
    </View>
  );
}

// Usage with FlatList
<FlatList
  data={products}
  renderItem={({ item }) => <ProductCard product={item} />}
  ListEmptyComponent={
    <EmptyState
      message="No products found. Try a different category."
      onAction={() => router.push('/explore')}
      actionLabel="Explore"
    />
  }
/>
```

---

### 2.5 Pagination

Page-based pagination — fetch a specific page number on demand.

```tsx
// src/hooks/useProducts.ts
export function usePagedProducts(page: number, limit = 20) {
  return useQuery({
    queryKey: queryKeys.products.list({ page, limit }),
    queryFn: () => productsApi.getAll({ page, limit }),
    placeholderData: keepPreviousData, // show previous page while next loads
  });
}
```

```tsx
import { usePagedProducts } from '@/hooks/useProducts';
import { useState } from 'react';
import { FlatList, View, Pressable, Text, ActivityIndicator } from 'react-native';
import { keepPreviousData } from '@tanstack/react-query';

export default function ProductListScreen() {
  const [page, setPage] = useState(1);
  const { data, isLoading, isFetching, isPlaceholderData } = usePagedProducts(page);

  return (
    <View style={{ flex: 1 }}>
      {isLoading ? (
        <ActivityIndicator style={{ flex: 1 }} />
      ) : (
        <FlatList
          data={data?.items}
          renderItem={({ item }) => <ProductCard product={item} />}
          // Dim the list while loading next page
          style={{ opacity: isFetching ? 0.5 : 1 }}
        />
      )}

      <View style={{ flexDirection: 'row', justifyContent: 'space-between', padding: 16 }}>
        <Pressable
          onPress={() => setPage(p => Math.max(1, p - 1))}
          disabled={page === 1}
          style={{ padding: 12, backgroundColor: '#007AFF', borderRadius: 8, opacity: page === 1 ? 0.4 : 1 }}
        >
          <Text style={{ color: '#fff' }}>← Prev</Text>
        </Pressable>

        <Text style={{ alignSelf: 'center' }}>Page {page} of {data?.totalPages}</Text>

        <Pressable
          onPress={() => setPage(p => p + 1)}
          disabled={isPlaceholderData || page === data?.totalPages}
          style={{ padding: 12, backgroundColor: '#007AFF', borderRadius: 8 }}
        >
          <Text style={{ color: '#fff' }}>Next →</Text>
        </Pressable>
      </View>
    </View>
  );
}
```

> `placeholderData: keepPreviousData` keeps the previous page visible while the next page loads — prevents a jarring blank state on every page change.

---

### 2.6 Infinite Scroll

Load more items as the user scrolls to the bottom — better UX than page buttons for feeds and lists.

```tsx
// src/hooks/useFeed.ts
import { useInfiniteQuery } from '@tanstack/react-query';
import { productsApi } from '@/api/productsApi';

type ProductPage = { items: Product[]; nextCursor: string | null; total: number };

export function useInfiniteProducts(category?: string) {
  return useInfiniteQuery({
    queryKey: queryKeys.products.list({ category, infinite: true }),
    queryFn: ({ pageParam }) =>
      productsApi.getAll({ category, cursor: pageParam, limit: 20 }) as Promise<ProductPage>,
    initialPageParam: null as string | null,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });
}
```

```tsx
import { useInfiniteProducts } from '@/hooks/useFeed';
import { FlatList, ActivityIndicator, View } from 'react-native';

export default function FeedScreen() {
  const {
    data,
    isLoading,
    isFetchingNextPage,
    hasNextPage,
    fetchNextPage,
    refetch,
    isRefetching,
  } = useInfiniteProducts();

  const items = data?.pages.flatMap(page => page.items) ?? [];

  if (isLoading) return <ActivityIndicator style={{ flex: 1 }} />;

  return (
    <FlatList
      data={items}
      keyExtractor={item => item.id}
      renderItem={({ item }) => <ProductCard product={item} />}

      // Pull-to-refresh
      onRefresh={refetch}
      refreshing={isRefetching}

      // Infinite scroll trigger
      onEndReached={() => {
        if (hasNextPage && !isFetchingNextPage) fetchNextPage();
      }}
      onEndReachedThreshold={0.5}

      // Loading spinner at the bottom
      ListFooterComponent={
        isFetchingNextPage
          ? <ActivityIndicator style={{ padding: 16 }} />
          : null
      }
    />
  );
}
```

---

## 3. Error Handling

### 3.1 Try-Catch Patterns

**In API functions — convert errors to typed domain errors:**
```tsx
// src/lib/errors.ts
export class ApiError extends Error {
  constructor(
    message: string,
    public statusCode: number,
    public code?: string
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

export class NetworkError extends Error {
  constructor(message = 'No internet connection') {
    super(message);
    this.name = 'NetworkError';
  }
}

export class TimeoutError extends Error {
  constructor(message = 'Request timed out') {
    super(message);
    this.name = 'TimeoutError';
  }
}
```

```tsx
// src/api/productsApi.ts
import { isAxiosError } from 'axios';
import { ApiError, NetworkError, TimeoutError } from '@/lib/errors';
import apiClient from '@/lib/apiClient';

export const productsApi = {
  getById: async (id: string): Promise<Product> => {
    try {
      const { data } = await apiClient.get(`/products/${id}`);
      return data;
    } catch (error) {
      if (isAxiosError(error)) {
        if (error.code === 'ECONNABORTED') throw new TimeoutError();
        if (!error.response) throw new NetworkError();
        const message = error.response.data?.message ?? error.message;
        throw new ApiError(message, error.response.status, error.response.data?.code);
      }
      throw error; // re-throw unknown errors
    }
  },
};
```

---

### 3.2 HTTP Status Code Handling

```tsx
// src/lib/apiClient.ts — handle common status codes in the response interceptor
apiClient.interceptors.response.use(
  response => response,
  (error: AxiosError) => {
    const status = error.response?.status;

    switch (status) {
      case 400:
        // Validation error — let the calling code handle field-level errors
        return Promise.reject(
          new ApiError(
            error.response?.data?.message ?? 'Invalid request',
            400,
            'VALIDATION_ERROR'
          )
        );

      case 401:
        // Handled separately in the token refresh interceptor
        break;

      case 403:
        router.replace('/forbidden');
        return Promise.reject(new ApiError('Access denied', 403, 'FORBIDDEN'));

      case 404:
        return Promise.reject(new ApiError('Resource not found', 404, 'NOT_FOUND'));

      case 429:
        return Promise.reject(new ApiError('Too many requests. Slow down.', 429, 'RATE_LIMITED'));

      case 500:
      case 502:
      case 503:
        return Promise.reject(new ApiError('Server error. Please try again later.', status ?? 500, 'SERVER_ERROR'));
    }

    return Promise.reject(error);
  }
);
```

---

### 3.3 Network Error Detection

```tsx
import NetInfo from '@react-native-community/netinfo';

// One-time check before a critical operation
async function checkNetworkBeforeCheckout() {
  const state = await NetInfo.fetch();
  if (!state.isConnected) {
    Alert.alert(
      'No Internet',
      'Please connect to the internet before completing your purchase.',
      [{ text: 'OK' }]
    );
    return false;
  }
  return true;
}

// Continuous monitoring — custom hook
export function useNetworkStatus() {
  const [isOnline, setIsOnline] = useState(true);
  const [connectionType, setConnectionType] = useState<string | null>(null);

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener(state => {
      setIsOnline(state.isConnected ?? false);
      setConnectionType(state.type); // 'wifi' | 'cellular' | 'none' | etc.
    });
    return unsubscribe;
  }, []);

  return { isOnline, connectionType };
}

// Distinguish network error from API error in catch blocks
function isNetworkError(error: unknown): boolean {
  if (isAxiosError(error) && !error.response) return true;
  if (error instanceof NetworkError) return true;
  return false;
}
```

---

### 3.4 Retry Logic

**TanStack Query — built-in retry with exponential backoff:**
```tsx
useQuery({
  queryKey: ['products'],
  queryFn: productsApi.getAll,
  retry: (failureCount, error) => {
    // Don't retry on client errors (4xx) — only on server/network errors
    if (error instanceof ApiError && error.statusCode < 500) return false;
    return failureCount < 3; // max 3 retries
  },
  retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30_000),
  // attempt 1: 1s, attempt 2: 2s, attempt 3: 4s, max: 30s
});
```

**Manual retry with exponential backoff:**
```tsx
async function withRetry<T>(
  fn: () => Promise<T>,
  maxAttempts = 3,
  baseDelayMs = 1000
): Promise<T> {
  let lastError: unknown;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;

      // Don't retry on client errors
      if (error instanceof ApiError && error.statusCode < 500) throw error;

      if (attempt < maxAttempts) {
        const delay = baseDelayMs * 2 ** (attempt - 1); // exponential: 1s, 2s, 4s
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }

  throw lastError;
}

// Usage
const products = await withRetry(() => productsApi.getAll());
```

---

### 3.5 Error Boundaries

Error boundaries catch JavaScript errors in the component tree and show a fallback UI instead of crashing the app. They must be class components (or use the `react-error-boundary` library).

```bash
npx expo install react-error-boundary
```

```tsx
// src/components/ErrorBoundary.tsx
import { ErrorBoundary as ReactErrorBoundary } from 'react-error-boundary';
import { View, Text, Pressable, StyleSheet } from 'react-native';

function ErrorFallback({ error, resetErrorBoundary }: {
  error: Error;
  resetErrorBoundary: () => void;
}) {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Something went wrong</Text>
      <Text style={styles.message}>{error.message}</Text>
      <Pressable style={styles.button} onPress={resetErrorBoundary}>
        <Text style={styles.buttonText}>Try Again</Text>
      </Pressable>
    </View>
  );
}

export function AppErrorBoundary({ children }: { children: React.ReactNode }) {
  return (
    <ReactErrorBoundary
      FallbackComponent={ErrorFallback}
      onError={(error, info) => {
        // Log to crash reporting service (Sentry, Bugsnag, etc.)
        console.error('[ErrorBoundary]', error, info);
      }}
    >
      {children}
    </ReactErrorBoundary>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', alignItems: 'center', padding: 24 },
  title: { fontSize: 20, fontWeight: 'bold', marginBottom: 8 },
  message: { color: '#666', textAlign: 'center', marginBottom: 24 },
  button: { backgroundColor: '#007AFF', padding: 14, borderRadius: 8 },
  buttonText: { color: '#fff', fontWeight: '600' },
});
```

**Use at different levels — whole screen or just a section:**
```tsx
// Wrap entire screen
export default function ProductScreen() {
  return (
    <AppErrorBoundary>
      <ProductContent />
    </AppErrorBoundary>
  );
}

// TanStack Query has a throwOnError option for error boundaries
useQuery({
  queryKey: ['products'],
  queryFn: productsApi.getAll,
  throwOnError: true, // throws to nearest error boundary on failure
});
```

---

### 3.6 Global Error Handlers

Catch unhandled Promise rejections and JS errors that slipped past error boundaries.

```tsx
// app/_layout.tsx
import { useEffect } from 'react';
import * as Sentry from '@sentry/react-native'; // or any crash reporter

export default function RootLayout() {
  useEffect(() => {
    // Catch unhandled promise rejections
    const handler = (event: PromiseRejectionEvent) => {
      console.error('[Unhandled Rejection]', event.reason);
      Sentry.captureException(event.reason);
    };

    // Expo / React Native global error handler
    const originalHandler = ErrorUtils.getGlobalHandler();
    ErrorUtils.setGlobalHandler((error, isFatal) => {
      console.error('[Global Error]', error);
      Sentry.captureException(error);
      originalHandler(error, isFatal); // call original (shows red screen in dev)
    });

    return () => {
      ErrorUtils.setGlobalHandler(originalHandler);
    };
  }, []);

  return <Stack />;
}
```

---

### 3.7 User-Friendly Error Messages

Map technical errors to human-readable messages. Never show raw error messages or stack traces to end users.

```tsx
// src/lib/errorMessages.ts
import { ApiError, NetworkError, TimeoutError } from './errors';

export function getErrorMessage(error: unknown): string {
  if (error instanceof NetworkError) {
    return 'No internet connection. Please check your network and try again.';
  }

  if (error instanceof TimeoutError) {
    return 'The request took too long. Please try again.';
  }

  if (error instanceof ApiError) {
    switch (error.statusCode) {
      case 400: return error.message; // validation messages are already user-friendly
      case 401: return 'Your session has expired. Please log in again.';
      case 403: return 'You don\'t have permission to do that.';
      case 404: return 'The item you\'re looking for doesn\'t exist.';
      case 409: return 'This action conflicts with existing data.';
      case 429: return 'Too many requests. Please wait a moment and try again.';
      case 500:
      case 502:
      case 503: return 'Our servers are having issues. Please try again later.';
      default: return error.message || 'Something went wrong. Please try again.';
    }
  }

  return 'An unexpected error occurred. Please try again.';
}
```

```tsx
// Usage in a mutation's onError
const { mutate } = useMutation({
  mutationFn: productsApi.create,
  onError: (error) => {
    Alert.alert('Error', getErrorMessage(error));
  },
});
```

---

## 4. Data Validation

### 4.1 Zod Schemas for API Responses

APIs can return unexpected shapes — a field goes missing, a type changes, a new field is added. Zod validates the response at runtime and throws a clear error if the shape doesn't match.

```bash
npx expo install zod
```

```tsx
// src/schemas/productSchema.ts
import { z } from 'zod';

export const ProductSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  price: z.number().positive(),
  category: z.enum(['shoes', 'clothing', 'accessories']),
  imageUrl: z.string().url().nullable(),
  inStock: z.boolean().default(true),
  createdAt: z.string().datetime(),
  tags: z.array(z.string()).default([]),
  metadata: z.record(z.string(), z.unknown()).optional(),
});

export const ProductListSchema = z.object({
  items: z.array(ProductSchema),
  total: z.number().int().nonnegative(),
  page: z.number().int().positive(),
  totalPages: z.number().int().nonnegative(),
});

// Zod for mutations — validate outgoing data too
export const CreateProductSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  price: z.number().positive('Price must be greater than 0'),
  category: z.enum(['shoes', 'clothing', 'accessories']),
});
```

---

### 4.2 Runtime Type Checking

Three ways to use Zod for runtime validation:

```tsx
import { z } from 'zod';
import { ProductSchema } from '@/schemas/productSchema';

// 1. parse — throws ZodError if invalid (stops execution)
const product = ProductSchema.parse(apiResponse);

// 2. safeParse — returns { success, data } or { success: false, error }
const result = ProductSchema.safeParse(apiResponse);
if (result.success) {
  console.log(result.data.name); // fully typed
} else {
  console.error(result.error.issues); // array of validation issues with paths
}

// 3. parseAsync — for schemas with async refinements
const product = await ProductSchema.parseAsync(apiResponse);
```

**Integration with API calls:**
```tsx
// src/api/productsApi.ts
export const productsApi = {
  getById: async (id: string): Promise<Product> => {
    const { data } = await apiClient.get(`/products/${id}`);

    // Validate the response — throws if the API changed its shape
    const result = ProductSchema.safeParse(data);
    if (!result.success) {
      // Log schema mismatch for debugging — don't crash the user
      console.warn('[Schema Mismatch] /products/:id', result.error.issues);
      // Fall back to casting in production, throw in development
      if (__DEV__) throw new Error(`Invalid product schema: ${result.error.message}`);
    }

    return result.success ? result.data : data as Product;
  },
};
```

**Partial validation — accept unknown extra fields:**
```tsx
// By default Zod strips unknown fields. Use passthrough to keep them.
const LaxProductSchema = ProductSchema.passthrough();

// Or use partial for optional migration scenarios
const PartialProductSchema = ProductSchema.partial();
```

---

### 4.3 TypeScript Types from Zod

Infer TypeScript types directly from Zod schemas — single source of truth, no duplication.

```tsx
import { z } from 'zod';

// Define schema
const ProductSchema = z.object({
  id: z.string(),
  name: z.string(),
  price: z.number(),
  category: z.enum(['shoes', 'clothing', 'accessories']),
});

// Infer TypeScript type from schema
type Product = z.infer<typeof ProductSchema>;
// Equivalent to: { id: string; name: string; price: number; category: 'shoes' | 'clothing' | 'accessories' }

// For input types (before defaults are applied)
type CreateProductInput = z.input<typeof CreateProductSchema>;
// For output types (after transforms/defaults)
type CreateProductOutput = z.output<typeof CreateProductSchema>;
```

```tsx
// src/types/index.ts — export inferred types for use across the app
export type { Product, ProductList, CreateProduct } from '@/schemas/productSchema';

// src/schemas/productSchema.ts
export type Product = z.infer<typeof ProductSchema>;
export type ProductList = z.infer<typeof ProductListSchema>;
export type CreateProduct = z.infer<typeof CreateProductSchema>;
```

**Transform at parse time:**
```tsx
const ProductSchema = z.object({
  id: z.string(),
  price: z.string().transform(val => parseFloat(val)), // API sends price as string
  createdAt: z.string().transform(val => new Date(val)), // convert to Date object
});

type Product = z.infer<typeof ProductSchema>;
// price is number, createdAt is Date — correctly typed after transform
```

---

## 5. Advanced Concepts

### 5.1 Request Cancellation (AbortController)

Cancel in-flight requests when a component unmounts or a new request supersedes the old one.

**With TanStack Query — automatic cancellation:**
```tsx
// TanStack Query automatically cancels the previous request when
// the query key changes or the component unmounts.
// Pass the signal through to your fetch/axios call:

export const productsApi = {
  getAll: async (params?: object, signal?: AbortSignal): Promise<Product[]> => {
    const { data } = await apiClient.get('/products', { params, signal });
    return data;
  },
};

// TanStack Query passes signal via the queryFn context
useQuery({
  queryKey: ['products', params],
  queryFn: ({ signal }) => productsApi.getAll(params, signal),
});
```

**Manual cancellation with `useEffect`:**
```tsx
import { useEffect, useState } from 'react';

function useProductSearch(query: string) {
  const [results, setResults] = useState<Product[]>([]);

  useEffect(() => {
    if (!query) return;

    const controller = new AbortController();

    apiClient
      .get('/products/search', { params: { q: query }, signal: controller.signal })
      .then(res => setResults(res.data))
      .catch(err => {
        if (axios.isCancel(err)) return; // ignore cancellation — not an error
        console.error(err);
      });

    return () => controller.abort(); // cancel when query changes or component unmounts
  }, [query]);

  return results;
}
```

---

### 5.2 Debouncing and Throttling

**Debouncing** — delay execution until input stops. Used for search inputs.

**Throttling** — limit execution to once per interval. Used for scroll events.

```bash
npx expo install @react-native-community/hooks
# or just implement manually / use lodash
```

**Debounce hook:**
```tsx
// src/hooks/useDebounce.ts
import { useState, useEffect } from 'react';

export function useDebounce<T>(value: T, delayMs: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delayMs);
    return () => clearTimeout(timer); // clear on value change
  }, [value, delayMs]);

  return debouncedValue;
}
```

```tsx
// Usage — search that fires API only after user stops typing for 400ms
import { useDebounce } from '@/hooks/useDebounce';
import { useQuery } from '@tanstack/react-query';

export default function SearchScreen() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 400);

  const { data, isLoading } = useQuery({
    queryKey: ['products', 'search', debouncedQuery],
    queryFn: () => productsApi.search(debouncedQuery),
    enabled: debouncedQuery.length >= 2, // don't search on 1 character
  });

  return (
    <View>
      <TextInput
        value={query}
        onChangeText={setQuery}
        placeholder="Search products..."
      />
      {isLoading && <ActivityIndicator />}
      <FlatList data={data} renderItem={({ item }) => <ProductCard product={item} />} />
    </View>
  );
}
```

**Throttle hook:**
```tsx
// src/hooks/useThrottle.ts
import { useRef, useCallback } from 'react';

export function useThrottle<T extends (...args: any[]) => any>(
  fn: T,
  limitMs: number
): T {
  const lastRunRef = useRef(0);

  return useCallback((...args: Parameters<T>) => {
    const now = Date.now();
    if (now - lastRunRef.current >= limitMs) {
      lastRunRef.current = now;
      return fn(...args);
    }
  }, [fn, limitMs]) as T;
}

// Usage — log scroll position at most once per 200ms
const logScroll = useThrottle((offset: number) => {
  analytics.track('scroll', { offset });
}, 200);
```

---

### 5.3 Race Condition Handling

A race condition occurs when multiple async operations complete out of order. Common example: user types quickly, responses arrive out of sequence, and the last visible result isn't for the last query typed.

**TanStack Query solves this automatically** — it only uses the response for the current query key and ignores stale ones.

**Manual fix with a request ID:**
```tsx
import { useRef, useState, useEffect } from 'react';

function useSearch(query: string) {
  const [results, setResults] = useState<Product[]>([]);
  const latestRequestId = useRef(0);

  useEffect(() => {
    if (!query) { setResults([]); return; }

    const requestId = ++latestRequestId.current; // increment for each new request

    productsApi.search(query).then(data => {
      // Only update state if this is still the latest request
      if (requestId === latestRequestId.current) {
        setResults(data);
      }
      // Otherwise, silently discard the stale response
    });
  }, [query]);

  return results;
}
```

**AbortController approach (cleaner):**
```tsx
useEffect(() => {
  const controller = new AbortController();

  productsApi.search(query, controller.signal)
    .then(setResults)
    .catch(err => { if (!axios.isCancel(err)) console.error(err); });

  return () => controller.abort(); // cancels previous request when query changes
}, [query]);
// The previous request is aborted before the new one starts — no race possible
```

---

### 5.4 Token Refresh Flow

When an access token expires, silently get a new one and replay the failed request — without logging the user out.

```tsx
// src/lib/apiClient.ts
import axios, { AxiosError, InternalAxiosRequestConfig } from 'axios';
import AsyncStorage from '@react-native-async-storage/async-storage';

const apiClient = axios.create({ baseURL: process.env.EXPO_PUBLIC_API_URL, timeout: 10_000 });

// Track whether a refresh is already in progress
let isRefreshing = false;
// Queue of requests that arrived while refreshing
let failedQueue: Array<{ resolve: (token: string) => void; reject: (err: unknown) => void }> = [];

function processQueue(error: unknown, token: string | null) {
  failedQueue.forEach(({ resolve, reject }) => {
    if (token) resolve(token);
    else reject(error);
  });
  failedQueue = [];
}

apiClient.interceptors.response.use(
  response => response,
  async (error: AxiosError) => {
    const originalRequest = error.config as InternalAxiosRequestConfig & { _retry?: boolean };

    if (error.response?.status !== 401 || originalRequest._retry) {
      return Promise.reject(error);
    }

    if (isRefreshing) {
      // Another request is already refreshing — queue this one
      return new Promise((resolve, reject) => {
        failedQueue.push({ resolve, reject });
      }).then(token => {
        originalRequest.headers.Authorization = `Bearer ${token}`;
        return apiClient(originalRequest);
      });
    }

    originalRequest._retry = true;
    isRefreshing = true;

    try {
      const refreshToken = await AsyncStorage.getItem('refreshToken');
      const { data } = await axios.post(`${process.env.EXPO_PUBLIC_API_URL}/auth/refresh`, {
        refreshToken,
      });

      const newAccessToken = data.accessToken;
      await AsyncStorage.setItem('accessToken', newAccessToken);
      apiClient.defaults.headers.Authorization = `Bearer ${newAccessToken}`;

      processQueue(null, newAccessToken);
      originalRequest.headers.Authorization = `Bearer ${newAccessToken}`;
      return apiClient(originalRequest); // replay original request with new token

    } catch (refreshError) {
      processQueue(refreshError, null);
      await AsyncStorage.multiRemove(['accessToken', 'refreshToken']);
      router.replace('/(auth)/login');
      return Promise.reject(refreshError);

    } finally {
      isRefreshing = false;
    }
  }
);

export default apiClient;
```

**Flow summary:**
```
Request → 401 Unauthorized
  ↓
isRefreshing? 
  No  → set isRefreshing = true → call /auth/refresh
  Yes → queue this request

/auth/refresh succeeds?
  Yes → save new token → replay queued requests → replay original request
  No  → clear tokens → redirect to login → reject all queued requests
```

---

### 5.5 API Versioning

Handle API version upgrades without breaking the running app.

**URL-based versioning (most common):**
```tsx
// src/lib/apiClient.ts
const apiClient = axios.create({
  baseURL: `${process.env.EXPO_PUBLIC_API_URL}/v1`, // version in base URL
});

// For v2 endpoints, create a separate instance or override per-request
const apiClientV2 = axios.create({
  baseURL: `${process.env.EXPO_PUBLIC_API_URL}/v2`,
});
```

**Header-based versioning:**
```tsx
const apiClient = axios.create({
  baseURL: process.env.EXPO_PUBLIC_API_URL,
  headers: { 'API-Version': '2024-01-01' }, // date-based version
});
```

**Managing version migrations — feature flags approach:**
```tsx
// src/config/apiConfig.ts
export const API_CONFIG = {
  version: 'v1' as 'v1' | 'v2',
  features: {
    newProductSchema: false, // flip when v2 product endpoint is stable
    paginatedCart: true,
  },
};

// In API file
export const productsApi = {
  getAll: async (): Promise<Product[]> => {
    const endpoint = API_CONFIG.features.newProductSchema
      ? '/v2/products'
      : '/v1/products';
    const { data } = await apiClient.get(endpoint);
    return data;
  },
};
```

**App version–aware requests** — tell the server which app version is calling:
```tsx
import * as Application from 'expo-application';

apiClient.interceptors.request.use(config => {
  config.headers['X-App-Version'] = Application.nativeApplicationVersion ?? '0.0.0';
  config.headers['X-Build-Number'] = Application.nativeBuildVersion ?? '0';
  return config;
});
```

> Versioning lets the server maintain backward compatibility while you migrate the app. Old app versions continue working; new versions opt into the new schema.

---

*End of Module 6*
