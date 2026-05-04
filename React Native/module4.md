# Module 4 — Navigation with Expo Router

> Expo Router is a file-based routing system built on top of React Navigation. If you know Next.js routing, this will feel immediately familiar.

---

## Table of Contents

1. [Expo Router](#1-expo-router)
   - 1.1 [File-Based Routing](#11-file-based-routing)
   - 1.2 [app Directory Structure](#12-app-directory-structure)
   - 1.3 [Layouts and Nested Routes](#13-layouts-and-nested-routes)
   - 1.4 [Dynamic Routes](#14-dynamic-routes-idtsx)
   - 1.5 [Route Groups](#15-route-groups)
   - 1.6 [Modal Routes](#16-modal-routes)

2. [Navigation Patterns](#2-navigation-patterns)
   - 2.1 [Passing Params Between Screens](#21-passing-params-between-screens)
   - 2.2 [Nested Navigators](#22-nested-navigators)
   - 2.3 [Authentication Flow Navigation](#23-authentication-flow-navigation)
   - 2.4 [Deep Linking](#24-deep-linking)
   - 2.5 [Universal Links / App Links](#25-universal-links--app-links)
   - 2.6 [useNavigation Hook](#26-usenavigation-hook)
   - 2.7 [useRoute Hook](#27-useroute-hook)
   - 2.8 [useFocusEffect](#28-usefocuseffect)

3. [Header & UI Customization](#3-header--ui-customization)
   - 3.1 [Screen Options](#31-screen-options)
   - 3.2 [Custom Headers](#32-custom-headers)
   - 3.3 [Tab Bar Customization](#33-tab-bar-customization)
   - 3.4 [Back Button Handling](#34-back-button-handling)
   - 3.5 [Hardware Back Button (Android)](#35-hardware-back-button-android)

---

## 1. Expo Router

### 1.1 File-Based Routing

Every file inside the `app/` directory automatically becomes a route. No route config file needed.

| File | Route |
|---|---|
| `app/index.tsx` | `/` (home) |
| `app/profile.tsx` | `/profile` |
| `app/settings/index.tsx` | `/settings` |
| `app/settings/notifications.tsx` | `/settings/notifications` |
| `app/product/[id].tsx` | `/product/123`, `/product/abc` |
| `app/+not-found.tsx` | 404 fallback |

**Special filenames:**

| File | Purpose |
|---|---|
| `_layout.tsx` | Layout wrapper for a segment (required for nested routes) |
| `index.tsx` | Default route for a directory |
| `+not-found.tsx` | Catch-all 404 screen |
| `+html.tsx` | Custom HTML shell (web only) |

**Navigate between screens:**
```tsx
import { Link, router } from 'expo-router';
import { View, Text, Pressable } from 'react-native';

export default function Home() {
  return (
    <View>
      {/* Declarative — like <a> tag */}
      <Link href="/profile">Go to Profile</Link>

      {/* With styling */}
      <Link href="/settings" asChild>
        <Pressable>
          <Text>Settings</Text>
        </Pressable>
      </Link>

      {/* Imperative */}
      <Pressable onPress={() => router.push('/profile')}>
        <Text>Push Profile</Text>
      </Pressable>
    </View>
  );
}
```

**Router methods:**
```tsx
router.push('/profile')      // push onto stack (adds to history)
router.replace('/home')      // replace current screen (no back button)
router.back()                // go back
router.navigate('/profile')  // push if not already in stack, else go back to it
router.dismiss()             // close modal or go back
```

---

### 1.2 app Directory Structure

A real-world Expo Router project structure:

```
app/
├── _layout.tsx                  ← Root layout (providers, fonts, theme)
├── index.tsx                    ← "/" — splash / redirect screen
├── +not-found.tsx               ← 404
│
├── (auth)/                      ← Route group: auth screens (not in URL)
│   ├── _layout.tsx
│   ├── login.tsx                ← "/login"
│   └── register.tsx             ← "/register"
│
├── (tabs)/                      ← Route group: main app with tab bar
│   ├── _layout.tsx              ← Tab navigator config
│   ├── index.tsx                ← "/" tab (Home)
│   ├── explore.tsx              ← "/explore" tab
│   └── profile.tsx              ← "/profile" tab
│
├── product/
│   ├── [id].tsx                 ← "/product/123"
│   └── [id]/
│       └── reviews.tsx          ← "/product/123/reviews"
│
├── modal.tsx                    ← "/modal" — presented as modal
└── settings/
    ├── _layout.tsx
    ├── index.tsx                ← "/settings"
    └── notifications.tsx        ← "/settings/notifications"
```

---

### 1.3 Layouts and Nested Routes

A `_layout.tsx` file wraps all sibling and child routes in that directory. It uses Expo Router's navigator components (`Stack`, `Tabs`, `Drawer`) and must render a `<Slot />` or navigator.

**Root layout — `app/_layout.tsx`:**
```tsx
import { Stack } from 'expo-router';
import { SafeAreaProvider } from 'react-native-safe-area-context';
import { ThemeProvider } from '@/theme/ThemeProvider';
import { useFonts } from 'expo-font';
import * as SplashScreen from 'expo-splash-screen';
import { useEffect } from 'react';

SplashScreen.preventAutoHideAsync();

export default function RootLayout() {
  const [loaded] = useFonts({
    'Inter-Regular': require('../assets/fonts/Inter-Regular.ttf'),
    'Inter-Bold': require('../assets/fonts/Inter-Bold.ttf'),
  });

  useEffect(() => {
    if (loaded) SplashScreen.hideAsync();
  }, [loaded]);

  if (!loaded) return null;

  return (
    <SafeAreaProvider>
      <ThemeProvider>
        <Stack screenOptions={{ headerShown: false }} />
      </ThemeProvider>
    </SafeAreaProvider>
  );
}
```

**Nested stack layout — `app/settings/_layout.tsx`:**
```tsx
import { Stack } from 'expo-router';

export default function SettingsLayout() {
  return (
    <Stack>
      <Stack.Screen name="index" options={{ title: 'Settings' }} />
      <Stack.Screen name="notifications" options={{ title: 'Notifications' }} />
    </Stack>
  );
}
```

**Tab layout — `app/(tabs)/_layout.tsx`:**
```tsx
import { Tabs } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';

export default function TabsLayout() {
  return (
    <Tabs screenOptions={{ headerShown: false }}>
      <Tabs.Screen
        name="index"
        options={{
          title: 'Home',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="home" color={color} size={size} />
          ),
        }}
      />
      <Tabs.Screen
        name="explore"
        options={{
          title: 'Explore',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="search" color={color} size={size} />
          ),
        }}
      />
      <Tabs.Screen
        name="profile"
        options={{
          title: 'Profile',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="person" color={color} size={size} />
          ),
        }}
      />
    </Tabs>
  );
}
```

---

### 1.4 Dynamic Routes ([id].tsx)

A filename wrapped in `[brackets]` creates a dynamic segment — it matches any value.

```
app/product/[id].tsx     →  /product/1, /product/abc, /product/nike-shoe
app/user/[username].tsx  →  /user/nawaz, /user/john
```

**Reading the dynamic param:**
```tsx
// app/product/[id].tsx
import { useLocalSearchParams } from 'expo-router';
import { View, Text } from 'react-native';

export default function ProductScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();

  return (
    <View>
      <Text>Product ID: {id}</Text>
    </View>
  );
}
```

**Navigating to a dynamic route:**
```tsx
import { router, Link } from 'expo-router';

// Imperative
router.push(`/product/${productId}`);
router.push({ pathname: '/product/[id]', params: { id: productId } });

// Declarative
<Link href={`/product/${productId}`}>View Product</Link>
<Link href={{ pathname: '/product/[id]', params: { id: productId } }}>View Product</Link>
```

**Catch-all routes** — match multiple segments:
```
app/docs/[...slug].tsx  →  /docs/intro, /docs/api/hooks, /docs/a/b/c
```

```tsx
// app/docs/[...slug].tsx
import { useLocalSearchParams } from 'expo-router';

export default function DocsPage() {
  const { slug } = useLocalSearchParams<{ slug: string[] }>();
  // /docs/api/hooks → slug = ['api', 'hooks']
  return <Text>{slug?.join(' / ')}</Text>;
}
```

---

### 1.5 Route Groups

Folders wrapped in `(parentheses)` are **route groups** — they organize files without adding the folder name to the URL. Think of them as logical folders, not URL segments.

```
app/(auth)/login.tsx      →  URL: /login  (not /auth/login)
app/(tabs)/index.tsx      →  URL: /       (not /tabs)
app/(tabs)/explore.tsx    →  URL: /explore
```

**Common use cases:**

```
app/
├── (auth)/              ← Screens shown before login
│   ├── _layout.tsx      ← Stack without tab bar
│   ├── login.tsx
│   └── register.tsx
│
├── (app)/               ← Screens shown after login
│   ├── _layout.tsx      ← Tabs navigator
│   ├── (tabs)/
│   └── ...
│
└── _layout.tsx          ← Root — decides which group to show
```

**Multiple layouts for the same URL depth** — route groups let you have different layouts for different sections without changing the URL:

```tsx
// app/_layout.tsx — root decides where to send the user
import { Redirect } from 'expo-router';
import { useAuth } from '@/hooks/useAuth';

export default function RootLayout() {
  const { isLoggedIn } = useAuth();

  // Redirect based on auth state
  return isLoggedIn
    ? <Redirect href="/(app)/(tabs)/" />
    : <Redirect href="/(auth)/login" />;
}
```

---

### 1.6 Modal Routes

Expo Router supports presenting screens as modals (slides up from bottom on iOS).

**Method 1 — Configure in the parent layout:**
```tsx
// app/_layout.tsx
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
      {/* This screen will present as a modal */}
      <Stack.Screen
        name="modal"
        options={{
          presentation: 'modal',         // slides up on iOS
          title: 'Options',
        }}
      />
      <Stack.Screen
        name="image-viewer"
        options={{
          presentation: 'fullScreenModal', // full screen, no drag-to-dismiss
        }}
      />
    </Stack>
  );
}
```

**Method 2 — Set options inside the screen file:**
```tsx
// app/modal.tsx
import { Stack, router } from 'expo-router';
import { View, Text, Pressable } from 'react-native';

export default function Modal() {
  return (
    <>
      <Stack.Screen options={{ presentation: 'modal', title: 'Filter' }} />
      <View style={{ flex: 1, padding: 16 }}>
        <Text>Modal content</Text>
        <Pressable onPress={() => router.dismiss()}>
          <Text>Close</Text>
        </Pressable>
      </View>
    </>
  );
}
```

**Navigate to modal:**
```tsx
router.push('/modal');
// or
<Link href="/modal">Open Modal</Link>
```

**Presentation options:**

| Value | Behavior |
|---|---|
| `'card'` (default) | Push from right |
| `'modal'` | Slide up from bottom, swipe-down to dismiss |
| `'transparentModal'` | Slide up, background is transparent |
| `'fullScreenModal'` | Full screen, no swipe-to-dismiss |
| `'formSheet'` | iOS sheet style (detent-based) |

---

## 2. Navigation Patterns

### 2.1 Passing Params Between Screens

**Method 1 — URL params (recommended for shareable/deep-linked data):**
```tsx
// Sender
router.push({ pathname: '/product/[id]', params: { id: '42', source: 'home' } });

// Receiver — app/product/[id].tsx
import { useLocalSearchParams } from 'expo-router';

export default function ProductScreen() {
  const { id, source } = useLocalSearchParams<{ id: string; source: string }>();
  // id = '42', source = 'home'
}
```

**Method 2 — Global state (recommended for complex/non-serializable data):**
```tsx
// Use Zustand, Redux, or Context to share data
// The screen reads from the store, not from route params
const useProductStore = create((set) => ({
  selectedProduct: null,
  setSelectedProduct: (p) => set({ selectedProduct: p }),
}));

// Sender
useProductStore.getState().setSelectedProduct(product);
router.push('/product-detail');

// Receiver
const { selectedProduct } = useProductStore();
```

**`useLocalSearchParams` vs `useGlobalSearchParams`:**

| Hook | Scope | Use when |
|---|---|---|
| `useLocalSearchParams` | Current screen only | Default — always prefer this |
| `useGlobalSearchParams` | Entire URL | Reading params from a parent layout |

> Params are always **strings** in the URL. Parse numbers explicitly: `const numId = Number(id)`.

---

### 2.2 Nested Navigators

Expo Router handles nesting automatically through directory structure. A tab screen can have its own stack:

```
app/
├── (tabs)/
│   ├── _layout.tsx          ← Tabs navigator
│   ├── index.tsx            ← Home tab (no stack)
│   └── shop/
│       ├── _layout.tsx      ← Stack inside the Shop tab
│       ├── index.tsx        ← /shop — shop list
│       └── [id].tsx         ← /shop/123 — product detail
```

```tsx
// app/(tabs)/shop/_layout.tsx — Stack inside a tab
import { Stack } from 'expo-router';

export default function ShopLayout() {
  return (
    <Stack>
      <Stack.Screen name="index" options={{ title: 'Shop' }} />
      <Stack.Screen name="[id]" options={{ title: 'Product' }} />
    </Stack>
  );
}
```

When navigating to `/shop/123`, the tab bar stays visible and the product detail pushes onto the shop stack — this is the standard e-commerce navigation pattern.

---

### 2.3 Authentication Flow Navigation

The standard pattern: redirect unauthenticated users to login, and authenticated users away from login.

```tsx
// app/_layout.tsx — root layout handles auth split
import { Stack, Redirect, SplashScreen } from 'expo-router';
import { useAuth } from '@/hooks/useAuth';
import { useEffect } from 'react';

SplashScreen.preventAutoHideAsync();

export default function RootLayout() {
  const { isLoggedIn, isLoading } = useAuth();

  useEffect(() => {
    if (!isLoading) SplashScreen.hideAsync();
  }, [isLoading]);

  if (isLoading) return null; // keep splash screen visible

  return (
    <Stack screenOptions={{ headerShown: false }}>
      <Stack.Screen name="(auth)" redirect={isLoggedIn} />
      <Stack.Screen name="(app)" redirect={!isLoggedIn} />
    </Stack>
  );
}
```

**Alternative — redirect in the screen itself:**
```tsx
// app/index.tsx — landing/redirect screen
import { Redirect } from 'expo-router';
import { useAuth } from '@/hooks/useAuth';

export default function Index() {
  const { isLoggedIn } = useAuth();

  return isLoggedIn
    ? <Redirect href="/(app)/(tabs)/" />
    : <Redirect href="/(auth)/login" />;
}
```

**After login — navigate and clear history:**
```tsx
// After successful login, replace so user can't back-navigate to login
router.replace('/(app)/(tabs)/');

// After logout — clear the entire stack
router.replace('/(auth)/login');
```

---

### 2.4 Deep Linking

Deep links open a specific screen in your app from a URL, notification, or another app.

**Configure scheme in `app.json`:**
```json
{
  "expo": {
    "scheme": "myapp",
    "ios": { "bundleIdentifier": "com.yourname.myapp" },
    "android": { "package": "com.yourname.myapp" }
  }
}
```

With this config, Expo Router automatically maps:
```
myapp://product/123       →  app/product/[id].tsx  (id = '123')
myapp://settings          →  app/settings/index.tsx
myapp://user/nawaz        →  app/user/[username].tsx
```

**Test deep links:**
```bash
# iOS Simulator
xcrun simctl openurl booted "myapp://product/123"

# Android Emulator
adb shell am start -W -a android.intent.action.VIEW -d "myapp://product/123"

# Expo CLI
npx uri-scheme open myapp://product/123 --ios
npx uri-scheme open myapp://product/123 --android
```

**Handle deep link params in the screen:**
```tsx
// app/product/[id].tsx — no extra code needed
// Expo Router parses the URL and injects the param automatically
import { useLocalSearchParams } from 'expo-router';

export default function ProductScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();
  // Works for both in-app navigation and deep links
}
```

---

### 2.5 Universal Links / App Links

Universal Links (iOS) and App Links (Android) use real `https://` URLs to open your app — they fall back to the website if the app isn't installed.

```
https://yourapp.com/product/123  →  opens app if installed, else opens website
```

**Step 1 — Host an AASA file (iOS):**
```json
// https://yourapp.com/.well-known/apple-app-site-association
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "TEAMID.com.yourname.myapp",
        "paths": ["/product/*", "/user/*", "/settings"]
      }
    ]
  }
}
```

**Step 2 — Host an assetlinks file (Android):**
```json
// https://yourapp.com/.well-known/assetlinks.json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.yourname.myapp",
    "sha256_cert_fingerprints": ["YOUR_CERT_FINGERPRINT"]
  }
}]
```

**Step 3 — Configure `app.json`:**
```json
{
  "expo": {
    "ios": {
      "associatedDomains": ["applinks:yourapp.com"]
    },
    "android": {
      "intentFilters": [
        {
          "action": "VIEW",
          "autoVerify": true,
          "data": [{ "scheme": "https", "host": "yourapp.com" }],
          "category": ["BROWSABLE", "DEFAULT"]
        }
      ]
    }
  }
}
```

> Expo Router handles the URL-to-screen mapping automatically — you only need to host the domain files and configure `app.json`.

---

### 2.6 useNavigation Hook

Gives access to the underlying React Navigation `navigation` object — useful for imperative actions not directly available via `router`.

```tsx
import { useNavigation } from 'expo-router';
import { useEffect } from 'react';

export default function ProductScreen() {
  const navigation = useNavigation();

  useEffect(() => {
    // Dynamically set the header title after data loads
    navigation.setOptions({ title: 'Nike Air Max' });
  }, []);

  return (/* ... */);
}
```

**Common uses:**
```tsx
navigation.setOptions({ title: 'Dynamic Title', headerRight: () => <SaveButton /> });
navigation.goBack();
navigation.canGoBack();  // returns boolean
navigation.getParent();  // access parent navigator
```

> For most navigation, prefer `router` from `expo-router`. Use `useNavigation` only when you need low-level React Navigation access like `setOptions` or `getParent`.

---

### 2.7 useRoute Hook

Access the current route's name and params. In Expo Router, `useLocalSearchParams` is preferred, but `useRoute` is available for compatibility.

```tsx
import { useRoute } from '@react-navigation/native';

export default function Screen() {
  const route = useRoute();

  console.log(route.name);    // e.g., 'product/[id]'
  console.log(route.params);  // e.g., { id: '42' }

  return (/* ... */);
}
```

> In Expo Router, prefer `useLocalSearchParams` over `useRoute` — it's typed and integrates with Expo Router's param system.

---

### 2.8 useFocusEffect

Runs a callback whenever the screen comes into focus (similar to `useEffect` but tied to navigation state). Runs cleanup when the screen loses focus.

```tsx
import { useFocusEffect } from 'expo-router';
import { useCallback } from 'react';
import { BackHandler } from 'react-native';

export default function CartScreen() {
  // Refresh cart data every time user navigates to this screen
  useFocusEffect(
    useCallback(() => {
      fetchCartItems();

      return () => {
        // Cleanup when screen loses focus
        cancelPendingRequests();
      };
    }, [])
  );

  return (/* ... */);
}
```

**`useFocusEffect` vs `useEffect`:**

| | `useEffect` | `useFocusEffect` |
|---|---|---|
| Runs on | Component mount | Every time screen gains focus |
| Re-runs when | Dependencies change | User navigates back to screen |
| Cleanup | On unmount | When screen loses focus |
| Use for | One-time setup | Refresh data, resume timers, analytics |

```tsx
// Track screen analytics
useFocusEffect(
  useCallback(() => {
    analytics.logScreen('CartScreen');
  }, [])
);
```

---

## 3. Header & UI Customization

### 3.1 Screen Options

Control header appearance through `options` on `Stack.Screen`. Can be set in the layout or inside the screen file itself.

**In the layout file:**
```tsx
// app/(tabs)/shop/_layout.tsx
import { Stack } from 'expo-router';

export default function ShopLayout() {
  return (
    <Stack
      screenOptions={{            // applies to all screens in this stack
        headerStyle: { backgroundColor: '#007AFF' },
        headerTintColor: '#fff',
        headerTitleStyle: { fontFamily: 'Inter-Bold' },
      }}
    >
      <Stack.Screen name="index" options={{ title: 'Shop' }} />
      <Stack.Screen name="[id]" options={{ headerBackTitle: 'Back' }} />
    </Stack>
  );
}
```

**In the screen file (dynamic options):**
```tsx
// app/product/[id].tsx
import { Stack } from 'expo-router';

export default function ProductScreen() {
  const { name } = useProductData();

  return (
    <>
      <Stack.Screen
        options={{
          title: name ?? 'Product',
          headerRight: () => <ShareButton />,
          headerBackVisible: true,
        }}
      />
      {/* screen content */}
    </>
  );
}
```

**Common screen options:**

| Option | Type | Description |
|---|---|---|
| `title` | string | Header title text |
| `headerShown` | boolean | Show/hide the header |
| `headerStyle` | object | Header background style |
| `headerTintColor` | string | Back button and title color |
| `headerTitleStyle` | object | Title text style |
| `headerBackTitle` | string | iOS back button label |
| `headerBackVisible` | boolean | Show/hide back button |
| `headerRight` | component | Right side of header |
| `headerLeft` | component | Left side of header (replaces back btn) |
| `presentation` | string | `'modal'`, `'card'`, etc. |
| `animation` | string | `'slide_from_right'`, `'fade'`, etc. |
| `gestureEnabled` | boolean | Swipe-to-go-back |

---

### 3.2 Custom Headers

Replace the default header entirely with your own component.

```tsx
// app/home/_layout.tsx
import { Stack } from 'expo-router';
import CustomHeader from '@/components/CustomHeader';

export default function HomeLayout() {
  return (
    <Stack
      screenOptions={{
        header: ({ navigation, route, options }) => (
          <CustomHeader title={options.title} canGoBack={navigation.canGoBack()} />
        ),
      }}
    />
  );
}
```

```tsx
// src/components/CustomHeader.tsx
import { View, Text, Pressable, StyleSheet } from 'react-native';
import { useSafeAreaInsets } from 'react-native-safe-area-context';
import { router } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';

type Props = {
  title?: string;
  canGoBack?: boolean;
};

export default function CustomHeader({ title, canGoBack }: Props) {
  const insets = useSafeAreaInsets();

  return (
    <View style={[styles.container, { paddingTop: insets.top }]}>
      {canGoBack && (
        <Pressable onPress={() => router.back()} style={styles.backButton}>
          <Ionicons name="arrow-back" size={24} color="#111" />
        </Pressable>
      )}
      <Text style={styles.title}>{title}</Text>
      <View style={styles.backButton} /> {/* spacer to center title */}
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    alignItems: 'center',
    paddingHorizontal: 16,
    paddingBottom: 12,
    backgroundColor: '#fff',
    borderBottomWidth: StyleSheet.hairlineWidth,
    borderBottomColor: '#e5e5e5',
  },
  backButton: { width: 40 },
  title: { flex: 1, textAlign: 'center', fontSize: 17, fontWeight: '600' },
});
```

---

### 3.3 Tab Bar Customization

Customize the bottom tab bar in the `(tabs)/_layout.tsx`.

```tsx
import { Tabs } from 'expo-router';
import { Platform, View, Text } from 'react-native';
import { Ionicons } from '@expo/vector-icons';
import { useSafeAreaInsets } from 'react-native-safe-area-context';
import { useTheme } from '@/theme/ThemeContext';

export default function TabsLayout() {
  const theme = useTheme();
  const insets = useSafeAreaInsets();

  return (
    <Tabs
      screenOptions={{
        headerShown: false,
        tabBarActiveTintColor: theme.colors.primary,
        tabBarInactiveTintColor: '#999',
        tabBarStyle: {
          backgroundColor: theme.colors.background,
          borderTopWidth: StyleSheet.hairlineWidth,
          borderTopColor: theme.colors.border,
          paddingBottom: insets.bottom,        // respect home indicator
          height: 56 + insets.bottom,
        },
        tabBarLabelStyle: {
          fontSize: 11,
          fontFamily: 'Inter-Medium',
        },
      }}
    >
      <Tabs.Screen
        name="index"
        options={{
          title: 'Home',
          tabBarIcon: ({ color, focused }) => (
            <Ionicons
              name={focused ? 'home' : 'home-outline'}
              size={24}
              color={color}
            />
          ),
          // Custom badge
          tabBarBadge: 3,
          tabBarBadgeStyle: { backgroundColor: 'red' },
        }}
      />
      <Tabs.Screen
        name="profile"
        options={{
          title: 'Profile',
          tabBarIcon: ({ color, focused }) => (
            <Ionicons
              name={focused ? 'person' : 'person-outline'}
              size={24}
              color={color}
            />
          ),
        }}
      />
    </Tabs>
  );
}
```

**Custom tab bar component** — replace the entire tab bar:
```tsx
<Tabs
  tabBar={(props) => <CustomTabBar {...props} />}
>
```

```tsx
// src/components/CustomTabBar.tsx
import { BottomTabBarProps } from '@react-navigation/bottom-tabs';
import { View, Pressable, Text, StyleSheet } from 'react-native';
import { useSafeAreaInsets } from 'react-native-safe-area-context';

export default function CustomTabBar({ state, descriptors, navigation }: BottomTabBarProps) {
  const insets = useSafeAreaInsets();

  return (
    <View style={[styles.container, { paddingBottom: insets.bottom }]}>
      {state.routes.map((route, index) => {
        const { options } = descriptors[route.key];
        const isFocused = state.index === index;

        return (
          <Pressable
            key={route.key}
            onPress={() => navigation.navigate(route.name)}
            style={styles.tab}
          >
            {options.tabBarIcon?.({ color: isFocused ? '#007AFF' : '#999', focused: isFocused, size: 24 })}
            <Text style={{ color: isFocused ? '#007AFF' : '#999', fontSize: 11 }}>
              {options.title ?? route.name}
            </Text>
          </Pressable>
        );
      })}
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    backgroundColor: '#fff',
    borderTopWidth: StyleSheet.hairlineWidth,
    borderTopColor: '#e5e5e5',
  },
  tab: { flex: 1, alignItems: 'center', paddingTop: 8, gap: 4 },
});
```

---

### 3.4 Back Button Handling

**Customize the back button label (iOS only):**
```tsx
<Stack.Screen
  options={{
    headerBackTitle: 'Back',       // default label
    headerBackTitle: 'Products',   // custom label
    headerBackVisible: false,      // hide entirely
  }}
/>
```

**Custom back button component:**
```tsx
<Stack.Screen
  options={{
    headerLeft: () => (
      <Pressable onPress={() => router.back()} style={{ padding: 8 }}>
        <Ionicons name="arrow-back" size={24} color="#007AFF" />
      </Pressable>
    ),
  }}
/>
```

**Intercept back navigation** — prompt user before leaving a form:
```tsx
import { useNavigation } from 'expo-router';
import { useEffect } from 'react';
import { Alert } from 'react-native';

export default function EditProfileScreen() {
  const navigation = useNavigation();
  const [hasUnsavedChanges, setHasUnsavedChanges] = useState(false);

  useEffect(() => {
    const unsubscribe = navigation.addListener('beforeRemove', (e) => {
      if (!hasUnsavedChanges) return; // allow back if no changes

      e.preventDefault(); // block the default back action

      Alert.alert(
        'Discard changes?',
        'You have unsaved changes. Are you sure you want to leave?',
        [
          { text: 'Stay', style: 'cancel' },
          { text: 'Discard', style: 'destructive', onPress: () => navigation.dispatch(e.data.action) },
        ]
      );
    });

    return unsubscribe;
  }, [navigation, hasUnsavedChanges]);
}
```

---

### 3.5 Hardware Back Button (Android)

Android devices have a hardware or gesture-based back button. By default it goes back in the stack. Override it with `BackHandler`.

```tsx
import { BackHandler, Alert } from 'react-native';
import { useFocusEffect } from 'expo-router';
import { useCallback } from 'react';

export default function HomeScreen() {
  useFocusEffect(
    useCallback(() => {
      const onBackPress = () => {
        Alert.alert(
          'Exit App',
          'Are you sure you want to exit?',
          [
            { text: 'Cancel', style: 'cancel', onPress: () => {} },
            { text: 'Exit', style: 'destructive', onPress: () => BackHandler.exitApp() },
          ]
        );
        return true; // return true = we handled the back press (prevent default)
      };

      BackHandler.addEventListener('hardwareBackPress', onBackPress);
      return () => BackHandler.removeEventListener('hardwareBackPress', onBackPress);
    }, [])
  );

  return (/* HomeScreen content */);
}
```

> The handler is wrapped in `useFocusEffect` so it only intercepts back presses while the screen is active. When you navigate away, the cleanup removes the listener automatically.

**Return values from the handler:**
```
return true   → you handled it, do nothing else (prevents default back)
return false  → you didn't handle it, continue with default back behavior
```

---

*End of Module 4*
