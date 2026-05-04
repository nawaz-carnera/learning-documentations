# Module 12 — Architecture, Code Organization & Best Practices

React Native projects grow fast. This module covers how to structure your code so it stays maintainable as the app scales.

---

## Table of Contents

1. [Folder Structure](#1-folder-structure)
   - [Feature-based structure](#feature-based-structure)
   - [Layer-based structure](#layer-based-structure)
   - [src/ organization](#src-organization)
   - [Barrel exports](#barrel-exports-indexts)
   - [Path aliases](#path-aliases-tsconfig-paths)
2. [Reusable Components](#2-reusable-components)
   - [Atomic design principles](#atomic-design-principles)
   - [Component library structure](#component-library-structure)
   - [Prop API design](#prop-api-design)
   - [Compound components](#compound-components)
   - [Render props and children functions](#render-props-and-children-functions)
   - [ForwardRef patterns](#forwardref-patterns)
3. [Custom Hooks](#3-custom-hooks)
   - [Hook extraction patterns](#hook-extraction-patterns)
   - [Naming conventions](#naming-conventions)
   - [Returning tuples vs objects](#returning-tuples-vs-objects)
   - [Shared business logic hooks](#shared-business-logic-hooks)
4. [Clean Code Practices](#4-clean-code-practices)
   - [Separation of concerns](#separation-of-concerns)
   - [Service layer pattern](#service-layer-pattern)
   - [Repository pattern](#repository-pattern)
   - [Container vs presentational components](#container-vs-presentational-components)
   - [Single responsibility](#single-responsibility)
   - [DRY vs premature abstraction](#dry-vs-premature-abstraction)
5. [TypeScript Patterns](#5-typescript-patterns)
   - [Strict mode configuration](#strict-mode-configuration)
   - [Type vs interface](#type-vs-interface)
   - [Generics in components](#generics-in-components)
   - [Utility types](#utility-types)
   - [Type guards](#type-guards)
   - [Discriminated unions](#discriminated-unions)
   - [Avoiding any](#avoiding-any)
6. [Code Quality Tools](#6-code-quality-tools)
   - [ESLint configuration](#eslint-configuration)
   - [Prettier configuration](#prettier-configuration)
   - [Husky pre-commit hooks](#husky-pre-commit-hooks)
   - [lint-staged](#lint-staged)
   - [Commit conventions](#commit-conventions)

---

## 1. Folder Structure

### Feature-based structure

Group everything related to a feature in one folder. You find all files for "orders" in `features/orders/` — no hunting across multiple top-level directories.

```
src/
├── features/
│   ├── auth/
│   │   ├── components/       # LoginForm, OTPInput
│   │   ├── hooks/            # useLogin, useSession
│   │   ├── store/            # authStore.ts
│   │   ├── services/         # authService.ts
│   │   ├── types.ts
│   │   └── index.ts          # barrel export
│   ├── orders/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── services/
│   │   ├── types.ts
│   │   └── index.ts
│   └── profile/
├── shared/
│   ├── components/           # Button, Input, Modal
│   ├── hooks/                # useDebounce, useNetwork
│   ├── utils/                # formatDate, formatCurrency
│   ├── constants/
│   └── types/
├── app/                      # Expo Router screens
└── lib/                      # third-party config (queryClient, axios)
```

**When to use:** Medium-to-large apps where features are owned by separate teams or grow independently.

---

### Layer-based structure

Group by technical layer: all components together, all services together.

```
src/
├── components/
├── hooks/
├── services/
├── stores/
├── utils/
├── types/
└── constants/
```

**When to use:** Small apps or solo projects. Simple to start but becomes hard to navigate once there are 30+ components.

---

### src/ organization

Always put your code under `src/` to separate app source from config files at the root.

```
my-app/
├── src/
│   ├── features/
│   ├── shared/
│   └── lib/
├── app/                 # Expo Router (must stay at root)
├── assets/
├── app.json
├── tsconfig.json
├── package.json
└── .env
```

> Expo Router's `app/` directory must stay at the project root. Your business logic lives in `src/`.

---

### Barrel exports (index.ts)

A barrel re-exports everything from a folder so consumers import from one path instead of deep paths.

```ts
// src/features/auth/index.ts
export { LoginForm } from './components/LoginForm';
export { useLogin } from './hooks/useLogin';
export { useAuthStore } from './store/authStore';
export type { AuthUser, LoginPayload } from './types';
```

```ts
// Before barrel
import { LoginForm } from '@/features/auth/components/LoginForm';
import { useLogin } from '@/features/auth/hooks/useLogin';

// After barrel
import { LoginForm, useLogin } from '@/features/auth';
```

**Avoid barrel exports for large shared/components** — Metro bundler must resolve the whole barrel even if you only need one export, which can slow cold start.

---

### Path aliases (tsconfig paths)

Eliminate `../../../` relative imports. Set up once in `tsconfig.json`.

```json
// tsconfig.json
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@features/*": ["src/features/*"],
      "@shared/*": ["src/shared/*"],
      "@lib/*": ["src/lib/*"]
    }
  }
}
```

Metro needs the same config via `babel-plugin-module-resolver`:

```js
// babel.config.js
module.exports = {
  presets: ['babel-preset-expo'],
  plugins: [
    [
      'module-resolver',
      {
        root: ['./src'],
        alias: {
          '@': './src',
          '@features': './src/features',
          '@shared': './src/shared',
          '@lib': './src/lib',
        },
      },
    ],
  ],
};
```

```ts
// Usage
import { Button } from '@shared/components';
import { useLogin } from '@features/auth';
```

---

## 2. Reusable Components

### Atomic design principles

Break UI into 5 levels from smallest to largest:

| Level | Description | Examples |
|-------|-------------|---------|
| **Atoms** | Smallest building blocks | Button, Text, Icon, Avatar |
| **Molecules** | Atoms combined | FormInput (Label + TextInput + ErrorText) |
| **Organisms** | Complex UI sections | LoginForm, ProductCard, NavigationBar |
| **Templates** | Page layout without data | AuthLayout, DashboardLayout |
| **Pages/Screens** | Templates with real data | LoginScreen, HomeScreen |

```
src/shared/components/
├── atoms/
│   ├── Button/
│   ├── Text/
│   └── Icon/
├── molecules/
│   ├── FormInput/
│   └── SearchBar/
└── organisms/
    ├── Header/
    └── ProductCard/
```

---

### Component library structure

Each component gets its own folder:

```
Button/
├── Button.tsx          # component
├── Button.styles.ts    # styles
├── Button.test.tsx     # tests
├── Button.stories.tsx  # Storybook (optional)
└── index.ts            # barrel: export { Button } from './Button'
```

This makes the component self-contained. Moving or deleting it is safe.

---

### Prop API design

Design props to be predictable and composable.

```tsx
// Bad — too many booleans, unclear combinations
type ButtonProps = {
  isPrimary?: boolean;
  isSecondary?: boolean;
  isDanger?: boolean;
  isSmall?: boolean;
  isLarge?: boolean;
};

// Good — use discriminated union for variants
type ButtonVariant = 'primary' | 'secondary' | 'danger' | 'ghost';
type ButtonSize = 'sm' | 'md' | 'lg';

type ButtonProps = {
  variant?: ButtonVariant;
  size?: ButtonSize;
  label: string;
  onPress: () => void;
  loading?: boolean;
  disabled?: boolean;
  leftIcon?: React.ReactNode;
  style?: StyleProp<ViewStyle>;
};
```

**Rules:**
- Extend native component props when wrapping (`TextInputProps`, `TouchableOpacityProps`)
- Avoid boolean props when a `variant` union is cleaner
- Accept `style` prop for layout overrides — never hardcode margins inside a reusable component
- `onPress` not `onClick`

```tsx
// Extending native props
import { TextInput, TextInputProps } from 'react-native';

type InputProps = TextInputProps & {
  label: string;
  error?: string;
};

export function Input({ label, error, ...rest }: InputProps) {
  return (
    <View>
      <Text>{label}</Text>
      <TextInput {...rest} />
      {error && <Text style={styles.error}>{error}</Text>}
    </View>
  );
}
```

---

### Compound components

Compound components share implicit state through context. The parent manages state; children consume it. Familiar from React — same pattern in RN.

```tsx
// Select/index.tsx
type SelectContextValue = {
  value: string;
  onChange: (v: string) => void;
};

const SelectContext = React.createContext<SelectContextValue | null>(null);

function useSelect() {
  const ctx = React.useContext(SelectContext);
  if (!ctx) throw new Error('Must be used inside <Select>');
  return ctx;
}

function Select({
  value,
  onChange,
  children,
}: {
  value: string;
  onChange: (v: string) => void;
  children: React.ReactNode;
}) {
  return (
    <SelectContext.Provider value={{ value, onChange }}>
      <View>{children}</View>
    </SelectContext.Provider>
  );
}

function Option({ value, label }: { value: string; label: string }) {
  const { value: selected, onChange } = useSelect();
  return (
    <Pressable
      onPress={() => onChange(value)}
      style={[styles.option, selected === value && styles.selected]}
    >
      <Text>{label}</Text>
    </Pressable>
  );
}

Select.Option = Option;
export { Select };
```

```tsx
// Usage — clean API
<Select value={size} onChange={setSize}>
  <Select.Option value="S" label="Small" />
  <Select.Option value="M" label="Medium" />
  <Select.Option value="L" label="Large" />
</Select>
```

---

### Render props and children functions

Pass a function as `children` (or a named prop) to let the parent control rendering while the child controls logic.

```tsx
// NetworkBoundary — handles loading/error, delegates success render
type NetworkBoundaryProps<T> = {
  loading: boolean;
  error: Error | null;
  data: T | undefined;
  children: (data: T) => React.ReactNode;
  loadingFallback?: React.ReactNode;
};

function NetworkBoundary<T>({
  loading,
  error,
  data,
  children,
  loadingFallback = <ActivityIndicator />,
}: NetworkBoundaryProps<T>) {
  if (loading) return <>{loadingFallback}</>;
  if (error) return <ErrorView message={error.message} />;
  if (!data) return null;
  return <>{children(data)}</>;
}
```

```tsx
// Usage
const { data, isLoading, error } = useQuery({ queryKey: ['user'], queryFn: fetchUser });

<NetworkBoundary loading={isLoading} error={error} data={data}>
  {(user) => <ProfileCard user={user} />}
</NetworkBoundary>
```

---

### ForwardRef patterns

When a parent needs a ref to a child component's inner element (e.g., to call `.focus()`).

```tsx
import { forwardRef, useImperativeHandle, useRef } from 'react';
import { TextInput, TextInputProps, View } from 'react-native';

// Expose only what the parent should control
type InputHandle = {
  focus: () => void;
  blur: () => void;
  clear: () => void;
};

type InputProps = TextInputProps & { label: string };

const Input = forwardRef<InputHandle, InputProps>(({ label, ...rest }, ref) => {
  const inputRef = useRef<TextInput>(null);

  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus(),
    blur: () => inputRef.current?.blur(),
    clear: () => inputRef.current?.clear(),
  }));

  return (
    <View>
      <Text>{label}</Text>
      <TextInput ref={inputRef} {...rest} />
    </View>
  );
});

Input.displayName = 'Input';
export { Input };
```

```tsx
// Parent
const emailRef = useRef<InputHandle>(null);

<Input ref={emailRef} label="Email" />
<Button label="Focus Email" onPress={() => emailRef.current?.focus()} />
```

> `useImperativeHandle` limits what the parent can call — better than handing over the full TextInput ref.

---

## 3. Custom Hooks

### Hook extraction patterns

Extract a hook when:
- A component has more than one `useEffect`
- State + derived values + side effects travel together
- The same stateful logic appears in 2+ components

```tsx
// Before — logic mixed into component
function ProductScreen({ id }: { id: string }) {
  const [product, setProduct] = useState<Product | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    setLoading(true);
    fetchProduct(id)
      .then(setProduct)
      .catch((e) => setError(e.message))
      .finally(() => setLoading(false));
  }, [id]);

  // ...render
}

// After — extracted hook
function useProduct(id: string) {
  const [product, setProduct] = useState<Product | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    setLoading(true);
    fetchProduct(id)
      .then(setProduct)
      .catch((e) => setError(e.message))
      .finally(() => setLoading(false));
  }, [id]);

  return { product, loading, error };
}

function ProductScreen({ id }: { id: string }) {
  const { product, loading, error } = useProduct(id);
  // clean render only
}
```

---

### Naming conventions

| Convention | Example | Use |
|------------|---------|-----|
| `use` prefix | `useCart`, `useAuth` | All hooks |
| Noun for state | `useUser`, `useCart` | Hooks that manage a resource |
| Verb for action | `useLogin`, `useUpload` | Hooks that perform an operation |
| Adjective suffix | `useIsOnline`, `useIsFocused` | Hooks that return a boolean |

```ts
useUser()          // returns user data + loading/error
useLogin()         // returns login mutation + status
useIsOnline()      // returns boolean
useCartItems()     // returns array of items
useCartActions()   // returns add/remove/clear functions
```

---

### Returning tuples vs objects

**Tuple** — when the hook has exactly 2 things (like `useState`). Caller picks their own names.

```ts
function useToggle(initial = false): [boolean, () => void] {
  const [value, setValue] = useState(initial);
  return [value, () => setValue((v) => !v)];
}

// Usage — rename freely
const [isOpen, toggleOpen] = useToggle();
const [isVisible, toggleVisible] = useToggle(true);
```

**Object** — when the hook returns 3+ values. Named properties are clearer.

```ts
function useAuth() {
  const user = useAuthStore((s) => s.user);
  const loading = useAuthStore((s) => s.loading);
  const login = useAuthStore((s) => s.login);
  const logout = useAuthStore((s) => s.logout);

  return { user, loading, login, logout };
}

// Usage
const { user, loading, login, logout } = useAuth();
```

**Rule:** 2 values → tuple. 3+ values → object.

---

### Shared business logic hooks

Put reusable business logic in `shared/hooks/`. These are pure logic hooks — no UI.

```ts
// shared/hooks/useDebounce.ts
function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);
  return debounced;
}

// shared/hooks/usePrevious.ts
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>();
  useEffect(() => { ref.current = value; });
  return ref.current;
}

// shared/hooks/useInterval.ts
function useInterval(callback: () => void, delay: number | null) {
  const savedCallback = useRef(callback);
  useEffect(() => { savedCallback.current = callback; }, [callback]);
  useEffect(() => {
    if (delay === null) return;
    const id = setInterval(() => savedCallback.current(), delay);
    return () => clearInterval(id);
  }, [delay]);
}
```

Feature-specific business logic goes in `features/<name>/hooks/`:

```ts
// features/cart/hooks/useCartSummary.ts
function useCartSummary() {
  const items = useCartStore((s) => s.items);

  const subtotal = useMemo(
    () => items.reduce((sum, item) => sum + item.price * item.qty, 0),
    [items]
  );
  const itemCount = useMemo(
    () => items.reduce((sum, item) => sum + item.qty, 0),
    [items]
  );
  const tax = subtotal * 0.1;
  const total = subtotal + tax;

  return { subtotal, tax, total, itemCount };
}
```

---

## 4. Clean Code Practices

### Separation of concerns

Each layer should have one job:

| Layer | Responsibility | Should NOT |
|-------|---------------|-----------|
| **Screen/Page** | Layout, navigation, wiring | Contain API calls or business logic |
| **Component** | Render UI from props | Fetch data or access global state directly |
| **Hook** | Encapsulate stateful logic | Render JSX |
| **Service** | API calls, data transformation | Know about UI or state |
| **Store** | Global state | Call APIs directly |

---

### Service layer pattern

A service is a plain object (or class) with async functions. It owns the network call and data transformation. Nothing else should know about the raw API shape.

```ts
// features/orders/services/orderService.ts
import { apiClient } from '@lib/apiClient';
import { Order, CreateOrderPayload, orderSchema } from '../types';

export const orderService = {
  async getAll(): Promise<Order[]> {
    const { data } = await apiClient.get('/orders');
    return data.map((item: unknown) => orderSchema.parse(item));
  },

  async getById(id: string): Promise<Order> {
    const { data } = await apiClient.get(`/orders/${id}`);
    return orderSchema.parse(data);
  },

  async create(payload: CreateOrderPayload): Promise<Order> {
    const { data } = await apiClient.post('/orders', payload);
    return orderSchema.parse(data);
  },

  async cancel(id: string): Promise<void> {
    await apiClient.patch(`/orders/${id}/cancel`);
  },
};
```

```ts
// Hook consumes the service — no axios/fetch in hooks
function useOrders() {
  return useQuery({
    queryKey: ['orders'],
    queryFn: () => orderService.getAll(),
  });
}
```

---

### Repository pattern

Repository abstracts the data source (API vs SQLite vs MMKV). Swap implementations without touching hooks.

```ts
// features/products/repositories/productRepository.ts
export interface ProductRepository {
  findAll(): Promise<Product[]>;
  findById(id: string): Promise<Product | null>;
  save(product: Product): Promise<void>;
  delete(id: string): Promise<void>;
}

// Remote implementation
export class RemoteProductRepository implements ProductRepository {
  async findAll() {
    const { data } = await apiClient.get('/products');
    return data;
  }
  async findById(id: string) {
    const { data } = await apiClient.get(`/products/${id}`);
    return data;
  }
  async save(product: Product) {
    await apiClient.post('/products', product);
  }
  async delete(id: string) {
    await apiClient.delete(`/products/${id}`);
  }
}

// Local SQLite implementation (for offline)
export class LocalProductRepository implements ProductRepository {
  async findAll() {
    return db.getAllAsync<Product>('SELECT * FROM products');
  }
  async findById(id: string) {
    return db.getFirstAsync<Product>('SELECT * FROM products WHERE id = ?', [id]);
  }
  async save(product: Product) {
    await db.runAsync(
      'INSERT OR REPLACE INTO products VALUES (?, ?, ?)',
      [product.id, product.name, product.price]
    );
  }
  async delete(id: string) {
    await db.runAsync('DELETE FROM products WHERE id = ?', [id]);
  }
}

// Inject via context or DI
export const productRepo = new RemoteProductRepository();
```

---

### Container vs presentational components

**Presentational** — pure UI, driven entirely by props, no data fetching.

```tsx
// Pure UI — easy to test, easy to reuse
type OrderCardProps = {
  order: Order;
  onPress: (id: string) => void;
  onCancel: (id: string) => void;
};

function OrderCard({ order, onPress, onCancel }: OrderCardProps) {
  return (
    <Pressable onPress={() => onPress(order.id)} style={styles.card}>
      <Text style={styles.id}>#{order.id}</Text>
      <Text style={styles.status}>{order.status}</Text>
      <Text style={styles.total}>${order.total}</Text>
      {order.status === 'pending' && (
        <Button label="Cancel" onPress={() => onCancel(order.id)} />
      )}
    </Pressable>
  );
}
```

**Container** — wires data to the presentational component.

```tsx
function OrderCardContainer({ orderId }: { orderId: string }) {
  const { data: order } = useOrder(orderId);
  const cancelOrder = useCancelOrder();
  const router = useRouter();

  if (!order) return <OrderCardSkeleton />;

  return (
    <OrderCard
      order={order}
      onPress={(id) => router.push(`/orders/${id}`)}
      onCancel={(id) => cancelOrder.mutate(id)}
    />
  );
}
```

---

### Single responsibility

Every function, hook, and component should do one thing.

```tsx
// Bad — one component does too much
function ProfileScreen() {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => { /* fetch user */ }, []);
  useEffect(() => { /* fetch posts */ }, []);

  function handleLogout() { /* logout logic */ }
  function handleAvatarChange() { /* upload logic */ }
  function handleBioUpdate() { /* update bio */ }

  return (/* 200 lines of JSX */);
}

// Good — split by responsibility
function ProfileScreen() {
  return (
    <ScrollView>
      <ProfileHeader />      {/* owns avatar + bio */}
      <ProfileStats />       {/* owns follower counts */}
      <ProfilePostGrid />    {/* owns user posts */}
    </ScrollView>
  );
}
```

---

### DRY vs premature abstraction

DRY (Don't Repeat Yourself) is good. But abstracting 2 similar-but-not-identical things too early creates code that's hard to change.

```tsx
// Two screens look similar — resist the urge to merge immediately
function LoginScreen() {
  return <AuthForm title="Log in" submitLabel="Sign in" onSubmit={handleLogin} />;
}

function RegisterScreen() {
  return <AuthForm title="Create account" submitLabel="Register" onSubmit={handleRegister} />;
}

// Only extract when: 3+ usages, the variation is clear, abstraction reduces bugs
// Rule of 3: wait until you need it a third time before abstracting
```

Signs you've abstracted too early:
- The abstraction has 10+ props
- You find yourself adding `if (variant === 'X')` inside it
- Reading the component requires jumping between 3 files

---

## 5. TypeScript Patterns

### Strict mode configuration

Always enable strict mode. It catches null/undefined bugs at compile time.

```json
// tsconfig.json
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

| Flag | What it catches |
|------|----------------|
| `strict` | Enables all strict flags |
| `noUncheckedIndexedAccess` | `arr[0]` is `T \| undefined`, not just `T` |
| `exactOptionalPropertyTypes` | `{ foo?: string }` — `foo` cannot be set to `undefined` explicitly |
| `noImplicitReturns` | Function paths that forget to return |

---

### Type vs interface

Both work for objects. Use `interface` for public APIs and `type` for everything else.

```ts
// interface — extendable, use for component props and public APIs
interface User {
  id: string;
  name: string;
  email: string;
}

interface AdminUser extends User {
  role: 'admin';
  permissions: string[];
}

// type — flexible, use for unions, intersections, computed types
type Status = 'idle' | 'loading' | 'success' | 'error';
type Nullable<T> = T | null;
type UserWithStatus = User & { status: Status };
type ButtonVariant = 'primary' | 'secondary' | 'ghost';
```

| Use case | Prefer |
|----------|--------|
| Component props | `interface` |
| Extending native props | `interface` |
| Union of values | `type` |
| Computed/mapped types | `type` |
| Generic utilities | `type` |

---

### Generics in components

Write components that work with any data shape without losing type safety.

```tsx
// Generic list component
type ListProps<T> = {
  data: T[];
  keyExtractor: (item: T) => string;
  renderItem: (item: T) => React.ReactNode;
  emptyMessage?: string;
};

function List<T>({ data, keyExtractor, renderItem, emptyMessage = 'No items' }: ListProps<T>) {
  if (data.length === 0) return <Text>{emptyMessage}</Text>;
  return (
    <FlatList
      data={data}
      keyExtractor={keyExtractor}
      renderItem={({ item }) => <>{renderItem(item)}</>}
    />
  );
}

// Usage — fully typed
<List
  data={orders}                          // inferred as Order[]
  keyExtractor={(order) => order.id}     // order is Order
  renderItem={(order) => <OrderCard order={order} onPress={...} onCancel={...} />}
/>
```

```tsx
// Generic form field hook
function useField<T extends string | number | boolean>(initial: T) {
  const [value, setValue] = useState<T>(initial);
  const [error, setError] = useState<string | null>(null);
  const reset = () => { setValue(initial); setError(null); };
  return { value, setValue, error, setError, reset };
}
```

---

### Utility types

TypeScript's built-in mapped types save you from repeating yourself.

```ts
interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
}

// Partial — all fields optional (useful for updates/patches)
type UserUpdate = Partial<User>;

// Required — all fields mandatory
type UserRequired = Required<User>;

// Pick — only some fields
type UserPreview = Pick<User, 'id' | 'name'>;

// Omit — all fields except some
type CreateUserPayload = Omit<User, 'id'>;

// Readonly — immutable
type ImmutableUser = Readonly<User>;

// Record — typed dictionary
type UserMap = Record<string, User>;

// ReturnType — infer return type of a function
const fetchUser = async () => ({ id: '1', name: 'Alice' });
type FetchedUser = Awaited<ReturnType<typeof fetchUser>>;

// Parameters — infer params of a function
type FetchParams = Parameters<typeof fetchUser>;

// NonNullable — remove null/undefined
type DefinitelyUser = NonNullable<User | null | undefined>;
```

---

### Type guards

Narrow a wide type to a specific one at runtime.

```ts
// typeof guard
function formatValue(value: string | number) {
  if (typeof value === 'string') return value.toUpperCase(); // string
  return value.toFixed(2);                                   // number
}

// instanceof guard
function handleError(error: unknown) {
  if (error instanceof Error) {
    console.log(error.message);  // Error
  } else {
    console.log(String(error));
  }
}

// Custom type guard (predicate)
type ApiError = { code: number; message: string };

function isApiError(error: unknown): error is ApiError {
  return (
    typeof error === 'object' &&
    error !== null &&
    'code' in error &&
    'message' in error
  );
}

// Usage
try {
  await apiCall();
} catch (error) {
  if (isApiError(error)) {
    // error is ApiError here
    showToast(error.message);
  }
}
```

---

### Discriminated unions

Model state that can be in several exclusive shapes. TypeScript narrows automatically based on the tag.

```ts
// Request state — 4 exclusive shapes
type RequestState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string };

function useAsync<T>(fn: () => Promise<T>) {
  const [state, setState] = useState<RequestState<T>>({ status: 'idle' });

  const run = async () => {
    setState({ status: 'loading' });
    try {
      const data = await fn();
      setState({ status: 'success', data });
    } catch (e) {
      setState({ status: 'error', error: (e as Error).message });
    }
  };

  return { state, run };
}

// Rendering — exhaustive narrowing
function render<T>(state: RequestState<T>, renderData: (data: T) => React.ReactNode) {
  switch (state.status) {
    case 'idle':    return null;
    case 'loading': return <ActivityIndicator />;
    case 'success': return <>{renderData(state.data)}</>; // data is T here
    case 'error':   return <ErrorText message={state.error} />; // error is string
  }
}
```

```ts
// Navigation route params — discriminated union ensures right params per route
type RootStackParams =
  | { screen: 'Home' }
  | { screen: 'Profile'; params: { userId: string } }
  | { screen: 'Order'; params: { orderId: string; fromCheckout?: boolean } };
```

---

### Avoiding any

`any` disables type checking entirely. Use safer alternatives.

```ts
// Bad
function processData(data: any) {
  return data.value.toUpperCase(); // no protection
}

// unknown — forces you to check before using
function processData(data: unknown) {
  if (typeof data === 'object' && data !== null && 'value' in data) {
    const { value } = data as { value: string };
    return value.toUpperCase();
  }
  throw new Error('Invalid data shape');
}

// never — exhaustive checks
function assertNever(x: never): never {
  throw new Error(`Unhandled case: ${x}`);
}

type Shape = 'circle' | 'square' | 'triangle';
function area(shape: Shape) {
  switch (shape) {
    case 'circle':   return Math.PI;
    case 'square':   return 1;
    case 'triangle': return 0.5;
    default:         return assertNever(shape); // compile error if case added but not handled
  }
}
```

| Escape hatch | When to use |
|-------------|-------------|
| `unknown` | External data, `catch` blocks, JSON responses |
| `as Type` | After a type guard has confirmed the shape |
| `never` | Exhaustive switches, unreachable branches |
| `@ts-expect-error` | One-off test cases — better than `@ts-ignore` (errors if the suppression is wrong) |

---

## 6. Code Quality Tools

### ESLint configuration

Install:

```bash
npx expo install eslint eslint-config-expo @typescript-eslint/eslint-plugin @typescript-eslint/parser --save-dev
```

```js
// eslint.config.js (flat config, ESLint v9+)
const { defineConfig } = require('eslint/config');
const expoConfig = require('eslint-config-expo/flat');
const tsPlugin = require('@typescript-eslint/eslint-plugin');

module.exports = defineConfig([
  expoConfig,
  {
    plugins: { '@typescript-eslint': tsPlugin },
    rules: {
      // TypeScript
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      '@typescript-eslint/consistent-type-imports': ['error', { prefer: 'type-imports' }],

      // React
      'react/self-closing-comp': 'error',
      'react-hooks/exhaustive-deps': 'warn',

      // General
      'no-console': ['warn', { allow: ['warn', 'error'] }],
      'prefer-const': 'error',
      'no-var': 'error',
    },
  },
  {
    ignores: ['node_modules/', '.expo/', 'dist/'],
  },
]);
```

---

### Prettier configuration

```bash
npm install --save-dev prettier
```

```js
// prettier.config.js
module.exports = {
  semi: true,
  singleQuote: true,
  trailingComma: 'all',
  printWidth: 100,
  tabWidth: 2,
  useTabs: false,
  bracketSpacing: true,
  arrowParens: 'always',
  endOfLine: 'lf',
};
```

```json
// .prettierignore
node_modules
.expo
dist
*.generated.*
```

Add scripts to `package.json`:

```json
{
  "scripts": {
    "lint": "eslint . --ext .ts,.tsx",
    "lint:fix": "eslint . --ext .ts,.tsx --fix",
    "format": "prettier --write \"src/**/*.{ts,tsx,json}\"",
    "format:check": "prettier --check \"src/**/*.{ts,tsx,json}\""
  }
}
```

---

### Husky pre-commit hooks

Husky runs scripts before `git commit`. Stops bad code from entering the repo.

```bash
npm install --save-dev husky
npx husky init
```

This creates `.husky/pre-commit`. Edit it:

```sh
# .husky/pre-commit
npx lint-staged
```

Add the `prepare` script so Husky installs automatically after `npm install`:

```json
// package.json
{
  "scripts": {
    "prepare": "husky"
  }
}
```

---

### lint-staged

Run linters only on staged files — much faster than linting the whole project.

```bash
npm install --save-dev lint-staged
```

```js
// lint-staged.config.js
module.exports = {
  '*.{ts,tsx}': [
    'eslint --fix',
    'prettier --write',
  ],
  '*.{json,md}': [
    'prettier --write',
  ],
};
```

Flow on `git commit`:
1. Husky fires `.husky/pre-commit`
2. `lint-staged` runs ESLint + Prettier on staged `.ts/.tsx` files only
3. Fixed files are re-staged automatically
4. If ESLint fails, the commit is blocked

---

### Commit conventions

Conventional Commits format: `<type>(<scope>): <description>`

```
feat(auth): add biometric login support
fix(cart): prevent duplicate item on rapid tap
refactor(orders): extract orderService from useOrders hook
chore(deps): upgrade expo-router to 4.1.0
docs(readme): update local dev setup steps
test(auth): add login flow integration tests
perf(images): switch to expo-image for better caching
```

| Type | When |
|------|------|
| `feat` | New feature |
| `fix` | Bug fix |
| `refactor` | Code change with no behavior change |
| `chore` | Dependencies, config, build scripts |
| `docs` | Documentation only |
| `test` | Adding or updating tests |
| `perf` | Performance improvement |
| `style` | Formatting only |

Enforce with `commitlint`:

```bash
npm install --save-dev @commitlint/cli @commitlint/config-conventional
```

```js
// commitlint.config.js
module.exports = { extends: ['@commitlint/config-conventional'] };
```

```sh
# .husky/commit-msg
npx --no -- commitlint --edit "$1"
```

Now a commit message like `updated stuff` will be rejected; `fix(auth): correct token expiry check` will pass.

---

> **Key takeaways**
> - Feature-based structure scales better than layer-based for large apps
> - Barrel exports simplify imports but skip them for large shared folders
> - Compound components and ForwardRef are the two most under-used patterns in RN codebases
> - Service layer keeps hooks thin; repository pattern makes swapping data sources easy
> - Discriminated unions + `unknown` replace 90% of `any` usage
> - Husky + lint-staged catches issues locally so CI stays green
