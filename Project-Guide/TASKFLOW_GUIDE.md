# TaskFlow — Complete React Native Developer Guide

This is your complete reference for the TaskFlow project. Every concept, pattern, decision, and code snippet used throughout development is documented here, in the same order it was built.

> **Stack:** Expo SDK 54 · React 19.1 · React Native 0.81.5 · TypeScript strict · Expo Router v4 · NativeWind v4 · Zustand v5 · TanStack Query v5 · React Hook Form v7 · Zod v3 · MMKV · Firebase Auth + FCM · Reanimated v3 · FlashList v2 · EAS Build/Update

---

## 📑 Table of Contents

### 🚀 Sprint 0 — Foundation Setup
1. [Project Initialization](#1-project-initialization)
2. [TypeScript Strict Mode](#2-typescript-strict-mode)
3. [Path Aliases — Why Two Configs](#3-path-aliases--why-two-configs)
4. [ESLint Configuration](#4-eslint-configuration)
5. [Prettier Configuration](#5-prettier-configuration)
6. [Husky & lint-staged](#6-husky--lint-staged)
7. [Folder Architecture](#7-folder-architecture)
8. [Expo Router Setup](#8-expo-router-setup)
9. [Route Tree Structure](#9-route-tree-structure)
10. [Barrel Exports](#10-barrel-exports)

### 🎨 Sprint 1 — UI Foundation
11. [NativeWind v4 Installation](#11-nativewind-v4-installation)
12. [Tailwind Config & Design Tokens](#12-tailwind-config--design-tokens)
13. [Global CSS & CSS Variables](#13-global-css--css-variables)
14. [Babel Config for NativeWind](#14-babel-config-for-nativewind)
15. [Metro Config for NativeWind](#15-metro-config-for-nativewind)
16. [Theme Store with Zustand](#16-theme-store-with-zustand)
17. [Dark Mode Wiring in Root Layout](#17-dark-mode-wiring-in-root-layout)
18. [Raw Theme Constants](#18-raw-theme-constants)
19. [The cn() Utility](#19-the-cn-utility)
20. [The Component Pattern](#20-the-component-pattern)
21. [Text Component](#21-text-component)
22. [Button Component](#22-button-component)
23. [Input Component](#23-input-component)
24. [Card Component](#24-card-component)
25. [Avatar Component](#25-avatar-component)
26. [Badge Component](#26-badge-component)
27. [Divider Component](#27-divider-component)
28. [Spinner Component](#28-spinner-component)
29. [Screen Component](#29-screen-component)
30. [Safe Areas & Status Bar](#30-safe-areas--status-bar)
31. [useResponsive Hook](#31-useresponsive-hook)

### 🗺️ Sprint 2 — Navigation
32. [Tab Bar with Icons](#32-tab-bar-with-icons)
33. [Theme-Aware Headers](#33-theme-aware-headers)
34. [Auth Stack Layout](#34-auth-stack-layout)
35. [Modal Routes](#35-modal-routes)
36. [Dynamic Routes](#36-dynamic-routes)
37. [ROUTES Constants](#37-routes-constants)
38. [Programmatic Navigation](#38-programmatic-navigation)
39. [Deep Linking](#39-deep-linking)

### 🗄️ Sprint 3 — State Management
40. [Zustand Store Pattern](#40-zustand-store-pattern)
41. [Persist Middleware & Storage Adapter](#41-persist-middleware--storage-adapter)
42. [Auth Types](#42-auth-types)
43. [Auth Store](#43-auth-store)
44. [UI Store](#44-ui-store)
45. [Selector Pattern](#45-selector-pattern)
46. [Auth Selector Hooks](#46-auth-selector-hooks)

### 🌐 Sprint 4 — API Layer
47. [TanStack Query Setup](#47-tanstack-query-setup)
48. [Query Key Factory](#48-query-key-factory)
49. [Axios Client](#49-axios-client)
50. [Request Interceptor — Auth Token](#50-request-interceptor--auth-token)
51. [Response Interceptor — Error Handling](#51-response-interceptor--error-handling)
52. [Zod API Response Validation](#52-zod-api-response-validation)
53. [Custom Error Classes](#53-custom-error-classes)
54. [safeRequest Wrapper](#54-saferequest-wrapper)

### 🔐 Sprint 5 — Authentication
55. [Firebase Project Setup](#55-firebase-project-setup)
56. [Auth Form Zod Schemas](#56-auth-form-zod-schemas)
57. [Login Screen — React Hook Form](#57-login-screen--react-hook-form)
58. [Signup Screen — Cross-field Validation](#58-signup-screen--cross-field-validation)
59. [Forgot Password Screen](#59-forgot-password-screen)
60. [Firebase Auth Service](#60-firebase-auth-service)
61. [Wiring Forms to Firebase](#61-wiring-forms-to-firebase)
62. [Firebase Error Code Mapping](#62-firebase-error-code-mapping)
63. [Protected Routes Pattern](#63-protected-routes-pattern)
64. [Secure Token Storage](#64-secure-token-storage)
65. [Token Refresh Flow](#65-token-refresh-flow)

### ✅ Sprint 6 — Core Features
66. [Supabase Setup](#66-supabase-setup)
67. [Task Types & Zod Schemas](#67-task-types--zod-schemas)
68. [Tasks API Service](#68-tasks-api-service)
69. [useTasksQuery — TanStack Query](#69-usetasksquery--tanstack-query)
70. [FlashList Task List](#70-flashlist-task-list)
71. [TaskCard Component](#71-taskcard-component)
72. [Empty State Component](#72-empty-state-component)
73. [Error State Component](#73-error-state-component)
74. [Skeleton Loading](#74-skeleton-loading)
75. [Pull-to-Refresh](#75-pull-to-refresh)
76. [Create Task — useMutation](#76-create-task--usemutation)
77. [Optimistic Updates](#77-optimistic-updates)
78. [Task Detail Screen](#78-task-detail-screen)
79. [Edit Task](#79-edit-task)
80. [Delete Task with Confirmation](#80-delete-task-with-confirmation)
81. [Search with Debounce](#81-search-with-debounce)
82. [Filter & Sort](#82-filter--sort)
83. [Infinite Scroll — useInfiniteQuery](#83-infinite-scroll--useinfinitequery)

### 📷 Sprint 7 — Native Features
84. [Permissions Handling Pattern](#84-permissions-handling-pattern)
85. [Image Picker — Gallery](#85-image-picker--gallery)
86. [Camera Capture](#86-camera-capture)
87. [Image Compression Before Upload](#87-image-compression-before-upload)
88. [File Upload to Supabase Storage](#88-file-upload-to-supabase-storage)
89. [Upload Progress Tracking](#89-upload-progress-tracking)
90. [expo-image for Rendering Attachments](#90-expo-image-for-rendering-attachments)

### 💾 Sprint 8 — Offline Support
91. [MMKV Storage Swap](#91-mmkv-storage-swap)
92. [TanStack Query Persistence](#92-tanstack-query-persistence)
93. [Network Status Hook](#93-network-status-hook)
94. [Offline Banner Component](#94-offline-banner-component)
95. [Offline Mutation Queue](#95-offline-mutation-queue)
96. [Cache Invalidation on Reconnect](#96-cache-invalidation-on-reconnect)

### 🔔 Sprint 9 — Push Notifications
97. [FCM Setup — Android](#97-fcm-setup--android)
98. [APNs Setup — iOS](#98-apns-setup--ios)
99. [Request Notification Permission](#99-request-notification-permission)
100. [FCM Token Registration & Refresh](#100-fcm-token-registration--refresh)
101. [Foreground Notification Handler](#101-foreground-notification-handler)
102. [Background Notification Handler](#102-background-notification-handler)
103. [Notification Tap — Deep Link Navigation](#103-notification-tap--deep-link-navigation)
104. [Local Notifications & Scheduling](#104-local-notifications--scheduling)
105. [Cancel Scheduled Notifications](#105-cancel-scheduled-notifications)

### ✨ Sprint 10 — Polish & Animations
106. [Reanimated v3 Fundamentals](#106-reanimated-v3-fundamentals)
107. [Worklets — Running on UI Thread](#107-worklets--running-on-ui-thread)
108. [Swipe-to-Delete with Gesture Handler](#108-swipe-to-delete-with-gesture-handler)
109. [List Item Enter/Exit Animation](#109-list-item-enterexit-animation)
110. [Button Press Spring Animation](#110-button-press-spring-animation)
111. [Accessibility Audit](#111-accessibility-audit)

### 🏗️ Sprint 11 — Production
112. [Multi-Environment Setup](#112-multi-environment-setup)
113. [app.config.ts — Dynamic Config](#113-appconfigts--dynamic-config)
114. [Typed Environment Variables](#114-typed-environment-variables)
115. [Sentry Integration](#115-sentry-integration)
116. [Error Boundaries](#116-error-boundaries)
117. [Analytics with PostHog](#117-analytics-with-posthog)
118. [Jest & React Native Testing Library](#118-jest--react-native-testing-library)
119. [Maestro E2E Testing](#119-maestro-e2e-testing)
120. [Performance Audit](#120-performance-audit)
121. [EAS Build Profiles](#121-eas-build-profiles)
122. [EAS Update — OTA](#122-eas-update--ota)
123. [Store Submission Prep](#123-store-submission-prep)

---

## 1. Project Initialization

Create a new Expo app with blank TypeScript template:

```bash
npx create-expo-app@latest TaskFlow --template blank-typescript
cd TaskFlow
npx expo start
```

Press `a` for Android emulator or scan QR code with Expo Go on iPhone.

**Confirmed stack after initialization:**
- Expo SDK: `54.x`
- React: `19.1.0`
- React Native: `0.81.5`
- New Architecture: enabled by default (`newArchEnabled: true` in `app.json`)

Verify the app boots and shows the default welcome screen before proceeding.

---

## 2. TypeScript Strict Mode

Strict mode catches entire categories of bugs at compile time instead of runtime. Update `tsconfig.json`:

```json
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@/components/*": ["src/components/*"],
      "@/hooks/*": ["src/hooks/*"],
      "@/services/*": ["src/services/*"],
      "@/store/*": ["src/store/*"],
      "@/lib/*": ["src/lib/*"],
      "@/types/*": ["src/types/*"],
      "@/constants/*": ["src/constants/*"],
      "@/features/*": ["src/features/*"]
    }
  },
  "include": [
    "**/*.ts",
    "**/*.tsx",
    ".expo/types/**/*.ts",
    "expo-env.d.ts",
    "nativewind-env.d.ts"
  ]
}
```

**What each strict flag catches:**

| Flag | What it prevents |
|---|---|
| `strict` | Enables 7+ strict checks including `noImplicitAny` and `strictNullChecks` |
| `noUncheckedIndexedAccess` | `arr[0]` could be undefined — forces you to check before using |
| `noImplicitOverride` | Must write `override` keyword when overriding class methods |
| `noFallthroughCasesInSwitch` | Missing `break` in switch = compile error |
| `forceConsistentCasingInFileNames` | `import Button` vs `import button` mismatch caught across OS |

**Why `.expo/types/**/*.ts` in include:**
Expo auto-generates TypeScript types when you run `expo start`. These include types for Expo Router routes and env variables. Without this line, the editor won't pick them up.

**Why `expo-env.d.ts` in include:**
Expo creates this file to declare types for static asset imports like `require('./assets/icon.png')`. Without it, importing images causes type errors.

Verify: `npx tsc --noEmit` should output nothing (zero errors).

---

## 3. Path Aliases — Why Two Configs

Path aliases let you write:
```typescript
import { Button } from '@/components/ui';
```
Instead of:
```typescript
import { Button } from '../../../components/ui/Button/Button';
```

**The critical insight: two separate systems need to know about aliases.**

**System 1: TypeScript** (`tsconfig.json`)
TypeScript checks your code *before* it runs. It uses `paths` in `tsconfig.json` to understand what `@/` means. This gives you autocomplete and catches import errors in your editor.

**System 2: Babel** (`babel.config.js`)
When your app runs on a device, TypeScript is gone — the device runs plain JavaScript. Babel is the tool that transforms your TypeScript into JavaScript. Babel reads `babel.config.js` separately and has no idea what TypeScript's `paths` say. Without `babel-plugin-module-resolver`, the device sees `@/components/ui` and crashes because it cannot resolve `@`.

**Both configs must match exactly.** TypeScript makes the editor happy. Babel makes the app run.

```bash
npm install --save-dev babel-plugin-module-resolver
```

`babel.config.js`:
```javascript
module.exports = function (api) {
  api.cache(true);
  return {
    presets: [
      ['babel-preset-expo', { jsxImportSource: 'nativewind' }],
      'nativewind/babel',
    ],
    plugins: [
      [
        'module-resolver',
        {
          root: ['./'],
          alias: {
            '@': './src',
            '@/components': './src/components',
            '@/hooks': './src/hooks',
            '@/services': './src/services',
            '@/store': './src/store',
            '@/lib': './src/lib',
            '@/types': './src/types',
            '@/constants': './src/constants',
            '@/features': './src/features',
          },
        },
      ],
      'react-native-reanimated/plugin', // MUST BE LAST
    ],
  };
};
```

**Test:** Create `src/lib/test.ts` with `export const testAlias = 'works'`. Import it in a screen via `import { testAlias } from '@/lib/test'`. If the screen renders "works", both systems are aligned. If the editor shows no error but the app crashes, only TypeScript was configured.

After any Babel change: `npx expo start --clear` to bust Metro's cache.

---

## 4. ESLint Configuration

```bash
npx expo lint  # installs eslint + eslint-config-expo, creates eslint.config.js
npm install --save-dev eslint-config-prettier eslint-plugin-prettier
```

`eslint.config.js` (flat config — ESLint 9 modern format):
```javascript
const { defineConfig } = require('eslint/config');
const expo = require('eslint-config-expo/flat');

module.exports = defineConfig([
  expo,
  {
    rules: {
      'prettier/prettier': 'error',
      '@typescript-eslint/no-unused-vars': 'error',
      '@typescript-eslint/no-explicit-any': 'warn',
      'react-hooks/exhaustive-deps': 'warn',
    },
    ignores: ['node_modules/', '.expo/', 'dist/'],
  },
]);
```

**Why `eslint-config-prettier` last in extends:** Prettier and ESLint can conflict on formatting rules (quotes, semicolons). `eslint-config-prettier` disables all ESLint rules that Prettier handles. It must be last so it overrides whatever came before.

Run `npx eslint .` to verify zero errors.

---

## 5. Prettier Configuration

`.prettierrc`:
```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "arrowParens": "always",
  "endOfLine": "lf",
  "bracketSpacing": true,
  "bracketSameLine": false,
  "plugins": ["prettier-plugin-tailwindcss"],
  "tailwindAttributes": ["className"]
}
```

`.prettierignore`:
```
node_modules/
.expo/
dist/
android/
ios/
*.lock
*.log
```

```bash
npm install --save-dev prettier-plugin-tailwindcss
```

`prettier-plugin-tailwindcss` auto-sorts Tailwind classes into a consistent order on every save. This eliminates class order as a code review concern and prevents merge conflicts caused by different developers ordering classes differently.

---

## 6. Husky & lint-staged

Husky runs quality checks automatically before every Git commit. If any check fails, the commit is blocked — the bad code never enters Git history.

```bash
npm install --save-dev husky lint-staged
npx husky init
```

`.husky/pre-commit`:
```bash
npx lint-staged
```

`package.json` (add at top level):
```json
"lint-staged": {
  "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
  "*.{json,md}": ["prettier --write"]
}
```

Add scripts to `package.json`:
```json
"scripts": {
  "start": "expo start",
  "android": "expo start --android",
  "ios": "expo start --ios",
  "lint": "eslint .",
  "lint:fix": "eslint . --fix",
  "format": "prettier --write .",
  "typecheck": "tsc --noEmit",
  "prepare": "husky"
}
```

**`"prepare": "husky"`** — runs automatically after `npm install`. This ensures any developer who clones the repo gets the pre-commit hooks installed automatically without a manual step.

**Testing Husky works:**
```bash
echo "const unused = 'test';" >> App.tsx
git add App.tsx
git commit -m "test"  # Should be BLOCKED by lint-staged
```
Expected: commit blocked with "no-unused-vars" error. Remove the test line, commit again — should succeed.

**Why Husky:** You can't rely on memory or discipline at scale. A gate that runs automatically removes entire classes of human error. Every commit in your history is guaranteed to be lint-clean.

---

## 7. Folder Architecture

TaskFlow uses **feature-based architecture with shared layers**. Two sources of code:
- `app/` — route files only (Expo Router). Every file here = a URL.
- `src/` — everything else. No routing concerns.

```
app/                    ← Route files. These ARE your URLs.
src/
├── components/
│   ├── ui/             ← Design system primitives (Button, Input, Card...)
│   ├── common/         ← App-specific shared components (EmptyState, NetworkBanner)
│   └── layouts/        ← Page-level wrappers (Screen, ModalLayout)
├── features/           ← Feature modules
│   └── tasks/
│       ├── components/ ← Used only within this feature
│       ├── hooks/      ← Feature-specific hooks
│       ├── screens/    ← Screen implementations (mounted by app/ files)
│       ├── services/   ← Feature API calls
│       ├── types.ts    ← Feature types
│       └── index.ts    ← Public exports
├── hooks/              ← Shared hooks (useDebounce, useResponsive, useNetworkStatus)
├── services/
│   ├── api/            ← Axios client, interceptors, error classes
│   ├── storage/        ← MMKV wrapper, SecureStore wrapper
│   └── auth/           ← Firebase auth plumbing
├── store/              ← Zustand global stores
├── lib/                ← Pure utility functions (no React, no side effects)
├── types/              ← Shared TypeScript types (used in 3+ places)
└── constants/          ← Theme tokens, ROUTES, enums
```

**Why feature-based over layer-based:**
Layer-based puts all components together, all services together, etc. When you work on the "tasks" feature, related files are scattered across 5 folders. Feature-based collocates everything — deleting a feature = deleting one folder.

**Dependency direction (never reverse this):**
```
app/ → features/ → services/ → lib/
```
Features can import from services and lib. Services can import from lib. Nothing imports from features or app.

**Decision tree — where does new code go:**
```
Route mount (thin file)?          → app/<route>.tsx
Screen implementation (logic)?    → features/<feature>/screens/
Pure function (no React)?         → src/lib/
Generic UI primitive?             → src/components/ui/
Feature component?                → features/<feature>/components/
Used in 2+ features?              → src/components/common/
Screen wrapper?                   → src/components/layouts/
Hook used in one feature?         → features/<feature>/hooks/
Hook shared across features?      → src/hooks/
API / storage / auth plumbing?    → src/services/<subfolder>/
Global state?                     → src/store/
Feature state?                    → features/<feature>/store/
Type used in 3+ places?           → src/types/
Feature-specific type?            → features/<feature>/types.ts
Constant?                         → src/constants/
```

---

## 8. Expo Router Setup

Expo Router provides file-based routing (like Next.js App Router). Files in `app/` define routes. All business logic lives in `src/`.

```bash
npx expo install expo-router react-native-safe-area-context react-native-screens expo-linking expo-constants expo-status-bar
```

Update `package.json`:
```json
"main": "expo-router/entry"
```

Update `app.json`:
```json
{
  "expo": {
    "name": "TaskFlow",
    "slug": "taskflow",
    "scheme": "taskflow",
    "version": "1.0.0",
    "orientation": "portrait",
    "newArchEnabled": true,
    "plugins": ["expo-router"],
    "experiments": {
      "typedRoutes": true
    }
  }
}
```

Delete `App.tsx` and `index.ts` — Expo Router takes over as the entry point.

**`scheme: "taskflow"`** — required for deep linking. Lets URLs like `taskflow://task/123` open the app.

**`typedRoutes: true`** — Expo Router auto-generates TypeScript types for every route file. If you write `router.push('/task/abc')` but no such route exists, TypeScript will error at compile time. Catches broken links before users do.

---

## 9. Route Tree Structure

```
app/
├── _layout.tsx              ← Root layout (wraps entire app)
├── index.tsx                ← / — redirects based on auth state
│
├── (auth)/                  ← Route group (parentheses = no URL impact)
│   ├── _layout.tsx          ← Auth stack navigator
│   ├── login.tsx            ← URL: /login
│   ├── signup.tsx           ← URL: /signup
│   └── forgot-password.tsx  ← URL: /forgot-password
│
└── (app)/                   ← Authenticated route group
    ├── _layout.tsx          ← App stack navigator with theme headers
    │
    ├── (tabs)/              ← Tab group
    │   ├── _layout.tsx      ← Bottom tab navigator
    │   ├── index.tsx        ← URL: / (Tasks tab)
    │   ├── projects.tsx     ← URL: /projects
    │   ├── notifications.tsx← URL: /notifications
    │   └── profile.tsx      ← URL: /profile
    │
    └── task/
        ├── [id].tsx         ← URL: /task/:id  (dynamic)
        └── new.tsx          ← URL: /task/new  (modal)
```

**Key file-based routing concepts:**

**`_layout.tsx`** — wraps all sibling/child routes in a navigator. Every folder that contains screens should have a `_layout.tsx` defining how those screens are navigated (Stack, Tabs, Drawer).

**`(group)/`** — parentheses create a route group. The group name doesn't appear in the URL. `(auth)/login.tsx` creates the URL `/login`, not `/(auth)/login`. Use groups to share layouts without affecting URLs.

**`[param].tsx`** — square brackets = dynamic route. `task/[id].tsx` matches `/task/anything`. The value is accessible via `useLocalSearchParams()`.

**`index.tsx`** — the default/index route for a folder. `(tabs)/index.tsx` is the URL `/`, not `/index`.

**Rule for `app/` files — keep them thin:**
```tsx
// app/(app)/(tabs)/index.tsx — CORRECT (thin route file)
import { TasksListScreen } from '@/features/tasks';
export default TasksListScreen;

// app/(app)/(tabs)/index.tsx — WRONG (logic in route file)
export default function TasksScreen() {
  const [tasks, setTasks] = useState([]);
  // 100 lines of logic...
}
```
Route files should be 1–5 lines. All implementation lives in `src/features/`.

---

## 10. Barrel Exports

A barrel is an `index.ts` that re-exports from a folder, enabling cleaner imports.

```typescript
// Without barrel
import { Button } from '@/components/ui/Button/Button';
import { Input } from '@/components/ui/Input/Input';
import { Card } from '@/components/ui/Card/Card';

// With barrel (src/components/ui/index.ts exports all three)
import { Button, Input, Card } from '@/components/ui';
```

**Benefits:**
- Imports are shorter and more stable
- Moving a file internally doesn't break all importers (only the barrel needs updating)
- Barrel is the public API of a folder — anything not in the barrel is internal

**Empty barrel placeholder** — prevents TypeScript "is not a module" error on empty folders:
```typescript
export {};
```

**`src/components/ui/index.ts`** (filled in as components are built):
```typescript
export { Text } from './Text';
export { Button } from './Button';
export { Input } from './Input';
export { Card } from './Card';
export { Avatar } from './Avatar';
export { Badge } from './Badge';
export { Divider } from './Divider';
export { Spinner } from './Spinner';
export { Screen } from './Screen';
```

**`src/components/index.ts`**:
```typescript
export * from './ui';
export * from './common';
export * from './layouts';
```

Apply the same barrel pattern to every `src/` subfolder.

---

## 11. NativeWind v4 Installation

NativeWind brings Tailwind CSS to React Native. v4 is a complete rewrite — it uses CSS variables, build-time compilation, and true dark mode support.

```bash
npx expo install nativewind react-native-reanimated react-native-safe-area-context
npm install --save-dev tailwindcss@^3.4.17 prettier-plugin-tailwindcss
```

> ⚠️ **Critical:** Install `tailwindcss@^3.4.17` (v3.4.x). NativeWind v4 is built against Tailwind v3.4. Installing Tailwind v4 breaks NativeWind.

Initialize Tailwind config:
```bash
npx tailwindcss init
```

Create TypeScript declaration in project root `nativewind-env.d.ts`:
```typescript
/// <reference types="nativewind/types" />
```

This adds the `className` prop to all React Native components in TypeScript. Without it, `<View className="flex-1">` causes a TypeScript error ("className is not a valid prop").

Add to `tsconfig.json` `include`:
```json
"nativewind-env.d.ts"
```

---

## 12. Tailwind Config & Design Tokens

`tailwind.config.js`:
```javascript
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    './app/**/*.{js,jsx,ts,tsx}',
    './src/**/*.{js,jsx,ts,tsx}',
  ],
  presets: [require('nativewind/preset')],
  darkMode: 'class',
  theme: {
    extend: {
      colors: {
        // Brand — Indigo
        primary: {
          50:  'rgb(var(--color-primary-50) / <alpha-value>)',
          100: 'rgb(var(--color-primary-100) / <alpha-value>)',
          200: 'rgb(var(--color-primary-200) / <alpha-value>)',
          300: 'rgb(var(--color-primary-300) / <alpha-value>)',
          400: 'rgb(var(--color-primary-400) / <alpha-value>)',
          500: 'rgb(var(--color-primary-500) / <alpha-value>)',
          600: 'rgb(var(--color-primary-600) / <alpha-value>)',
          700: 'rgb(var(--color-primary-700) / <alpha-value>)',
          800: 'rgb(var(--color-primary-800) / <alpha-value>)',
          900: 'rgb(var(--color-primary-900) / <alpha-value>)',
        },
        // Semantic tokens — change these and the whole app updates
        background:        'rgb(var(--color-background) / <alpha-value>)',
        foreground:        'rgb(var(--color-foreground) / <alpha-value>)',
        card:              'rgb(var(--color-card) / <alpha-value>)',
        'card-foreground': 'rgb(var(--color-card-foreground) / <alpha-value>)',
        muted:             'rgb(var(--color-muted) / <alpha-value>)',
        'muted-foreground':'rgb(var(--color-muted-foreground) / <alpha-value>)',
        border:            'rgb(var(--color-border) / <alpha-value>)',
        input:             'rgb(var(--color-input) / <alpha-value>)',
        ring:              'rgb(var(--color-ring) / <alpha-value>)',
        // Status
        success: {
          DEFAULT:    'rgb(var(--color-success) / <alpha-value>)',
          foreground: 'rgb(var(--color-success-foreground) / <alpha-value>)',
        },
        warning: {
          DEFAULT:    'rgb(var(--color-warning) / <alpha-value>)',
          foreground: 'rgb(var(--color-warning-foreground) / <alpha-value>)',
        },
        danger: {
          DEFAULT:    'rgb(var(--color-danger) / <alpha-value>)',
          foreground: 'rgb(var(--color-danger-foreground) / <alpha-value>)',
        },
      },
      fontSize: {
        '2xs': ['10px', { lineHeight: '14px' }],
        xs:    ['12px', { lineHeight: '16px' }],
        sm:    ['14px', { lineHeight: '20px' }],
        base:  ['16px', { lineHeight: '24px' }],
        lg:    ['18px', { lineHeight: '28px' }],
        xl:    ['20px', { lineHeight: '28px' }],
        '2xl': ['24px', { lineHeight: '32px' }],
        '3xl': ['30px', { lineHeight: '36px' }],
        '4xl': ['36px', { lineHeight: '40px' }],
      },
      borderRadius: {
        none: '0',
        sm:   '4px',
        DEFAULT: '8px',
        md:   '12px',
        lg:   '16px',
        xl:   '20px',
        '2xl':'24px',
        full: '9999px',
      },
    },
  },
  plugins: [],
};
```

**Why CSS variables for colors:** Color tokens are defined as CSS variable references (`var(--color-background)`). The actual RGB values live in `global.css`. When the theme switches from light to dark, only the CSS variables change — every `bg-background`, `text-foreground`, etc. across the entire app automatically uses the new values. Zero component changes needed for theme switching.

**`<alpha-value>` syntax:** Enables opacity modifiers. `bg-primary-500/50` = primary color at 50% opacity. `bg-background/80` = background at 80% opacity. Without this syntax, you can't use opacity modifiers on custom colors.

**`darkMode: 'class'`:** Dark mode is triggered by the presence of a CSS class on the root element, not by a media query. This gives us programmatic control (user preference persisted in Zustand) rather than just following the OS.

---

## 13. Global CSS & CSS Variables

`src/styles/global.css`:
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    /* Brand — Indigo palette */
    --color-primary-50:  238 242 255;
    --color-primary-100: 224 231 255;
    --color-primary-200: 199 210 254;
    --color-primary-300: 165 180 252;
    --color-primary-400: 129 140 248;
    --color-primary-500: 99 102 241;
    --color-primary-600: 79 70 229;
    --color-primary-700: 67 56 202;
    --color-primary-800: 55 48 163;
    --color-primary-900: 49 46 129;

    /* Light theme semantic tokens */
    --color-background:        255 255 255;
    --color-foreground:        15 23 42;
    --color-card:              255 255 255;
    --color-card-foreground:   15 23 42;
    --color-muted:             241 245 249;
    --color-muted-foreground:  100 116 139;
    --color-border:            226 232 240;
    --color-input:             226 232 240;
    --color-ring:              99 102 241;

    /* Status — Light */
    --color-success:            34 197 94;
    --color-success-foreground: 255 255 255;
    --color-warning:            234 179 8;
    --color-warning-foreground: 15 23 42;
    --color-danger:             239 68 68;
    --color-danger-foreground:  255 255 255;
  }

  .dark:root,
  :root[class~='dark'] {
    /* Dark theme semantic tokens — only what changes */
    --color-background:        15 23 42;
    --color-foreground:        248 250 252;
    --color-card:              30 41 59;
    --color-card-foreground:   248 250 252;
    --color-muted:             30 41 59;
    --color-muted-foreground:  148 163 184;
    --color-border:            51 65 85;
    --color-input:             51 65 85;
    --color-ring:              129 140 248;

    /* Status — Dark (slightly adjusted for dark backgrounds) */
    --color-danger:             248 113 113;
    --color-warning:            250 204 21;
  }
}
```

**RGB values without commas** — `255 255 255` not `255, 255, 255` and not `#ffffff`. This is the format required by `rgb(var(--name) / 1)`. Commas would break the CSS.

**Only override what changes** — the brand primary colors (`--color-primary-*`) are the same in both themes. Brand identity doesn't change. Only semantic background/text/border colors change between themes.

**Import in root layout** — `import '../src/styles/global.css'` in `app/_layout.tsx`. This single import activates NativeWind globally.

---

## 14. Babel Config for NativeWind

Complete `babel.config.js`:
```javascript
module.exports = function (api) {
  api.cache(true);
  return {
    presets: [
      ['babel-preset-expo', { jsxImportSource: 'nativewind' }],
      'nativewind/babel',
    ],
    plugins: [
      [
        'module-resolver',
        {
          root: ['./'],
          alias: {
            '@': './src',
            '@/components': './src/components',
            '@/hooks': './src/hooks',
            '@/services': './src/services',
            '@/store': './src/store',
            '@/lib': './src/lib',
            '@/types': './src/types',
            '@/constants': './src/constants',
            '@/features': './src/features',
          },
        },
      ],
      'react-native-reanimated/plugin', // ← MUST BE LAST
    ],
  };
};
```

**Three non-negotiable rules:**

1. `jsxImportSource: 'nativewind'` on `babel-preset-expo` — required for NativeWind v4. Without it, className is not processed.

2. `nativewind/babel` in `presets`, NOT `plugins` — this is a common mistake. The preset must be in the `presets` array.

3. `react-native-reanimated/plugin` MUST be the last item in `plugins`. It transforms async function calls into Reanimated worklets. If any plugin runs after it, the transformation breaks silently.

After any Babel config change: `npx expo start --clear`

---

## 15. Metro Config for NativeWind

`metro.config.js`:
```javascript
const { getDefaultConfig } = require('expo/metro-config');
const { withNativeWind } = require('nativewind/metro');

const config = getDefaultConfig(__dirname);

module.exports = withNativeWind(config, {
  input: './src/styles/global.css',
});
```

Metro is React Native's JavaScript bundler. `withNativeWind` wraps the default config to:
1. Process your CSS file through the Tailwind compiler at bundle time
2. Convert CSS classes into React Native StyleSheet objects
3. Inject CSS variable values

Without this, NativeWind classes are never compiled and the app renders unstyled.

---

## 16. Theme Store with Zustand

`src/store/theme.store.ts`:
```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import { Appearance } from 'react-native';
import { persistStorage } from '@/services/storage';

export type ThemePreference = 'light' | 'dark' | 'system';
export type ResolvedTheme = 'light' | 'dark';

interface ThemeState {
  preference: ThemePreference;
  resolvedTheme: ResolvedTheme;
  setPreference: (preference: ThemePreference) => void;
}

const getResolvedTheme = (preference: ThemePreference): ResolvedTheme => {
  if (preference === 'system') return Appearance.getColorScheme() ?? 'light';
  return preference;
};

export const useThemeStore = create<ThemeState>()(
  persist(
    (set) => ({
      preference: 'system',
      resolvedTheme: getResolvedTheme('system'),
      setPreference: (preference) =>
        set({ preference, resolvedTheme: getResolvedTheme(preference) }),
    }),
    {
      name: 'taskflow-theme',
      storage: persistStorage,
      partialize: (state) => ({ preference: state.preference }),
      onRehydrateStorage: () => (state) => {
        if (state) state.resolvedTheme = getResolvedTheme(state.preference);
      },
    },
  ),
);

// React to OS theme changes when preference is 'system'
Appearance.addChangeListener(({ colorScheme }) => {
  const { preference } = useThemeStore.getState();
  if (preference === 'system') {
    useThemeStore.setState({ resolvedTheme: colorScheme ?? 'light' });
  }
});
```

**Three preference modes:**
- `light` — always light, ignores OS
- `dark` — always dark, ignores OS
- `system` — follows OS, updates live when user changes OS theme

**`partialize`** — only `preference` is saved to storage. `resolvedTheme` is derived from preference and is recomputed in `onRehydrateStorage` after the app restarts. Never persist derived state.

**`Appearance.addChangeListener`** — listens for OS theme changes at the module level (outside of React). Only fires when preference is `system`.

---

## 17. Dark Mode Wiring in Root Layout

`app/_layout.tsx`:
```tsx
import { Stack } from 'expo-router';
import { useEffect } from 'react';
import { colorScheme } from 'nativewind';
import { SafeAreaProvider } from 'react-native-safe-area-context';
import { useThemeStore } from '@/store';
import '../src/styles/global.css';

export default function RootLayout() {
  const resolvedTheme = useThemeStore((state) => state.resolvedTheme);

  useEffect(() => {
    colorScheme.set(resolvedTheme);
  }, [resolvedTheme]);

  return (
    <SafeAreaProvider>
      <Stack screenOptions={{ headerShown: false }}>
        <Stack.Screen name="index" />
        <Stack.Screen name="(auth)" />
        <Stack.Screen name="(app)" />
        <Stack.Screen name="(dev)" />
      </Stack>
    </SafeAreaProvider>
  );
}
```

**What `colorScheme.set(resolvedTheme)` does:** NativeWind internally adds or removes the `dark` class from the root element. All `dark:` prefixed Tailwind classes activate when the `dark` class is present. By calling this whenever `resolvedTheme` changes in Zustand, the entire app re-themes instantly.

**`SafeAreaProvider` at the root:** Must wrap the entire app. All descendants can now call `useSafeAreaInsets()`. If `Screen` is used inside a tree without `SafeAreaProvider`, it crashes.

**`import '../src/styles/global.css'`** — this single import activates NativeWind. The `../` because `app/` and `src/` are siblings.

---

## 18. Raw Theme Constants

For cases where Tailwind classes can't be used (Reanimated worklets, `ActivityIndicator` color prop, navigation theme objects), raw color values are needed.

`src/constants/theme.ts`:
```typescript
export const colors = {
  light: {
    background:     'rgb(255, 255, 255)',
    foreground:     'rgb(15, 23, 42)',
    card:           'rgb(255, 255, 255)',
    cardForeground: 'rgb(15, 23, 42)',
    muted:          'rgb(241, 245, 249)',
    mutedForeground:'rgb(100, 116, 139)',
    border:         'rgb(226, 232, 240)',
    primary:        'rgb(99, 102, 241)',
    success:        'rgb(34, 197, 94)',
    warning:        'rgb(234, 179, 8)',
    danger:         'rgb(239, 68, 68)',
  },
  dark: {
    background:     'rgb(15, 23, 42)',
    foreground:     'rgb(248, 250, 252)',
    card:           'rgb(30, 41, 59)',
    cardForeground: 'rgb(248, 250, 252)',
    muted:          'rgb(30, 41, 59)',
    mutedForeground:'rgb(148, 163, 184)',
    border:         'rgb(51, 65, 85)',
    primary:        'rgb(129, 140, 248)',
    success:        'rgb(34, 197, 94)',
    warning:        'rgb(250, 204, 21)',
    danger:         'rgb(248, 113, 113)',
  },
} as const;

export const spacing = {
  xs: 4, sm: 8, md: 16, lg: 24, xl: 32, '2xl': 48, '3xl': 64,
} as const;

export const radius = {
  none: 0, sm: 4, DEFAULT: 8, md: 12, lg: 16, xl: 20, '2xl': 24, full: 9999,
} as const;

export type ThemeColors = typeof colors.light;
```

**Rule:** For UI styling, always prefer Tailwind classes. Only use these raw values when you literally cannot use a `className` prop — such as inside Reanimated worklets or when passing a color string to a native component's style prop.

---

## 19. The cn() Utility

The single most important utility in the design system. Every component uses it.

`src/lib/cn.ts`:
```typescript
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

/**
 * Compose Tailwind classes safely.
 * - Filters falsy values
 * - Resolves Tailwind conflicts (last wins)
 * - Handles conditionals, arrays, objects
 */
export function cn(...inputs: ClassValue[]): string {
  return twMerge(clsx(inputs));
}
```

```bash
npm install clsx tailwind-merge
```

**What each library does:**

`clsx` — filters and combines class values of any shape:
```typescript
clsx('base', false && 'hidden', null, undefined)  // → 'base'
clsx('p-4', isLarge && 'p-8')                     // → 'p-4' or 'p-4 p-8'
clsx({ 'bg-primary-500': isPrimary })             // → '' or 'bg-primary-500'
```

`tailwind-merge` — resolves conflicting Tailwind utilities (last wins):
```typescript
twMerge('p-4 p-8')           // → 'p-8'
twMerge('text-red-500 text-blue-500') // → 'text-blue-500'
```

Together they enable safe composition:
```typescript
// Component default: p-4. User override: p-8. Result: p-8.
cn('rounded p-4', className)
// If className = 'p-8', result = 'rounded p-8' (not 'rounded p-4 p-8')
```

Without `tailwind-merge`, passing `className="p-8"` to a component with default `p-4` would produce `p-4 p-8` — both classes in the DOM. In Tailwind, the winner is whichever class appears last in the compiled CSS, not in your JSX. `twMerge` removes the losing class entirely.

---

## 20. The Component Pattern

Every UI component follows the same recipe. Internalize this once and building 50 more components becomes mechanical.

```tsx
// 1. Narrow variant types up front
type ButtonVariant = 'primary' | 'secondary' | 'ghost' | 'danger';
type ButtonSize = 'sm' | 'md' | 'lg';

// 2. Interface extends the native component's props
// This gives you all native props (onPress, hitSlop, testID...) for free
interface ButtonProps extends Omit<PressableProps, 'children'> {
  variant?: ButtonVariant;   // custom prop
  size?: ButtonSize;
  label: string;
  loading?: boolean;
  disabled?: boolean;
  className?: string;        // always accept override
}

// 3. Defaults via destructuring
export function Button({
  variant = 'primary',       // sensible default
  size = 'md',
  label,
  loading = false,
  disabled = false,
  className,
  ...rest                    // capture all native props
}: ButtonProps) {
  const isDisabled = disabled || loading;

  return (
    <Pressable
      // 4. Accessibility always included
      accessibilityRole="button"
      accessibilityLabel={label}
      accessibilityState={{ disabled: isDisabled, busy: loading }}
      disabled={isDisabled}
      // 5. cn() for clean class composition
      className={cn(
        'base-classes always applied',
        variant === 'primary' && 'bg-primary-500 active:bg-primary-600',
        variant === 'danger' && 'bg-danger',
        size === 'sm' && 'h-9 px-3',
        size === 'md' && 'h-11 px-4',
        isDisabled && 'opacity-50',
        className,             // user override goes LAST
      )}
      {...rest}                // 6. spread native props
    >
      <Text>{label}</Text>
    </Pressable>
  );
}
```

**Why `Pressable` over `TouchableOpacity`:**
- `active:bg-primary-600` (Tailwind's active prefix) only works with `Pressable`
- Full press feedback control — not locked into opacity fade
- Plays well with Gesture Handler (used by Reanimated gestures)
- Built-in accessibility state management
- React Native team recommends it — `TouchableOpacity` is legacy

**Why no `forwardRef`:** React 19 treats refs as regular props. The old `forwardRef()` wrapper is no longer needed. Just accept a `ref` prop like any other.

**Why `className` goes last in `cn()`:** `twMerge` resolves conflicts in left-to-right order, last wins. By placing `className` last, user-supplied overrides always beat component defaults.

---

## 21. Text Component

`src/components/ui/Text/Text.tsx`:
```tsx
import { Text as RNText, type TextProps as RNTextProps } from 'react-native';
import { cn } from '@/lib';

type TextVariant =
  | 'h1' | 'h2' | 'h3' | 'h4'
  | 'body-lg' | 'body' | 'body-sm'
  | 'caption' | 'label' | 'overline';

type TextColor =
  | 'foreground' | 'muted' | 'primary'
  | 'success' | 'warning' | 'danger' | 'inherit';

interface TextProps extends RNTextProps {
  variant?: TextVariant;
  color?: TextColor;
  className?: string;
}

export function Text({ variant = 'body', color = 'foreground', className, children, ...rest }: TextProps) {
  return (
    <RNText
      className={cn(
        variant === 'h1'      && 'text-4xl font-bold',
        variant === 'h2'      && 'text-3xl font-bold',
        variant === 'h3'      && 'text-2xl font-semibold',
        variant === 'h4'      && 'text-xl font-semibold',
        variant === 'body-lg' && 'text-lg',
        variant === 'body'    && 'text-base',
        variant === 'body-sm' && 'text-sm',
        variant === 'caption' && 'text-xs',
        variant === 'label'   && 'text-sm font-medium',
        variant === 'overline'&& 'text-2xs uppercase tracking-wider font-semibold',
        color === 'foreground' && 'text-foreground',
        color === 'muted'      && 'text-muted-foreground',
        color === 'primary'    && 'text-primary-500',
        color === 'success'    && 'text-success',
        color === 'warning'    && 'text-warning',
        color === 'danger'     && 'text-danger',
        className,
      )}
      {...rest}
    >
      {children}
    </RNText>
  );
}
```

**Critical React Native rule:** ALL text must be inside a `<Text>` component. Unlike HTML where you can put text anywhere in the DOM, React Native will throw a runtime error for bare strings inside `<View>`. This custom `Text` wraps the native one and adds a variant + color system.

**Why wrap the native `Text`:** Without a wrapper, every text in the app would have inconsistent sizing, weight, and color. By centralizing through this component, changing `h2` styling in one place updates every heading in the entire app.

---

## 22. Button Component

`src/components/ui/Button/Button.tsx`:
```tsx
import { Pressable, ActivityIndicator, View, type PressableProps } from 'react-native';
import type { ReactNode } from 'react';
import { cn } from '@/lib';
import { Text } from '../Text';

type ButtonVariant = 'primary' | 'secondary' | 'ghost' | 'danger';
type ButtonSize = 'sm' | 'md' | 'lg';

interface ButtonProps extends Omit<PressableProps, 'children'> {
  variant?: ButtonVariant;
  size?: ButtonSize;
  label: string;
  loading?: boolean;
  disabled?: boolean;
  leftIcon?: ReactNode;
  rightIcon?: ReactNode;
  fullWidth?: boolean;
  className?: string;
}

export function Button({
  variant = 'primary', size = 'md', label, loading = false,
  disabled = false, leftIcon, rightIcon, fullWidth = false, className, ...rest
}: ButtonProps) {
  const isDisabled = disabled || loading;

  return (
    <Pressable
      accessibilityRole="button"
      accessibilityLabel={label}
      accessibilityState={{ disabled: isDisabled, busy: loading }}
      disabled={isDisabled}
      className={cn(
        'flex-row items-center justify-center rounded-md',
        variant === 'primary'   && 'bg-primary-500 active:bg-primary-600',
        variant === 'secondary' && 'bg-muted active:bg-border',
        variant === 'ghost'     && 'bg-transparent active:bg-muted',
        variant === 'danger'    && 'bg-danger active:opacity-90',
        size === 'sm' && 'h-9 px-3 gap-1.5',
        size === 'md' && 'h-11 px-4 gap-2',
        size === 'lg' && 'h-14 px-6 gap-2',
        fullWidth && 'w-full',
        isDisabled && 'opacity-50',
        className,
      )}
      {...rest}
    >
      {loading ? (
        <ActivityIndicator
          size="small"
          color={variant === 'primary' || variant === 'danger' ? '#fff' : undefined}
        />
      ) : (
        <>
          {leftIcon ? <View>{leftIcon}</View> : null}
          <Text
            className={cn(
              'font-semibold',
              size === 'sm' && 'text-sm',
              size === 'md' && 'text-base',
              size === 'lg' && 'text-lg',
              (variant === 'primary' || variant === 'danger') && 'text-white',
              (variant === 'secondary' || variant === 'ghost') && 'text-foreground',
            )}
          >
            {label}
          </Text>
          {rightIcon ? <View>{rightIcon}</View> : null}
        </>
      )}
    </Pressable>
  );
}
```

**`accessibilityState={{ busy: loading }}`** — when `loading` is true, screen readers announce the button as "busy". Users with VoiceOver/TalkBack know to wait without seeing the spinner.

**`active:bg-primary-600`** — Tailwind's `active:` prefix applies the class only while the button is pressed. This provides press feedback without `useState` or `onPressIn`/`onPressOut` handlers.

---

## 23. Input Component

`src/components/ui/Input/Input.tsx`:
```tsx
import { TextInput, View, type TextInputProps } from 'react-native';
import { useState, type ReactNode } from 'react';
import { cn } from '@/lib';
import { Text } from '../Text';

interface InputProps extends Omit<TextInputProps, 'style'> {
  label?: string;
  helperText?: string;
  error?: string;
  leftIcon?: ReactNode;
  rightIcon?: ReactNode;
  disabled?: boolean;
  className?: string;
  containerClassName?: string;
}

export function Input({
  label, helperText, error, leftIcon, rightIcon, disabled = false,
  className, containerClassName, onFocus, onBlur, ...rest
}: InputProps) {
  const [isFocused, setIsFocused] = useState(false);
  const hasError = Boolean(error);

  return (
    <View className={cn('w-full', containerClassName)}>
      {label ? <Text variant="label" className="mb-1.5">{label}</Text> : null}
      <View className={cn(
        'flex-row items-center rounded-md border h-11 px-3 bg-input/30',
        isFocused && !hasError && 'border-ring',
        !isFocused && !hasError && 'border-border',
        hasError && 'border-danger',
        disabled && 'opacity-50',
      )}>
        {leftIcon ? <View className="mr-2">{leftIcon}</View> : null}
        <TextInput
          accessibilityLabel={label}
          accessibilityState={{ disabled }}
          editable={!disabled}
          placeholderTextColor="rgb(148 163 184)"
          onFocus={(e) => { setIsFocused(true); onFocus?.(e); }}
          onBlur={(e) => { setIsFocused(false); onBlur?.(e); }}
          className={cn('flex-1 text-base text-foreground', className)}
          {...rest}
        />
        {rightIcon ? <View className="ml-2">{rightIcon}</View> : null}
      </View>
      {error
        ? <Text variant="caption" color="danger" className="mt-1">{error}</Text>
        : helperText
        ? <Text variant="caption" color="muted" className="mt-1">{helperText}</Text>
        : null}
    </View>
  );
}
```

**Three border states:**
- Default: `border-border` (gray)
- Focused: `border-ring` (primary/indigo)
- Error: `border-danger` (red)

These use CSS variable tokens so they automatically update in dark mode.

**`placeholderTextColor`** must be a raw color string, not a Tailwind class. Tailwind classes don't work on this prop.

**`onFocus?.(e)`** — optional chaining on the forwarded handler. If the parent passed `onFocus`, we call it. If not, nothing happens. We still need to update `isFocused` state regardless.

---

## 24. Card Component

`src/components/ui/Card/Card.tsx`:
```tsx
import { View, type ViewProps } from 'react-native';
import { cn } from '@/lib';

type CardVariant = 'elevated' | 'outlined' | 'flat';

interface CardProps extends ViewProps {
  variant?: CardVariant;
  className?: string;
}

export function Card({ variant = 'elevated', className, children, ...rest }: CardProps) {
  return (
    <View
      className={cn(
        'rounded-lg bg-card p-4',
        variant === 'elevated' && 'shadow-sm',
        variant === 'outlined' && 'border border-border',
        className,
      )}
      {...rest}
    >
      {children}
    </View>
  );
}
```

**Three variants:**
- `elevated` — shadow elevation. iOS renders it natively; Android uses `elevation` prop (NativeWind handles the platform difference).
- `outlined` — border with no shadow. Cleaner, predictable across platforms.
- `flat` — no visual treatment. Used inside already-bordered containers to avoid double borders.

---

## 25. Avatar Component

`src/components/ui/Avatar/Avatar.tsx`:
```tsx
import { View } from 'react-native';
import { Image } from 'expo-image';
import { cn } from '@/lib';
import { Text } from '../Text';

type AvatarSize = 'sm' | 'md' | 'lg' | 'xl';

interface AvatarProps {
  source?: string;
  fallback?: string;
  size?: AvatarSize;
  className?: string;
}

const sizeMap: Record<AvatarSize, { container: string; text: string; pixels: number }> = {
  sm: { container: 'h-8 w-8',   text: 'text-xs', pixels: 32 },
  md: { container: 'h-10 w-10', text: 'text-sm', pixels: 40 },
  lg: { container: 'h-14 w-14', text: 'text-lg', pixels: 56 },
  xl: { container: 'h-20 w-20', text: 'text-2xl', pixels: 80 },
};

export function Avatar({ source, fallback, size = 'md', className }: AvatarProps) {
  const { container, text, pixels } = sizeMap[size];
  const initials = fallback?.slice(0, 2).toUpperCase() ?? '??';

  return (
    <View
      accessibilityRole="image"
      accessibilityLabel={fallback ? `Avatar for ${fallback}` : 'Avatar'}
      className={cn(
        'items-center justify-center overflow-hidden rounded-full bg-muted',
        container,
        className,
      )}
    >
      {source ? (
        <Image
          source={source}
          style={{ width: pixels, height: pixels }}
          contentFit="cover"
          transition={200}
        />
      ) : (
        <Text className={cn('font-semibold text-muted-foreground', text)}>
          {initials}
        </Text>
      )}
    </View>
  );
}
```

**Why `expo-image` over React Native's `Image`:**
- **Disk + memory caching** — downloads once, renders instantly on subsequent mounts
- **`blurhash` placeholder** — show a colored blur while loading (no blank space)
- **`priority` hint** — mark hero images as high priority for preloading
- **`transition`** — smooth fade-in animation when image loads
- **Better error handling** — graceful fallback without crashes

---

## 26. Badge Component

`src/components/ui/Badge/Badge.tsx`:
```tsx
import { View } from 'react-native';
import { cn } from '@/lib';
import { Text } from '../Text';

type BadgeVariant = 'default' | 'primary' | 'success' | 'warning' | 'danger';
type BadgeSize = 'sm' | 'md';

interface BadgeProps {
  label: string;
  variant?: BadgeVariant;
  size?: BadgeSize;
  className?: string;
}

export function Badge({ label, variant = 'default', size = 'sm', className }: BadgeProps) {
  return (
    <View className={cn(
      'self-start rounded-full',
      size === 'sm' && 'px-2 py-0.5',
      size === 'md' && 'px-2.5 py-1',
      variant === 'default' && 'bg-muted',
      variant === 'primary' && 'bg-primary-500',
      variant === 'success' && 'bg-success',
      variant === 'warning' && 'bg-warning',
      variant === 'danger'  && 'bg-danger',
      className,
    )}>
      <Text className={cn(
        'font-medium',
        size === 'sm' && 'text-xs',
        size === 'md' && 'text-sm',
        variant === 'default'              && 'text-foreground',
        variant === 'warning'              && 'text-foreground',
        (variant === 'primary' || variant === 'success' || variant === 'danger') && 'text-white',
      )}>
        {label}
      </Text>
    </View>
  );
}
```

**`self-start`** prevents the badge from stretching to full width when inside a flex container. Without it, a badge inside a `<View className="flex-1">` would stretch across the entire container.

---

## 27. Divider Component

`src/components/ui/Divider/Divider.tsx`:
```tsx
import { View } from 'react-native';
import { cn } from '@/lib';

interface DividerProps {
  orientation?: 'horizontal' | 'vertical';
  className?: string;
}

export function Divider({ orientation = 'horizontal', className }: DividerProps) {
  return (
    <View
      accessibilityRole="none"
      className={cn(
        'bg-border',
        orientation === 'horizontal' && 'h-px w-full',
        orientation === 'vertical'   && 'w-px self-stretch',
        className,
      )}
    />
  );
}
```

**`h-px`** — 1 logical pixel. On Retina/high-DPI screens this renders as a hairline (0.5px physical pixels). Uses `bg-border` token so it automatically changes in dark mode.

**`accessibilityRole="none"`** — tells screen readers this is a decorative element with no meaning. Screen readers will skip over it rather than announcing "horizontal rule" or similar.

---

## 28. Spinner Component

`src/components/ui/Spinner/Spinner.tsx`:
```tsx
import { ActivityIndicator, View } from 'react-native';
import { cn } from '@/lib';
import { useThemeStore } from '@/store';
import { colors } from '@/constants';

type SpinnerSize = 'sm' | 'md' | 'lg';

interface SpinnerProps {
  size?: SpinnerSize;
  color?: string;
  className?: string;
}

const sizeMap: Record<SpinnerSize, 'small' | 'large' | number> = {
  sm: 'small',
  md: 'large',
  lg: 48,
};

export function Spinner({ size = 'md', color, className }: SpinnerProps) {
  const resolvedTheme = useThemeStore((state) => state.resolvedTheme);
  const spinnerColor = color ?? colors[resolvedTheme].primary;

  return (
    <View accessibilityRole="progressbar" className={cn('items-center', className)}>
      <ActivityIndicator size={sizeMap[size]} color={spinnerColor} />
    </View>
  );
}
```

**`ActivityIndicator`** is a native component — it uses the platform's built-in spinner animation (iOS: circular spin, Android: circular progress arc). The animation runs natively and is never affected by JS thread load.

**Why `color` must be a raw string:** `ActivityIndicator`'s `color` prop accepts a color string, not a Tailwind class. That's why we read from the `colors` constant object keyed by `resolvedTheme` — to get the actual `rgb(...)` value.

---

## 29. Screen Component

The `Screen` component is used by every single screen in the app. It centralizes safe area handling, status bar control, scrollability, and padding.

`src/components/ui/Screen/Screen.tsx`:
```tsx
import { View, ScrollView, type ViewProps, useWindowDimensions } from 'react-native';
import { useSafeAreaInsets } from 'react-native-safe-area-context';
import { StatusBar } from 'expo-status-bar';
import { useThemeStore } from '@/store';
import { cn } from '@/lib';
import type { ReactNode } from 'react';

type SafeAreaEdge = 'top' | 'bottom' | 'left' | 'right';

interface ScreenProps extends ViewProps {
  scrollable?: boolean;
  edges?: SafeAreaEdge[];
  padded?: boolean;
  statusBarStyle?: 'light' | 'dark' | 'auto';
  className?: string;
  children: ReactNode;
}

export function Screen({
  scrollable = false,
  edges = ['top', 'bottom'],
  padded = true,
  statusBarStyle,
  className,
  children,
  ...rest
}: ScreenProps) {
  const insets = useSafeAreaInsets();
  const resolvedTheme = useThemeStore((state) => state.resolvedTheme);
  const { width } = useWindowDimensions();

  const isTablet = width >= 768;
  const horizontalPadding = isTablet ? 32 : 16;

  const innerPadding = {
    paddingTop:    edges.includes('top')    ? insets.top    : 0,
    paddingBottom: edges.includes('bottom') ? insets.bottom : 0,
    paddingLeft:   (edges.includes('left')  ? insets.left   : 0) + (padded ? horizontalPadding : 0),
    paddingRight:  (edges.includes('right') ? insets.right  : 0) + (padded ? horizontalPadding : 0),
  };

  const resolvedStatusBarStyle =
    statusBarStyle ?? (resolvedTheme === 'dark' ? 'light' : 'dark');

  return (
    <View className="flex-1 bg-background" {...rest}>
      <StatusBar style={resolvedStatusBarStyle} />
      {scrollable ? (
        <ScrollView
          className="flex-1"
          contentContainerStyle={[innerPadding, { flexGrow: 1 }]}
          keyboardShouldPersistTaps="handled"
          showsVerticalScrollIndicator={false}
        >
          <View className={cn('flex-1', className)}>{children}</View>
        </ScrollView>
      ) : (
        <View style={innerPadding} className={cn('flex-1', className)}>
          {children}
        </View>
      )}
    </View>
  );
}
```

**Critical bug to avoid — ScrollView padding:**
Padding applied to a `ScrollView`'s `style` affects the outer container box, not the scrollable content. This causes content to clip at edges. Always use `contentContainerStyle` for padding inside scroll views:
```tsx
// ❌ WRONG — padding on ScrollView style clips content at edges
<ScrollView className="flex-1 px-4">

// ✅ CORRECT — padding on contentContainerStyle
<ScrollView contentContainerStyle={{ paddingHorizontal: 16 }}>
```

**Usage:**
```tsx
<Screen>                          // basic screen
<Screen scrollable>               // scrollable content
<Screen padded={false}>           // full-bleed (maps, images)
<Screen edges={['bottom']}>       // modal — header handles top
<Screen statusBarStyle="light">   // force light icons (dark hero images)
```

---

## 30. Safe Areas & Status Bar

Modern phones have physical features that overlap the screen:
- **Notch** (iPhone X through 13) — camera cutout at top
- **Dynamic Island** (iPhone 14 Pro+) — interactive pill cutout at top
- **Home indicator** (iPhone X+) — swipe bar at bottom
- **Status bar** (all phones) — time, battery, signal icons at top
- **Navigation bar** (Android) — back/home/recents at bottom

**Content rendered under these areas is clipped or inaccessible.**

`react-native-safe-area-context` measures these areas:

```typescript
const insets = useSafeAreaInsets();
// On iPhone 14 Pro Max:
// insets.top = 59 (Dynamic Island + status bar)
// insets.bottom = 34 (home indicator)
// insets.left = 0
// insets.right = 0

// On iPhone SE (no notch):
// insets.top = 20 (status bar only)
// insets.bottom = 0 (physical home button, no indicator)
```

**All safe area handling in TaskFlow is centralized in `Screen`.** Individual screens never call `useSafeAreaInsets()` directly. They just use `<Screen>` and trust it to add correct padding.

**Status bar:** `expo-status-bar` controls the icon colors:
- `style="dark"` → dark/black icons (use when background is light)
- `style="light"` → white/light icons (use when background is dark)

`Screen` automatically picks the correct style based on `resolvedTheme`.

---

## 31. useResponsive Hook

`src/hooks/useResponsive.ts`:
```typescript
import { useWindowDimensions } from 'react-native';

export type DeviceSize = 'sm' | 'md' | 'lg' | 'xl';

interface ResponsiveInfo {
  width: number;
  height: number;
  size: DeviceSize;
  isPhone: boolean;
  isTablet: boolean;
  isLandscape: boolean;
}

const BREAKPOINTS = {
  sm: 0,    // < 640px  — small phones (iPhone SE)
  md: 640,  // 640–767  — large phones
  lg: 768,  // 768–1023 — small tablets (iPad Mini)
  xl: 1024, // 1024+    — large tablets, foldables
} as const;

function getDeviceSize(width: number): DeviceSize {
  if (width >= BREAKPOINTS.xl) return 'xl';
  if (width >= BREAKPOINTS.lg) return 'lg';
  if (width >= BREAKPOINTS.md) return 'md';
  return 'sm';
}

export function useResponsive(): ResponsiveInfo {
  const { width, height } = useWindowDimensions();
  const size = getDeviceSize(width);
  return {
    width, height, size,
    isPhone: size === 'sm' || size === 'md',
    isTablet: size === 'lg' || size === 'xl',
    isLandscape: width > height,
  };
}
```

**Always `useWindowDimensions()` not `Dimensions.get()`:**
- `Dimensions.get()` is a one-time snapshot. It doesn't update when the user rotates the phone, opens a foldable, or enters iPad split-screen.
- `useWindowDimensions()` is a hook. It triggers re-renders when dimensions change, keeping your layout in sync.

**Usage:**
```tsx
const { isTablet, isLandscape } = useResponsive();

<View className={cn('gap-4', isTablet ? 'flex-row' : 'flex-col')}>
  {/* Horizontal on tablet, vertical on phone */}
</View>
```

---

## 32. Tab Bar with Icons

`app/(app)/(tabs)/_layout.tsx`:
```tsx
import { Tabs } from 'expo-router';
import { CheckSquare, FolderKanban, Bell, User } from 'lucide-react-native';
import { useThemeStore } from '@/store';
import { colors } from '@/constants';

export default function TabsLayout() {
  const resolvedTheme = useThemeStore((s) => s.resolvedTheme);
  const themeColors = colors[resolvedTheme];

  return (
    <Tabs
      screenOptions={{
        headerStyle: { backgroundColor: themeColors.background },
        headerTitleStyle: { color: themeColors.foreground, fontWeight: '600' },
        headerShadowVisible: false,
        tabBarStyle: {
          backgroundColor: themeColors.background,
          borderTopColor: themeColors.border,
        },
        tabBarActiveTintColor: themeColors.primary,
        tabBarInactiveTintColor: themeColors.mutedForeground,
        tabBarLabelStyle: { fontSize: 11, fontWeight: '500' },
      }}
    >
      <Tabs.Screen name="index" options={{
        title: 'Tasks',
        tabBarIcon: ({ color, size }) => <CheckSquare color={color} size={size} />,
      }} />
      <Tabs.Screen name="projects" options={{
        title: 'Projects',
        tabBarIcon: ({ color, size }) => <FolderKanban color={color} size={size} />,
      }} />
      <Tabs.Screen name="notifications" options={{
        title: 'Notifications',
        tabBarIcon: ({ color, size }) => <Bell color={color} size={size} />,
      }} />
      <Tabs.Screen name="profile" options={{
        title: 'Profile',
        tabBarIcon: ({ color, size }) => <User color={color} size={size} />,
      }} />
    </Tabs>
  );
}
```

`tabBarActiveTintColor` and `tabBarInactiveTintColor` are automatically passed as `color` to the `tabBarIcon` function. Icons change color based on active state without any conditional logic — the navigator handles it.

`headerShadowVisible: false` removes the default 1px separator line under the header on iOS. Produces a cleaner look that matches the rest of the design.

---

## 33. Theme-Aware Headers

`app/(app)/_layout.tsx`:
```tsx
import { Stack } from 'expo-router';
import { useThemeStore } from '@/store';
import { colors } from '@/constants';

export default function AppLayout() {
  const resolvedTheme = useThemeStore((s) => s.resolvedTheme);
  const themeColors = colors[resolvedTheme];

  return (
    <Stack
      screenOptions={{
        headerStyle: { backgroundColor: themeColors.background },
        headerTitleStyle: { color: themeColors.foreground, fontWeight: '600' },
        headerTintColor: themeColors.primary,
        headerShadowVisible: false,
        contentStyle: { backgroundColor: themeColors.background },
      }}
    >
      <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
      <Stack.Screen name="task/[id]" options={{ title: 'Task Detail', headerBackTitle: 'Back' }} />
      <Stack.Screen name="task/new" options={{ presentation: 'modal', headerShown: false }} />
    </Stack>
  );
}
```

**`headerTintColor`** controls the back arrow and any `headerRight`/`headerLeft` buttons. Setting to primary gives the header a branded feel.

**`contentStyle`** sets the background color of the area between the header and the screen content. Critical for dark mode — without it, the area flashes white during transitions.

**`headerBackTitle: 'Back'`** — iOS shows the previous screen's title in the back button by default. This is often too long. Overriding with `'Back'` is cleaner.

---

## 34. Auth Stack Layout

`app/(auth)/_layout.tsx`:
```tsx
import { Stack } from 'expo-router';
import { useThemeStore } from '@/store';
import { colors } from '@/constants';

export default function AuthLayout() {
  const resolvedTheme = useThemeStore((s) => s.resolvedTheme);
  const themeColors = colors[resolvedTheme];

  return (
    <Stack
      screenOptions={{
        headerShown: false,
        contentStyle: { backgroundColor: themeColors.background },
      }}
    />
  );
}
```

Auth screens have no header. They have their own custom design (logo, tagline, illustration). The `contentStyle` still needs to match the theme to prevent background color flashes during transitions.

---

## 35. Modal Routes

Modals slide up from the bottom on iOS. On Android they fade in. The `presentation: 'modal'` option enables this.

```tsx
// In parent _layout.tsx
<Stack.Screen
  name="task/new"
  options={{
    presentation: 'modal',
    headerShown: false,
  }}
/>
```

**Why `headerShown: false` on modals:** The native modal header on Android can overlap with safe area insets, causing status bar icon overlap. Instead, we build a custom header inside the screen:

```tsx
// app/(app)/task/new.tsx
export default function NewTaskScreen() {
  const router = useRouter();
  const resolvedTheme = useThemeStore((s) => s.resolvedTheme);
  const iconColor = colors[resolvedTheme].foreground;

  return (
    <Screen>
      {/* Custom header */}
      <View className="mb-4 flex-row items-center justify-between">
        <Text variant="h2">New Task</Text>
        <Pressable
          onPress={() => router.back()}
          accessibilityRole="button"
          accessibilityLabel="Close"
          hitSlop={12}
          className="rounded-full p-2 active:bg-muted"
        >
          <X size={24} color={iconColor} />
        </Pressable>
      </View>
      {/* Content */}
    </Screen>
  );
}
```

**`hitSlop={12}`** — expands the tappable area 12px beyond the visual boundary. Critical for small touch targets like the close button (24px icon). Prevents "I tried to close but couldn't tap the X" frustration.

---

## 36. Dynamic Routes

Files with `[param]` match any URL segment at that position.

**File:** `app/(app)/task/[id].tsx`
**Matches:** `/task/abc123`, `/task/xyz`, `/task/any-value`

```tsx
import { useLocalSearchParams, useRouter } from 'expo-router';

export default function TaskDetailScreen() {
  // Typed generic — TypeScript knows id is a string
  const { id } = useLocalSearchParams<{ id: string }>();
  const router = useRouter();

  return (
    <Screen scrollable>
      <Text variant="overline" color="muted">Task ID</Text>
      <Text variant="h3">{id}</Text>
      <Button label="Go Back" onPress={() => router.back()} />
    </Screen>
  );
}
```

**`useLocalSearchParams<{ id: string }>()`** — the generic type tells TypeScript what params to expect. Without it, params are typed as `string | string[]` and you'd need to handle the array case.

**Multiple params** — a file named `[id]/edit.tsx` would match `/task/abc123/edit` and `useLocalSearchParams` would give you `{ id: 'abc123' }`.

---

## 37. ROUTES Constants

Never write raw route strings in components. A centralized file means:
- Find all navigation points with one grep: `grep -r "ROUTES.LOGIN" src/ app/`
- Rename a route in one place, not scattered across 30 files
- TypeScript autocomplete on all routes

`src/constants/routes.ts`:
```typescript
export const ROUTES = {
  // Auth
  LOGIN:           '/(auth)/login',
  SIGNUP:          '/(auth)/signup',
  FORGOT_PASSWORD: '/(auth)/forgot-password',

  // App tabs
  TASKS:         '/(app)/(tabs)',
  PROJECTS:      '/(app)/(tabs)/projects',
  NOTIFICATIONS: '/(app)/(tabs)/notifications',
  PROFILE:       '/(app)/(tabs)/profile',

  // Task routes
  TASK_NEW:    '/(app)/task/new',
  TASK_DETAIL: (id: string) => `/(app)/task/${id}` as const,

  // Dev
  COMPONENT_SHOWCASE: '/(dev)/components',
} as const;
```

Update `src/constants/index.ts`:
```typescript
export * from './theme';
export * from './routes';
```

---

## 38. Programmatic Navigation

```tsx
import { useRouter } from 'expo-router';
import { ROUTES } from '@/constants';

function MyComponent() {
  const router = useRouter();

  return (
    <>
      {/* push — adds to history, back button returns here */}
      <Button label="Open Task" onPress={() => router.push(ROUTES.TASK_DETAIL('abc123'))} />

      {/* replace — removes current from history */}
      {/* Use for logout — don't want back button to return to authenticated screen */}
      <Button label="Sign Out" onPress={() => router.replace(ROUTES.LOGIN)} />

      {/* back — go to previous screen */}
      <Button label="Go Back" onPress={() => router.back()} />

      {/* Declarative navigation — used in entry redirect */}
      {/* <Redirect href={ROUTES.LOGIN} /> */}
    </>
  );
}
```

**`push` vs `replace`:**
- `push` → stacks on history. User can go back. For normal forward navigation.
- `replace` → removes current screen from history. Used for logout, auth redirects. Prevents users going "back" to a screen they shouldn't access.

**`<Redirect>`** — renders nothing, immediately navigates. Used in `app/index.tsx` to redirect to login or tabs based on auth state.

---

## 39. Deep Linking

Deep links open your app at a specific route from an external source (push notification tap, web URL, share link, email link).

**Format:** `taskflow://task/abc123` → opens `/task/abc123`

**Required config in `app.json`:**
```json
{ "expo": { "scheme": "taskflow" } }
```

Expo Router handles routing automatically — your file structure maps directly to deep link paths. No additional routing config.

**Test on Android emulator:**
```bash
adb shell am start -W -a android.intent.action.VIEW \
  -d "taskflow://task/abc123" com.taskflow.app
```

**Test on iOS Simulator (Mac only):**
```bash
xcrun simctl openurl booted "taskflow://task/abc123"
```

**Where deep links are used in TaskFlow:**
- Push notification taps (Task 103) → open specific task
- Share feature → share link opens the task in the app
- Email links → password reset opens the forgot password screen

---

## 40. Zustand Store Pattern

Zustand is a minimal state manager. No Provider, no boilerplate, no reducers.

```typescript
import { create } from 'zustand';

interface CounterState {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
}

const useCounterStore = create<CounterState>((set, get) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}));
```

**`set`** — merges partial state (like `setState` in React). Only provided keys are updated.

**`get`** — reads current state inside actions. Use when an action needs current state that isn't in the update:
```typescript
addToCount: (n: number) => set({ count: get().count + n })
```

**No Provider needed** — Zustand stores are module-level singletons. Any component anywhere imports and uses them directly without wrapping the tree.

**In components — always use selectors:**
```tsx
// ❌ Subscribes to entire store — re-renders on any change
const { count, increment } = useCounterStore();

// ✅ Subscribes only to count — re-renders only when count changes
const count = useCounterStore((s) => s.count);
const increment = useCounterStore((s) => s.increment);
```

---

## 41. Persist Middleware & Storage Adapter

Makes store state survive app restarts by reading from and writing to AsyncStorage (later MMKV).

`src/services/storage/persist-storage.ts`:
```typescript
import AsyncStorage from '@react-native-async-storage/async-storage';
import { createJSONStorage } from 'zustand/middleware';

/**
 * Persistence adapter for Zustand stores.
 * Swap: change this file in Task 91 to use MMKV (~10x faster reads).
 * All stores using this adapter automatically use MMKV after the swap.
 */
export const persistStorage = createJSONStorage(() => AsyncStorage);
```

```bash
npx expo install @react-native-async-storage/async-storage
```

**`createJSONStorage`** — wraps the storage adapter with JSON serialization/deserialization. Stores complex objects (not just strings).

**Why a separate adapter file:** All stores reference `persistStorage`. Swapping AsyncStorage → MMKV is a one-file change. No store files need to be touched.

**How persist middleware works:**
```typescript
const useStore = create(
  persist(
    (set) => ({
      user: null,
      setUser: (user) => set({ user }),
    }),
    {
      name: 'taskflow-auth',        // key in AsyncStorage
      storage: persistStorage,
      partialize: (state) => ({     // only persist these fields
        user: state.user,
      }),
      onRehydrateStorage: () => (state) => {
        // Runs once after app restarts and storage loads
        // Set derived state that you didn't persist
        if (state) {
          state.status = state.user ? 'authenticated' : 'unauthenticated';
        }
      },
    },
  ),
);
```

**`partialize`** — only save what needs to survive a restart. Never persist loading booleans, error messages, or computed/derived state.

---

## 42. Auth Types

`src/types/auth.types.ts`:
```typescript
export interface User {
  id: string;
  email: string;
  displayName: string | null;
  photoURL: string | null;
  emailVerified: boolean;
  createdAt: string; // ISO 8601
}

export type AuthStatus =
  | 'idle'            // initial — before auth check completes
  | 'loading'         // auth operation in progress
  | 'authenticated'   // signed in with valid user
  | 'unauthenticated' // explicitly signed out
```

**Why 4 states instead of a boolean:**
A boolean `isAuthenticated` doesn't capture the intermediate state. On app start:
1. App opens — state is `idle` (haven't checked storage yet)
2. Storage loads — state becomes `authenticated` or `unauthenticated`

Without `idle`, screens would flash the wrong content during the storage load. With `idle`, you show a loading spinner until the state is known.

---

## 43. Auth Store

`src/store/auth.store.ts`:
```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import { persistStorage } from '@/services/storage';
import type { User, AuthStatus } from '@/types';

interface AuthState {
  user: User | null;
  status: AuthStatus;
  isAuthenticated: () => boolean;
  setUser: (user: User | null) => void;
  setStatus: (status: AuthStatus) => void;
  signIn: (user: User) => void;
  signOut: () => void;
  reset: () => void;
}

const initialState = {
  user: null,
  status: 'idle' as AuthStatus,
};

export const useAuthStore = create<AuthState>()(
  persist(
    (set, get) => ({
      ...initialState,

      isAuthenticated: () => {
        const { user, status } = get();
        return status === 'authenticated' && user !== null;
      },

      setUser: (user) => set({ user }),
      setStatus: (status) => set({ status }),

      signIn: (user) => set({ user, status: 'authenticated' }),
      signOut: () => set({ user: null, status: 'unauthenticated' }),
      reset: () => set(initialState),
    }),
    {
      name: 'taskflow-auth',
      storage: persistStorage,
      partialize: (state) => ({ user: state.user }),
      onRehydrateStorage: () => (state) => {
        if (state) {
          state.status = state.user ? 'authenticated' : 'unauthenticated';
        }
      },
    },
  ),
);
```

**`isAuthenticated()` is a function, not a computed property** — Zustand v5 doesn't have built-in computed selectors. A getter function called at render time is the idiomatic pattern.

---

## 44. UI Store

For ephemeral UI state that doesn't survive restarts.

`src/store/ui.store.ts`:
```typescript
import { create } from 'zustand';

interface SnackbarConfig {
  message: string;
  variant: 'default' | 'success' | 'warning' | 'danger';
  duration?: number;
}

interface UIState {
  isOffline: boolean;
  setOffline: (offline: boolean) => void;

  snackbar: SnackbarConfig | null;
  showSnackbar: (config: SnackbarConfig) => void;
  hideSnackbar: () => void;

  isAppLoading: boolean;
  setAppLoading: (loading: boolean) => void;
}

export const useUIStore = create<UIState>((set) => ({
  isOffline: false,
  setOffline: (isOffline) => set({ isOffline }),

  snackbar: null,
  showSnackbar: (snackbar) => set({ snackbar }),
  hideSnackbar: () => set({ snackbar: null }),

  isAppLoading: false,
  setAppLoading: (isAppLoading) => set({ isAppLoading }),
}));
```

**No `persist`** — snackbars, offline banners, and loading overlays should reset on app restart. They're responses to current session events, not user preferences.

---

## 45. Selector Pattern

The single most important performance pattern in Zustand usage.

```tsx
// ❌ BAD — subscribes to entire store
// Re-renders every time ANY field in the store changes
const { user, status, signIn, signOut } = useAuthStore();

// ✅ GOOD — subscribes to specific field
// Re-renders only when that specific field changes
const user = useAuthStore((s) => s.user);
const status = useAuthStore((s) => s.status);
```

**Why this matters at scale:** Imagine a list of 100 task cards. Each imports a store value. If they import the whole store object, all 100 re-render whenever *anything* in the store changes — even unrelated fields. With selectors, they only re-render when their specific slice changes.

**Selecting objects — the shallow comparison gotcha:**
```tsx
// ❌ This creates a new object every render — defeats the purpose
const { user, status } = useAuthStore((s) => ({ user: s.user, status: s.status }));

// ✅ Option 1: Two separate selectors
const user = useAuthStore((s) => s.user);
const status = useAuthStore((s) => s.status);

// ✅ Option 2: Use shallow comparison
import { useShallow } from 'zustand/react/shallow';
const { user, status } = useAuthStore(
  useShallow((s) => ({ user: s.user, status: s.status }))
);
```

---

## 46. Auth Selector Hooks

Convenience hooks that wrap selectors — cleaner imports in components.

`src/store/auth.selectors.ts`:
```typescript
import { useAuthStore } from './auth.store';

// State selectors — each re-renders only when its value changes
export const useUser = () => useAuthStore((s) => s.user);
export const useAuthStatus = () => useAuthStore((s) => s.status);
export const useIsAuthenticated = () => useAuthStore((s) => s.isAuthenticated());

// Action selector — actions are stable references, no perf concern
export const useAuthActions = () =>
  useAuthStore((s) => ({
    setUser: s.setUser,
    setStatus: s.setStatus,
    signIn: s.signIn,
    signOut: s.signOut,
    reset: s.reset,
  }));
```

**Usage in components:**
```tsx
// Clean, readable, correctly scoped
const user = useUser();
const isAuthenticated = useIsAuthenticated();
const { signIn, signOut } = useAuthActions();
```

Update `src/store/index.ts` to export everything:
```typescript
export { useAuthStore } from './auth.store';
export { useUser, useAuthStatus, useIsAuthenticated, useAuthActions } from './auth.selectors';
export { useThemeStore } from './theme.store';
export type { ThemePreference, ResolvedTheme } from './theme.store';
export { useUIStore } from './ui.store';
```

---

## 47. TanStack Query Setup

TanStack Query manages all server state: fetching, caching, background refetching, and synchronization.

```bash
npm install @tanstack/react-query
```

`src/lib/query-client.ts`:
```typescript
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,    // 5 min: treat cached data as fresh
      gcTime: 1000 * 60 * 10,      // 10 min: keep in cache after unmount
      retry: 2,                     // retry failed requests twice
      refetchOnWindowFocus: false,  // don't refetch when app foregrounds
    },
    mutations: {
      retry: 0,                     // never auto-retry mutations
    },
  },
});
```

Wrap the app in `app/_layout.tsx`:
```tsx
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from '@/lib/query-client';

// Inside RootLayout return:
<QueryClientProvider client={queryClient}>
  <SafeAreaProvider>
    {/* ... */}
  </SafeAreaProvider>
</QueryClientProvider>
```

**`staleTime`** — how long before data is considered stale and needs refetching. Set to 5 minutes for tasks (changes aren't that time-sensitive). Set to 0 for real-time data.

**`gcTime`** (formerly `cacheTime`) — how long to keep data in memory after the component unmounts. User can navigate away and come back within 10 minutes and see instant cached results.

---

## 48. Query Key Factory

Query keys identify specific pieces of cached data. A factory ensures consistent, hierarchical, typed keys.

`src/services/api/query-keys.ts`:
```typescript
import type { TaskFilters } from '@/features/tasks/types';

export const queryKeys = {
  tasks: {
    all:     ['tasks'] as const,
    lists:   () => [...queryKeys.tasks.all, 'list'] as const,
    list:    (filters: TaskFilters) => [...queryKeys.tasks.lists(), filters] as const,
    details: () => [...queryKeys.tasks.all, 'detail'] as const,
    detail:  (id: string) => [...queryKeys.tasks.details(), id] as const,
  },
  auth: {
    user: ['auth', 'user'] as const,
  },
} as const;
```

**Hierarchical keys enable targeted invalidation:**
```typescript
// After creating a task, invalidate all list queries
// (but NOT detail queries for tasks the user didn't modify)
queryClient.invalidateQueries({ queryKey: queryKeys.tasks.lists() });

// After updating a specific task, only invalidate that detail
queryClient.invalidateQueries({ queryKey: queryKeys.tasks.detail('abc123') });

// After a force-sync, invalidate everything task-related
queryClient.invalidateQueries({ queryKey: queryKeys.tasks.all });
```

**Why arrays, not strings:** TanStack Query does deep array comparison. `['tasks', 'list', { status: 'pending' }]` and `['tasks', 'list', { status: 'completed' }]` are different keys and have separate caches. String-based keys can't represent this structure.

---

## 49. Axios Client

`src/services/api/client.ts`:
```typescript
import axios from 'axios';
import { env } from '@/lib/env';

export const apiClient = axios.create({
  baseURL: env.apiUrl,
  timeout: 15000,
  headers: {
    'Content-Type': 'application/json',
    Accept: 'application/json',
  },
});
```

```bash
npm install axios
```

**`timeout: 15000`** — if a request takes more than 15 seconds, it's automatically cancelled and throws an error. Without a timeout, requests can hang indefinitely on slow connections.

---

## 50. Request Interceptor — Auth Token

Automatically attaches the Firebase ID token to every request. No component needs to manually add auth headers.

```typescript
import * as SecureStore from 'expo-secure-store';

apiClient.interceptors.request.use(
  async (config) => {
    const token = await SecureStore.getItemAsync('taskflow-auth-token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error),
);
```

**Why interceptors, not manual header passing:** With interceptors, you define auth logic once. Every `apiClient.get()`, `apiClient.post()`, etc. throughout the codebase automatically includes the token. Without interceptors, every API call would need to manually read the token and set the header — leading to bugs when you forget.

---

## 51. Response Interceptor — Error Handling

Handles common error cases globally:

```typescript
import { useAuthStore } from '@/store';

apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const status = error.response?.status;

    if (status === 401) {
      // Token expired or invalid — sign out user
      useAuthStore.getState().signOut();
      throw new AuthError('Session expired. Please sign in again.');
    }

    if (status === 403) {
      throw new AuthError('You do not have permission to do this.');
    }

    if (status && status >= 500) {
      throw new NetworkError('Server error. Please try again later.');
    }

    if (!error.response) {
      // No response at all — likely network error
      throw new NetworkError('No internet connection.');
    }

    return Promise.reject(error);
  },
);
```

**`useAuthStore.getState().signOut()`** — reads current state outside of a React component. Zustand stores expose `.getState()` for exactly this use case (calling store actions from non-React contexts like interceptors, background tasks, etc.).

---

## 52. Zod API Response Validation

Validates API responses at runtime. Catches backend schema changes before they cause cryptic UI crashes.

```typescript
import { z } from 'zod';

const TaskSchema = z.object({
  id: z.string(),
  title: z.string(),
  description: z.string().nullable(),
  status: z.enum(['pending', 'in_progress', 'completed']),
  priority: z.enum(['low', 'medium', 'high']),
  due_date: z.string().datetime().nullable(),
  created_at: z.string().datetime(),
  user_id: z.string(),
});

const TasksResponseSchema = z.object({
  data: z.array(TaskSchema),
  count: z.number(),
});

// TypeScript type derived from schema — single source of truth
export type Task = z.infer<typeof TaskSchema>;
```

**Usage with API calls:**
```typescript
const response = await apiClient.get('/tasks');
const parsed = TasksResponseSchema.parse(response.data);
// If backend returns unexpected shape, parse() throws ZodError
// You get a clear error: "Expected string, received undefined at path data[0].title"
// instead of a cryptic "Cannot read property 'slice' of undefined"
```

---

## 53. Custom Error Classes

Typed errors let different handlers react differently to different failure types.

`src/services/api/errors.ts`:
```typescript
export class NetworkError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'NetworkError';
  }
}

export class AuthError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'AuthError';
  }
}

export class ValidationError extends Error {
  constructor(message: string, public field?: string) {
    super(message);
    this.name = 'ValidationError';
  }
}

export class NotFoundError extends Error {
  constructor(message = 'Resource not found') {
    super(message);
    this.name = 'NotFoundError';
  }
}
```

**Usage in error boundaries and catch blocks:**
```typescript
try {
  await createTask(data);
} catch (error) {
  if (error instanceof NetworkError) {
    showOfflineBanner();
  } else if (error instanceof AuthError) {
    signOut();
  } else if (error instanceof ValidationError) {
    setFieldError(error.field, error.message);
  }
}
```

---

## 54. safeRequest Wrapper

A utility that combines Axios + Zod validation into a single typed, validated call.

`src/services/api/safe-request.ts`:
```typescript
import { z, type ZodSchema } from 'zod';
import { apiClient } from './client';
import type { AxiosRequestConfig } from 'axios';

export async function safeRequest<T>(
  schema: ZodSchema<T>,
  config: AxiosRequestConfig,
): Promise<T> {
  const response = await apiClient(config);
  return schema.parse(response.data);
}
```

**Usage:**
```typescript
const tasks = await safeRequest(TasksResponseSchema, {
  method: 'GET',
  url: '/tasks',
  params: { status: 'pending' },
});
// tasks is fully typed as TasksResponse — TypeScript knows the shape
```

**One call, three guarantees:**
1. Auth token attached (from request interceptor)
2. Error handled globally (from response interceptor)
3. Response shape validated (from Zod schema)

---

## 55. Firebase Project Setup

1. Go to [console.firebase.google.com](https://console.firebase.google.com) → New project → name it "taskflow-dev"

2. Add iOS app:
   - Bundle ID: `com.taskflow.app.dev`
   - Download `GoogleService-Info.plist`

3. Add Android app:
   - Package: `com.taskflow.app.dev`
   - Download `google-services.json`

4. Place both files in the project root (Expo config plugins will pick them up)

5. Enable Email/Password auth in Firebase Console → Authentication → Sign-in method

6. Install native Firebase SDK:
```bash
npx expo install @react-native-firebase/app @react-native-firebase/auth
```

7. Add config plugins to `app.json`:
```json
{
  "plugins": [
    "expo-router",
    "@react-native-firebase/app",
    "@react-native-firebase/auth"
  ]
}
```

8. Build a dev client (Expo Go won't work — Firebase needs native code):
```bash
eas build --profile development --platform all
```

**Why native Firebase over Firebase JS SDK:** The native SDK (`@react-native-firebase`) uses platform-specific Firebase SDKs under the hood. This gives you proper background token refresh, better APNs/FCM integration, and reliable offline persistence. The JS SDK works but is a compatibility layer — not the recommended path for production.

---

## 56. Auth Form Zod Schemas

`src/features/auth/schemas.ts`:
```typescript
import { z } from 'zod';

export const loginSchema = z.object({
  email: z.string()
    .min(1, 'Email is required')
    .email('Please enter a valid email'),
  password: z.string()
    .min(1, 'Password is required')
    .min(8, 'Password must be at least 8 characters'),
});

export const signupSchema = z.object({
  email: z.string()
    .min(1, 'Email is required')
    .email('Please enter a valid email'),
  password: z.string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Must contain at least one uppercase letter')
    .regex(/[0-9]/, 'Must contain at least one number'),
  confirmPassword: z.string().min(1, 'Please confirm your password'),
}).refine(
  (data) => data.password === data.confirmPassword,
  { message: 'Passwords do not match', path: ['confirmPassword'] },
);

export const forgotPasswordSchema = z.object({
  email: z.string()
    .min(1, 'Email is required')
    .email('Please enter a valid email'),
});

export type LoginFormData = z.infer<typeof loginSchema>;
export type SignupFormData = z.infer<typeof signupSchema>;
export type ForgotPasswordFormData = z.infer<typeof forgotPasswordSchema>;
```

**`z.infer<typeof schema>`** — derives TypeScript types from the schema. One source of truth for both runtime validation and compile-time types. When the schema changes, types update automatically.

**`.refine()`** — cross-field validation. Runs after all individual field validations pass. The `path: ['confirmPassword']` means the error appears on the `confirmPassword` field, not as a form-level error.

---

## 57. Login Screen — React Hook Form

`src/features/auth/screens/LoginScreen.tsx`:
```tsx
import { View } from 'react-native';
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { Link } from 'expo-router';
import { Screen, Text, Input, Button } from '@/components/ui';
import { loginSchema, type LoginFormData } from '../schemas';

interface LoginScreenProps {
  onSubmit: (data: LoginFormData) => Promise<void>;
  error?: string | null;
}

export function LoginScreen({ onSubmit, error }: LoginScreenProps) {
  const {
    control,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<LoginFormData>({
    resolver: zodResolver(loginSchema),
    defaultValues: { email: '', password: '' },
  });

  return (
    <Screen scrollable>
      <View className="flex-1 justify-center gap-6 py-8">
        <View>
          <Text variant="h1">Welcome back</Text>
          <Text color="muted" className="mt-1">Sign in to your TaskFlow account</Text>
        </View>

        {error ? (
          <View className="rounded-md bg-danger/10 p-3">
            <Text color="danger" variant="body-sm">{error}</Text>
          </View>
        ) : null}

        <View className="gap-4">
          <Controller
            control={control}
            name="email"
            render={({ field: { onChange, value, onBlur } }) => (
              <Input
                label="Email"
                value={value}
                onChangeText={onChange}
                onBlur={onBlur}
                error={errors.email?.message}
                keyboardType="email-address"
                autoCapitalize="none"
                autoCorrect={false}
                returnKeyType="next"
              />
            )}
          />
          <Controller
            control={control}
            name="password"
            render={({ field: { onChange, value, onBlur } }) => (
              <Input
                label="Password"
                value={value}
                onChangeText={onChange}
                onBlur={onBlur}
                error={errors.password?.message}
                secureTextEntry
                returnKeyType="done"
                onSubmitEditing={handleSubmit(onSubmit)}
              />
            )}
          />
        </View>

        <Button
          label="Sign In"
          loading={isSubmitting}
          onPress={handleSubmit(onSubmit)}
          fullWidth
        />

        <Link href="/forgot-password">
          <Text color="primary" className="text-center">Forgot password?</Text>
        </Link>

        <View className="flex-row justify-center gap-1">
          <Text color="muted">Don't have an account?</Text>
          <Link href="/signup">
            <Text color="primary">Sign up</Text>
          </Link>
        </View>
      </View>
    </Screen>
  );
}
```

**Why `Controller` instead of `register`:** React Native inputs are controlled components — they don't work with `register` (which relies on DOM refs). `Controller` wraps any controlled input (including custom ones like our `Input` component) with React Hook Form's state management.

**`handleSubmit(onSubmit)`** — wraps your submit function. It runs validation first. Only calls `onSubmit` if all fields pass validation. If any field fails, it updates `errors` and the relevant fields show error messages.

**`onSubmitEditing={handleSubmit(onSubmit)}`** on the last input — pressing the keyboard's "Done" button submits the form. Combined with `returnKeyType="done"`, this provides a native-feeling form submission flow.

---

## 58. Signup Screen — Cross-field Validation

```tsx
export function SignupScreen({ onSubmit, error }: SignupScreenProps) {
  const passwordRef = useRef<TextInput>(null);
  const confirmRef = useRef<TextInput>(null);

  const { control, handleSubmit, formState: { errors, isSubmitting } } =
    useForm<SignupFormData>({
      resolver: zodResolver(signupSchema),
      defaultValues: { email: '', password: '', confirmPassword: '' },
    });

  return (
    <Screen scrollable>
      <View className="flex-1 justify-center gap-6 py-8">
        <Text variant="h1">Create account</Text>

        <View className="gap-4">
          <Controller
            control={control}
            name="email"
            render={({ field: { onChange, value, onBlur } }) => (
              <Input
                label="Email"
                value={value}
                onChangeText={onChange}
                onBlur={onBlur}
                error={errors.email?.message}
                keyboardType="email-address"
                autoCapitalize="none"
                returnKeyType="next"
                onSubmitEditing={() => passwordRef.current?.focus()}
              />
            )}
          />
          <Controller
            control={control}
            name="password"
            render={({ field: { onChange, value, onBlur } }) => (
              <Input
                ref={passwordRef}
                label="Password"
                value={value}
                onChangeText={onChange}
                onBlur={onBlur}
                error={errors.password?.message}
                secureTextEntry
                returnKeyType="next"
                onSubmitEditing={() => confirmRef.current?.focus()}
              />
            )}
          />
          <Controller
            control={control}
            name="confirmPassword"
            render={({ field: { onChange, value, onBlur } }) => (
              <Input
                ref={confirmRef}
                label="Confirm Password"
                value={value}
                onChangeText={onChange}
                onBlur={onBlur}
                error={errors.confirmPassword?.message}
                secureTextEntry
                returnKeyType="done"
                onSubmitEditing={handleSubmit(onSubmit)}
              />
            )}
          />
        </View>

        <Button label="Create Account" loading={isSubmitting} onPress={handleSubmit(onSubmit)} fullWidth />
      </View>
    </Screen>
  );
}
```

**Focus management with refs:** `onSubmitEditing={() => passwordRef.current?.focus()}` moves focus to the next input when the user taps the keyboard's "Next" button. This creates a smooth tabbing flow without the user needing to tap each field.

---

## 59. Forgot Password Screen

```tsx
export function ForgotPasswordScreen({ onSubmit, error, success }: ForgotPasswordScreenProps) {
  const { control, handleSubmit, formState: { errors, isSubmitting } } =
    useForm<ForgotPasswordFormData>({
      resolver: zodResolver(forgotPasswordSchema),
      defaultValues: { email: '' },
    });

  if (success) {
    return (
      <Screen>
        <View className="flex-1 items-center justify-center gap-4">
          <Text variant="h2">Check your email</Text>
          <Text color="muted" className="text-center">
            We've sent a password reset link to your email address.
          </Text>
          <Link href="/login">
            <Button label="Back to Login" variant="secondary" />
          </Link>
        </View>
      </Screen>
    );
  }

  return (
    <Screen scrollable>
      <View className="flex-1 justify-center gap-6">
        <View>
          <Text variant="h1">Forgot password?</Text>
          <Text color="muted" className="mt-1">
            Enter your email and we'll send you a reset link.
          </Text>
        </View>
        <Controller
          control={control}
          name="email"
          render={({ field: { onChange, value, onBlur } }) => (
            <Input
              label="Email"
              value={value}
              onChangeText={onChange}
              onBlur={onBlur}
              error={errors.email?.message}
              keyboardType="email-address"
              autoCapitalize="none"
            />
          )}
        />
        <Button label="Send Reset Link" loading={isSubmitting} onPress={handleSubmit(onSubmit)} fullWidth />
      </View>
    </Screen>
  );
}
```

---

## 60. Firebase Auth Service

`src/services/auth/firebase-auth.ts`:
```typescript
import auth from '@react-native-firebase/auth';
import type { User } from '@/types';

function mapFirebaseUser(firebaseUser: FirebaseUser): User {
  return {
    id: firebaseUser.uid,
    email: firebaseUser.email ?? '',
    displayName: firebaseUser.displayName,
    photoURL: firebaseUser.photoURL,
    emailVerified: firebaseUser.emailVerified,
    createdAt: firebaseUser.metadata.creationTime ?? new Date().toISOString(),
  };
}

export const firebaseAuth = {
  signUp: async (email: string, password: string) => {
    const { user } = await auth().createUserWithEmailAndPassword(email, password);
    return mapFirebaseUser(user);
  },

  signIn: async (email: string, password: string) => {
    const { user } = await auth().signInWithEmailAndPassword(email, password);
    return mapFirebaseUser(user);
  },

  signOut: async () => {
    await auth().signOut();
  },

  sendPasswordReset: async (email: string) => {
    await auth().sendPasswordResetEmail(email);
  },

  getIdToken: async (): Promise<string | null> => {
    const user = auth().currentUser;
    if (!user) return null;
    return user.getIdToken(false); // false = don't force refresh
  },

  refreshIdToken: async (): Promise<string | null> => {
    const user = auth().currentUser;
    if (!user) return null;
    return user.getIdToken(true); // true = force refresh
  },

  onAuthStateChanged: (callback: (user: User | null) => void) => {
    return auth().onAuthStateChanged((firebaseUser) => {
      callback(firebaseUser ? mapFirebaseUser(firebaseUser) : null);
    });
  },
};
```

**`mapFirebaseUser`** — translates Firebase's `FirebaseAuthTypes.User` to our own `User` type. This decouples the rest of the app from Firebase's shape. If we ever switch auth providers, only this file changes.

**`onAuthStateChanged` returns unsubscribe function** — call the returned function to stop listening. Essential for preventing memory leaks:
```typescript
useEffect(() => {
  const unsubscribe = firebaseAuth.onAuthStateChanged((user) => {
    if (user) signIn(user);
    else signOut();
  });
  return unsubscribe; // cleanup on unmount
}, []);
```

---

## 61. Wiring Forms to Firebase

Route files remain thin. The screen component handles the form. The route file orchestrates auth and navigation.

`app/(auth)/login.tsx`:
```tsx
import { useState } from 'react';
import { useRouter } from 'expo-router';
import { LoginScreen } from '@/features/auth';
import { firebaseAuth } from '@/services/auth';
import { useAuthActions } from '@/store';
import { secureStorage } from '@/services/storage';
import { ROUTES } from '@/constants';
import { firebaseErrorToMessage } from '@/features/auth/utils';
import type { LoginFormData } from '@/features/auth/schemas';

export default function LoginRoute() {
  const [error, setError] = useState<string | null>(null);
  const { signIn } = useAuthActions();
  const router = useRouter();

  const handleLogin = async (data: LoginFormData) => {
    try {
      setError(null);
      const user = await firebaseAuth.signIn(data.email, data.password);
      const token = await firebaseAuth.getIdToken();
      if (token) await secureStorage.setToken(token);
      signIn(user);
      router.replace(ROUTES.TASKS);
    } catch (err) {
      setError(firebaseErrorToMessage(err));
    }
  };

  return <LoginScreen onSubmit={handleLogin} error={error} />;
}
```

**This separation of concerns:**
- Screen component owns: form state, validation UI, visual layout
- Route file owns: auth side effects, navigation, token storage, Zustand update
- Service owns: Firebase API calls, user mapping

---

## 62. Firebase Error Code Mapping

Firebase throws errors with codes like `auth/wrong-password`. Map these to user-friendly messages.

`src/features/auth/utils.ts`:
```typescript
export function firebaseErrorToMessage(error: unknown): string {
  if (typeof error === 'object' && error !== null && 'code' in error) {
    const code = (error as { code: string }).code;
    const messages: Record<string, string> = {
      'auth/invalid-email':         'Please enter a valid email address.',
      'auth/user-not-found':        'No account found with this email.',
      'auth/wrong-password':        'Incorrect password. Please try again.',
      'auth/too-many-requests':     'Too many failed attempts. Please try again later.',
      'auth/user-disabled':         'This account has been disabled.',
      'auth/email-already-in-use':  'An account with this email already exists.',
      'auth/weak-password':         'Password must be at least 6 characters.',
      'auth/network-request-failed':'No internet connection. Please try again.',
    };
    return messages[code] ?? 'Something went wrong. Please try again.';
  }
  return 'Something went wrong. Please try again.';
}
```

**Never show raw Firebase error codes to users.** "auth/wrong-password" is a development message. "Incorrect password. Please try again." is what users see.

---

## 63. Protected Routes Pattern

`app/index.tsx` — the entry point. Redirects based on auth state:

```tsx
import { Redirect } from 'expo-router';
import { View } from 'react-native';
import { useAuthStatus } from '@/store';
import { Spinner } from '@/components/ui';
import { ROUTES } from '@/constants';

export default function Index() {
  const status = useAuthStatus();

  // While persisted auth is loading from AsyncStorage, show spinner
  if (status === 'idle' || status === 'loading') {
    return (
      <View className="flex-1 items-center justify-center bg-background">
        <Spinner size="lg" />
      </View>
    );
  }

  if (status === 'authenticated') {
    return <Redirect href={ROUTES.TASKS} />;
  }

  return <Redirect href={ROUTES.LOGIN} />;
}
```

**The `idle` state prevents flash of wrong screen:**

Without `idle`:
1. App opens → `status` is `unauthenticated` (default before storage loads)
2. Flash of login screen
3. Storage loads → `status` becomes `authenticated`
4. Navigate to tasks

With `idle`:
1. App opens → `status` is `idle` → show spinner
2. Storage loads → `status` becomes `authenticated` → redirect to tasks
3. User sees: spinner → tasks (no flash of login)

---

## 64. Secure Token Storage

`src/services/storage/secure-storage.ts`:
```typescript
import * as SecureStore from 'expo-secure-store';

const TOKEN_KEY = 'taskflow-auth-token';
const REFRESH_TOKEN_KEY = 'taskflow-refresh-token';

export const secureStorage = {
  setToken: async (token: string) => {
    await SecureStore.setItemAsync(TOKEN_KEY, token);
  },
  getToken: async (): Promise<string | null> => {
    return SecureStore.getItemAsync(TOKEN_KEY);
  },
  deleteToken: async () => {
    SecureStore.deleteItemAsync(TOKEN_KEY);
  },
  setRefreshToken: async (token: string) => {
    await SecureStore.setItemAsync(REFRESH_TOKEN_KEY, token);
  },
  getRefreshToken: async (): Promise<string | null> => {
    return SecureStore.getItemAsync(REFRESH_TOKEN_KEY);
  },
  deleteAll: async () => {
    await Promise.all([
      SecureStore.deleteItemAsync(TOKEN_KEY),
      SecureStore.deleteItemAsync(REFRESH_TOKEN_KEY),
    ]);
  },
};
```

```bash
npx expo install expo-secure-store
```

**SecureStore vs MMKV for tokens:**
- SecureStore → iOS Keychain / Android Keystore. Hardware-backed encryption. Data is protected even if the device is physically compromised. For credentials, auth tokens, anything that would cause account compromise if leaked.
- MMKV → Fast key-value store. App sandbox only. For preferences, cache, non-sensitive state.

**Rule:** Anything that could log someone into your app goes in SecureStore. Everything else goes in MMKV.

---

## 65. Token Refresh Flow

Firebase ID tokens expire after 1 hour. The app must refresh them silently.

Update the Axios request interceptor:
```typescript
apiClient.interceptors.request.use(
  async (config) => {
    let token = await secureStorage.getToken();

    if (token) {
      // Check if token is near expiry (Firebase tokens are JWTs)
      // Firebase's getIdToken(false) returns cached token if valid
      // Firebase's getIdToken(true) forces a refresh
      try {
        const freshToken = await firebaseAuth.getIdToken();
        if (freshToken && freshToken !== token) {
          await secureStorage.setToken(freshToken);
          token = freshToken;
        }
      } catch {
        // If refresh fails, proceed with existing token
        // The response interceptor's 401 handler will sign out if needed
      }
    }

    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }

    return config;
  },
);
```

**Firebase handles most token refresh automatically.** `getIdToken(false)` returns a cached valid token if available, or silently refreshes it if expired. You only need to call `getIdToken(true)` to force a refresh in specific scenarios (like after updating user profile).

---

## 66. Supabase Setup

Supabase provides PostgreSQL database + REST API + real-time subscriptions.

```bash
npm install @supabase/supabase-js
npx expo install expo-secure-store  # already installed
```

`src/services/api/supabase.ts`:
```typescript
import { createClient } from '@supabase/supabase-js';
import { env } from '@/lib/env';

export const supabase = createClient(
  env.supabaseUrl,
  env.supabaseAnonKey,
  {
    auth: {
      storage: {
        // Custom storage using SecureStore for Supabase's own auth tokens
        getItem: (key) => SecureStore.getItemAsync(key),
        setItem: (key, value) => SecureStore.setItemAsync(key, value),
        removeItem: (key) => SecureStore.deleteItemAsync(key),
      },
    },
  },
);
```

**Create the tasks table in Supabase Dashboard:**
```sql
CREATE TABLE tasks (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  title TEXT NOT NULL,
  description TEXT,
  status TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'in_progress', 'completed')),
  priority TEXT NOT NULL DEFAULT 'medium'
    CHECK (priority IN ('low', 'medium', 'high')),
  due_date TIMESTAMPTZ,
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Row Level Security — users can only see their own tasks
ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users can manage their own tasks"
  ON tasks FOR ALL
  USING (auth.uid() = user_id);
```

---

## 67. Task Types & Zod Schemas

`src/features/tasks/types.ts`:
```typescript
import { z } from 'zod';

export const TaskStatusSchema = z.enum(['pending', 'in_progress', 'completed']);
export const TaskPrioritySchema = z.enum(['low', 'medium', 'high']);

export const TaskSchema = z.object({
  id: z.string().uuid(),
  title: z.string().min(1),
  description: z.string().nullable(),
  status: TaskStatusSchema,
  priority: TaskPrioritySchema,
  due_date: z.string().datetime().nullable(),
  user_id: z.string().uuid(),
  created_at: z.string().datetime(),
  updated_at: z.string().datetime(),
});

export const CreateTaskSchema = z.object({
  title: z.string().min(1, 'Title is required').max(100, 'Title too long'),
  description: z.string().max(500).optional(),
  priority: TaskPrioritySchema.default('medium'),
  due_date: z.string().datetime().nullable().optional(),
});

export const UpdateTaskSchema = CreateTaskSchema.partial().extend({
  status: TaskStatusSchema.optional(),
});

export const TaskFiltersSchema = z.object({
  status: TaskStatusSchema.optional(),
  priority: TaskPrioritySchema.optional(),
  search: z.string().optional(),
  sortBy: z.enum(['created_at', 'due_date', 'priority']).optional(),
  sortOrder: z.enum(['asc', 'desc']).optional(),
});

export type Task = z.infer<typeof TaskSchema>;
export type CreateTaskInput = z.infer<typeof CreateTaskSchema>;
export type UpdateTaskInput = z.infer<typeof UpdateTaskSchema>;
export type TaskFilters = z.infer<typeof TaskFiltersSchema>;
export type TaskStatus = z.infer<typeof TaskStatusSchema>;
export type TaskPriority = z.infer<typeof TaskPrioritySchema>;
```

---

## 68. Tasks API Service

`src/features/tasks/services/tasks-api.ts`:
```typescript
import { supabase } from '@/services/api/supabase';
import { TaskSchema, type Task, type CreateTaskInput, type UpdateTaskInput, type TaskFilters } from '../types';
import { z } from 'zod';

export const tasksApi = {
  getAll: async (filters: TaskFilters = {}, page = 1, pageSize = 20): Promise<{ data: Task[]; count: number }> => {
    let query = supabase
      .from('tasks')
      .select('*', { count: 'exact' })
      .order(filters.sortBy ?? 'created_at', { ascending: filters.sortOrder === 'asc' })
      .range((page - 1) * pageSize, page * pageSize - 1);

    if (filters.status) query = query.eq('status', filters.status);
    if (filters.priority) query = query.eq('priority', filters.priority);
    if (filters.search) query = query.ilike('title', `%${filters.search}%`);

    const { data, error, count } = await query;
    if (error) throw new Error(error.message);

    return {
      data: z.array(TaskSchema).parse(data),
      count: count ?? 0,
    };
  },

  getById: async (id: string): Promise<Task> => {
    const { data, error } = await supabase
      .from('tasks')
      .select('*')
      .eq('id', id)
      .single();
    if (error) throw new Error(error.message);
    return TaskSchema.parse(data);
  },

  create: async (input: CreateTaskInput): Promise<Task> => {
    const { data, error } = await supabase
      .from('tasks')
      .insert(input)
      .select()
      .single();
    if (error) throw new Error(error.message);
    return TaskSchema.parse(data);
  },

  update: async (id: string, input: UpdateTaskInput): Promise<Task> => {
    const { data, error } = await supabase
      .from('tasks')
      .update({ ...input, updated_at: new Date().toISOString() })
      .eq('id', id)
      .select()
      .single();
    if (error) throw new Error(error.message);
    return TaskSchema.parse(data);
  },

  delete: async (id: string): Promise<void> => {
    const { error } = await supabase.from('tasks').delete().eq('id', id);
    if (error) throw new Error(error.message);
  },
};
```

---

## 69. useTasksQuery — TanStack Query

`src/features/tasks/hooks/useTasksQuery.ts`:
```typescript
import { useQuery, useSuspenseQuery } from '@tanstack/react-query';
import { queryKeys } from '@/services/api/query-keys';
import { tasksApi } from '../services/tasks-api';
import type { TaskFilters } from '../types';

export function useTasksQuery(filters: TaskFilters = {}) {
  return useQuery({
    queryKey: queryKeys.tasks.list(filters),
    queryFn: () => tasksApi.getAll(filters),
    staleTime: 1000 * 60 * 2, // 2 min — tasks change more frequently
  });
}

export function useTaskQuery(id: string) {
  return useQuery({
    queryKey: queryKeys.tasks.detail(id),
    queryFn: () => tasksApi.getById(id),
    enabled: Boolean(id), // don't fetch if id is empty
  });
}
```

**`enabled: Boolean(id)`** — TanStack Query won't fetch if `enabled` is false. This prevents firing a request with an empty `id` when the component first mounts before the param is set.

**`staleTime` per query** — overrides the global default for queries that need fresher data. Tasks are user-generated content that changes often, so 2 minutes is more appropriate than the 5-minute global default.

---

## 70. FlashList Task List

FlashList is Shopify's replacement for FlatList — dramatically faster for large lists.

```bash
npx expo install @shopify/flash-list
```

`src/features/tasks/screens/TasksListScreen.tsx`:
```tsx
import { View, RefreshControl } from 'react-native';
import { FlashList } from '@shopify/flash-list';
import { Screen, Text, Spinner } from '@/components/ui';
import { useTasksQuery } from '../hooks/useTasksQuery';
import { TaskCard } from '../components/TaskCard';
import { EmptyState } from '@/components/common';
import { ErrorState } from '@/components/common';
import type { Task, TaskFilters } from '../types';
import { memo, useCallback } from 'react';

interface TasksListScreenProps {
  filters?: TaskFilters;
}

export function TasksListScreen({ filters = {} }: TasksListScreenProps) {
  const { data, isLoading, isError, refetch, isRefetching } = useTasksQuery(filters);

  const renderItem = useCallback(
    ({ item }: { item: Task }) => <TaskCard task={item} />,
    [],
  );

  const keyExtractor = useCallback((item: Task) => item.id, []);

  if (isLoading) return <View className="flex-1 items-center justify-center"><Spinner /></View>;
  if (isError) return <ErrorState onRetry={refetch} />;

  return (
    <Screen padded={false} edges={['bottom']}>
      <FlashList
        data={data?.data ?? []}
        keyExtractor={keyExtractor}
        renderItem={renderItem}
        estimatedItemSize={80}
        contentContainerStyle={{ paddingHorizontal: 16, paddingTop: 8 }}
        ItemSeparatorComponent={() => <View className="h-2" />}
        ListEmptyComponent={<EmptyState title="No tasks" description="Create your first task to get started." />}
        refreshControl={
          <RefreshControl refreshing={isRefetching} onRefresh={refetch} />
        }
      />
    </Screen>
  );
}
```

**`estimatedItemSize`** — required by FlashList. Measure one rendered `TaskCard` and use its height. Wrong value = jumpy scrolling. Use the closest whole pixel value (e.g., if card is 76px, use 76).

**`useCallback` on renderItem and keyExtractor** — these functions are passed to FlashList. Without `useCallback`, new function references on each render could cause unnecessary list re-renders.

**FlatList vs FlashList:**

| | FlatList | FlashList |
|---|---|---|
| Performance | Good | ~10x better |
| `estimatedItemSize` | Optional | Required |
| API compatibility | — | ~95% same |
| When to use | < 20 items | Always |

---

## 71. TaskCard Component

`src/features/tasks/components/TaskCard.tsx`:
```tsx
import { Pressable, View } from 'react-native';
import { useRouter } from 'expo-router';
import { memo } from 'react';
import { Text, Badge } from '@/components/ui';
import { ROUTES } from '@/constants';
import { cn } from '@/lib';
import type { Task } from '../types';

interface TaskCardProps {
  task: Task;
}

const priorityConfig = {
  low:    { label: 'Low',    variant: 'default' as const },
  medium: { label: 'Medium', variant: 'warning' as const },
  high:   { label: 'High',   variant: 'danger'  as const },
};

const statusConfig = {
  pending:     { label: 'Pending',     dotClass: 'bg-muted-foreground' },
  in_progress: { label: 'In Progress', dotClass: 'bg-primary-500'      },
  completed:   { label: 'Completed',   dotClass: 'bg-success'          },
};

export const TaskCard = memo(function TaskCard({ task }: TaskCardProps) {
  const router = useRouter();
  const priority = priorityConfig[task.priority];
  const status = statusConfig[task.status];
  const isCompleted = task.status === 'completed';

  return (
    <Pressable
      accessibilityRole="button"
      accessibilityLabel={`Task: ${task.title}. Priority: ${priority.label}. Status: ${status.label}.`}
      onPress={() => router.push(ROUTES.TASK_DETAIL(task.id))}
      className="rounded-lg border border-border bg-card p-4 active:opacity-80"
    >
      <View className="flex-row items-start justify-between gap-2">
        <Text
          variant="label"
          className={cn('flex-1', isCompleted && 'line-through text-muted-foreground')}
          numberOfLines={2}
        >
          {task.title}
        </Text>
        <Badge label={priority.label} variant={priority.variant} />
      </View>

      {task.description ? (
        <Text
          variant="body-sm"
          color="muted"
          numberOfLines={1}
          className="mt-1"
        >
          {task.description}
        </Text>
      ) : null}

      <View className="mt-3 flex-row items-center gap-2">
        <View className={cn('h-2 w-2 rounded-full', status.dotClass)} />
        <Text variant="caption" color="muted">{status.label}</Text>
        {task.due_date ? (
          <>
            <Text variant="caption" color="muted">·</Text>
            <Text variant="caption" color="muted">
              Due {new Date(task.due_date).toLocaleDateString()}
            </Text>
          </>
        ) : null}
      </View>
    </Pressable>
  );
});
```

**`memo()`** — wraps the component in React.memo. TaskCard only re-renders when its `task` prop changes. In a list of 100 tasks, this means updating one task only re-renders one card, not all 100.

**Comprehensive `accessibilityLabel`** — screen readers read the entire label: "Task: Fix login bug. Priority: High. Status: In Progress." Users don't have to explore sub-elements to understand the card.

**`numberOfLines={2}`** — truncates long titles at 2 lines with an ellipsis. Keeps the list uniform regardless of title length.

---

## 72. Empty State Component

`src/components/common/EmptyState.tsx`:
```tsx
import { View } from 'react-native';
import { Text, Button } from '@/components/ui';
import { ClipboardList } from 'lucide-react-native';
import { useThemeStore } from '@/store';
import { colors } from '@/constants';

interface EmptyStateProps {
  title?: string;
  description?: string;
  actionLabel?: string;
  onAction?: () => void;
}

export function EmptyState({
  title = 'Nothing here yet',
  description = 'Get started by adding something new.',
  actionLabel,
  onAction,
}: EmptyStateProps) {
  const resolvedTheme = useThemeStore((s) => s.resolvedTheme);
  const iconColor = colors[resolvedTheme].mutedForeground;

  return (
    <View className="flex-1 items-center justify-center gap-4 px-8 py-16">
      <ClipboardList size={48} color={iconColor} strokeWidth={1.5} />
      <View className="items-center gap-1">
        <Text variant="h4">{title}</Text>
        <Text color="muted" className="text-center">{description}</Text>
      </View>
      {actionLabel && onAction ? (
        <Button label={actionLabel} onPress={onAction} variant="primary" />
      ) : null}
    </View>
  );
}
```

---

## 73. Error State Component

`src/components/common/ErrorState.tsx`:
```tsx
import { View } from 'react-native';
import { Text, Button } from '@/components/ui';
import { AlertCircle } from 'lucide-react-native';
import { useThemeStore } from '@/store';
import { colors } from '@/constants';

interface ErrorStateProps {
  title?: string;
  description?: string;
  onRetry?: () => void;
}

export function ErrorState({
  title = 'Something went wrong',
  description = 'An error occurred. Please try again.',
  onRetry,
}: ErrorStateProps) {
  const resolvedTheme = useThemeStore((s) => s.resolvedTheme);
  const iconColor = colors[resolvedTheme].danger;

  return (
    <View className="flex-1 items-center justify-center gap-4 px-8 py-16">
      <AlertCircle size={48} color={iconColor} />
      <View className="items-center gap-1">
        <Text variant="h4">{title}</Text>
        <Text color="muted" className="text-center">{description}</Text>
      </View>
      {onRetry ? (
        <Button label="Try Again" onPress={onRetry} variant="secondary" />
      ) : null}
    </View>
  );
}
```

Update `src/components/common/index.ts`:
```typescript
export { EmptyState } from './EmptyState';
export { ErrorState } from './ErrorState';
```

---

## 74. Skeleton Loading

Show the shape of content before data loads. Better UX than a spinner in the middle of the screen.

`src/components/common/Skeleton.tsx`:
```tsx
import { View, type ViewProps } from 'react-native';
import Animated, { useSharedValue, useAnimatedStyle, withRepeat, withTiming, Easing } from 'react-native-reanimated';
import { useEffect } from 'react';
import { cn } from '@/lib';

interface SkeletonProps extends ViewProps {
  className?: string;
}

export function Skeleton({ className, ...rest }: SkeletonProps) {
  const opacity = useSharedValue(1);

  useEffect(() => {
    opacity.value = withRepeat(
      withTiming(0.4, { duration: 800, easing: Easing.inOut(Easing.ease) }),
      -1, // repeat forever
      true, // reverse direction (ping-pong)
    );
  }, []);

  const animatedStyle = useAnimatedStyle(() => ({ opacity: opacity.value }));

  return (
    <Animated.View
      style={animatedStyle}
      className={cn('rounded-md bg-muted', className)}
      {...rest}
    />
  );
}

// Pre-built TaskCard skeleton
export function TaskCardSkeleton() {
  return (
    <View className="rounded-lg border border-border bg-card p-4">
      <View className="flex-row items-center justify-between">
        <Skeleton className="h-4 w-3/4" />
        <Skeleton className="h-5 w-14 rounded-full" />
      </View>
      <Skeleton className="mt-2 h-3 w-1/2" />
      <View className="mt-3 flex-row gap-2">
        <Skeleton className="h-3 w-3 rounded-full" />
        <Skeleton className="h-3 w-20" />
      </View>
    </View>
  );
}
```

**`withRepeat(-1, true)`** — `-1` means repeat forever. `true` reverses direction on each repeat (ping-pong effect). The result is a smooth pulse: full opacity → 40% → full → 40% → ...

**Skeleton mirrors the real card's structure** — same border-radius, same padding, same element positions. When data loads and the skeleton is replaced by the real card, there's no layout shift.

---

## 75. Pull-to-Refresh

```tsx
import { RefreshControl } from 'react-native';
import { useThemeStore } from '@/store';
import { colors } from '@/constants';

// Inside FlashList or ScrollView:
const resolvedTheme = useThemeStore((s) => s.resolvedTheme);
const themeColors = colors[resolvedTheme];

<FlashList
  // ...
  refreshControl={
    <RefreshControl
      refreshing={isRefetching}       // from useQuery's isRefetching flag
      onRefresh={refetch}             // from useQuery's refetch function
      tintColor={themeColors.primary} // iOS spinner color
      colors={[themeColors.primary]}  // Android spinner color (array)
    />
  }
/>
```

**`isRefetching` vs `isFetching`:**
- `isFetching` — true on initial load AND on background refetches
- `isRefetching` — true ONLY on background refetches (user-triggered or background)
- Use `isRefetching` for `refreshing` prop so the spinner doesn't show on initial load (which has its own skeleton loading state)

---

## 76. Create Task — useMutation

`src/features/tasks/hooks/useCreateTask.ts`:
```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { queryKeys } from '@/services/api/query-keys';
import { tasksApi } from '../services/tasks-api';
import type { CreateTaskInput } from '../types';

export function useCreateTask() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (input: CreateTaskInput) => tasksApi.create(input),
    onSuccess: () => {
      // After successful creation, invalidate the tasks list
      // This causes any mounted TasksList to refetch
      queryClient.invalidateQueries({ queryKey: queryKeys.tasks.lists() });
    },
    onError: (error) => {
      console.error('Failed to create task:', error);
    },
  });
}
```

**Usage in form component:**
```tsx
const { mutate: createTask, isPending } = useCreateTask();

const onSubmit = (data: CreateTaskInput) => {
  createTask(data, {
    onSuccess: () => router.back(),
    onError: (error) => setError(error.message),
  });
};
```

---

## 77. Optimistic Updates

Show the result of an action immediately in the UI before the server confirms it. If the server rejects, roll back.

```typescript
export function useCreateTask() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (input: CreateTaskInput) => tasksApi.create(input),

    onMutate: async (newTaskInput) => {
      // 1. Cancel any in-flight queries to avoid overwriting optimistic update
      await queryClient.cancelQueries({ queryKey: queryKeys.tasks.lists() });

      // 2. Snapshot current cache (for rollback)
      const previousData = queryClient.getQueriesData({ queryKey: queryKeys.tasks.lists() });

      // 3. Optimistically add the new task to the cache
      const optimisticTask = {
        id: `temp-${Date.now()}`,
        ...newTaskInput,
        status: 'pending' as const,
        user_id: '',
        created_at: new Date().toISOString(),
        updated_at: new Date().toISOString(),
        due_date: newTaskInput.due_date ?? null,
        description: newTaskInput.description ?? null,
      };

      queryClient.setQueriesData(
        { queryKey: queryKeys.tasks.lists() },
        (old: any) => old
          ? { ...old, data: [optimisticTask, ...old.data] }
          : { data: [optimisticTask], count: 1 },
      );

      return { previousData };
    },

    onError: (error, variables, context) => {
      // 4. On error, restore the previous cache snapshot
      if (context?.previousData) {
        context.previousData.forEach(([queryKey, data]) => {
          queryClient.setQueryData(queryKey, data);
        });
      }
    },

    onSettled: () => {
      // 5. Always refetch after mutation (success or error) to sync with server
      queryClient.invalidateQueries({ queryKey: queryKeys.tasks.lists() });
    },
  });
}
```

**Why 3 callbacks:**
- `onMutate` — fires before request. Update UI immediately. Save snapshot.
- `onError` — fires if request fails. Restore snapshot. User sees original state.
- `onSettled` — fires after either success or error. Always refetch to ensure server and client are in sync.

---

## 78. Task Detail Screen

`src/features/tasks/screens/TaskDetailScreen.tsx`:
```tsx
import { View } from 'react-native';
import { memo } from 'react';
import { Screen, Text, Button, Card, Badge, Spinner } from '@/components/ui';
import { ErrorState } from '@/components/common';
import { useTaskQuery } from '../hooks/useTasksQuery';
import { useDeleteTask } from '../hooks/useDeleteTask';
import { useRouter } from 'expo-router';
import type { Task } from '../types';

interface TaskDetailScreenProps {
  taskId: string;
  onEdit: () => void;
}

export function TaskDetailScreen({ taskId, onEdit }: TaskDetailScreenProps) {
  const { data: task, isLoading, isError, refetch } = useTaskQuery(taskId);
  const router = useRouter();

  if (isLoading) return <View className="flex-1 items-center justify-center"><Spinner /></View>;
  if (isError || !task) return <ErrorState onRetry={refetch} />;

  return (
    <Screen scrollable>
      <View className="gap-4 py-4">
        <View className="flex-row items-start justify-between gap-3">
          <Text variant="h2" className="flex-1">{task.title}</Text>
          <Badge label={task.priority} variant={task.priority === 'high' ? 'danger' : task.priority === 'medium' ? 'warning' : 'default'} />
        </View>

        {task.description ? (
          <Card variant="outlined">
            <Text variant="label" color="muted" className="mb-1">Description</Text>
            <Text>{task.description}</Text>
          </Card>
        ) : null}

        <Card variant="outlined">
          <View className="gap-3">
            <View className="flex-row justify-between">
              <Text color="muted">Status</Text>
              <Text className="font-medium capitalize">{task.status.replace('_', ' ')}</Text>
            </View>
            {task.due_date ? (
              <View className="flex-row justify-between">
                <Text color="muted">Due Date</Text>
                <Text className="font-medium">
                  {new Date(task.due_date).toLocaleDateString('en-US', { dateStyle: 'medium' })}
                </Text>
              </View>
            ) : null}
            <View className="flex-row justify-between">
              <Text color="muted">Created</Text>
              <Text className="font-medium">
                {new Date(task.created_at).toLocaleDateString('en-US', { dateStyle: 'medium' })}
              </Text>
            </View>
          </View>
        </Card>

        <View className="gap-2">
          <Button label="Edit Task" variant="primary" onPress={onEdit} fullWidth />
          <Button label="Delete Task" variant="danger" onPress={() => {/* see topic 80 */}} fullWidth />
        </View>
      </View>
    </Screen>
  );
}
```

---

## 79. Edit Task

```typescript
// src/features/tasks/hooks/useUpdateTask.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { queryKeys } from '@/services/api/query-keys';
import { tasksApi } from '../services/tasks-api';
import type { UpdateTaskInput } from '../types';

export function useUpdateTask(taskId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (input: UpdateTaskInput) => tasksApi.update(taskId, input),
    onSuccess: (updatedTask) => {
      // Update the specific task in cache immediately
      queryClient.setQueryData(queryKeys.tasks.detail(taskId), updatedTask);
      // Invalidate lists (order/filters may have changed)
      queryClient.invalidateQueries({ queryKey: queryKeys.tasks.lists() });
    },
  });
}
```

**Setting cache directly on success:** After an update, we know the exact updated task from the server response. We can set it in the detail cache immediately (`setQueryData`) rather than triggering a refetch. This is more efficient — one less network request.

---

## 80. Delete Task with Confirmation

```typescript
// src/features/tasks/hooks/useDeleteTask.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { queryKeys } from '@/services/api/query-keys';
import { tasksApi } from '../services/tasks-api';

export function useDeleteTask() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (id: string) => tasksApi.delete(id),
    onSuccess: (_, deletedId) => {
      // Remove from detail cache
      queryClient.removeQueries({ queryKey: queryKeys.tasks.detail(deletedId) });
      // Invalidate lists
      queryClient.invalidateQueries({ queryKey: queryKeys.tasks.lists() });
    },
  });
}
```

**Confirmation dialog before delete:**
```tsx
import { Alert } from 'react-native';

const { mutate: deleteTask } = useDeleteTask();

const handleDelete = () => {
  Alert.alert(
    'Delete Task',
    'This action cannot be undone. Are you sure?',
    [
      { text: 'Cancel', style: 'cancel' },
      {
        text: 'Delete',
        style: 'destructive',
        onPress: () => {
          deleteTask(task.id, {
            onSuccess: () => router.back(),
          });
        },
      },
    ],
  );
};
```

`Alert.alert` is native — it renders the platform's native confirmation dialog (iOS: action sheet, Android: alert dialog). Never use a custom JS modal for destructive actions. The platform dialog is more trustworthy, more accessible, and follows platform conventions.

---

## 81. Search with Debounce

`src/hooks/useDebounce.ts`:
```typescript
import { useState, useEffect } from 'react';

export function useDebounce<T>(value: T, delay: number = 300): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(timer); // cleanup cancels the pending timer
  }, [value, delay]);

  return debouncedValue;
}
```

**Usage in tasks screen:**
```tsx
const [searchText, setSearchText] = useState('');
const debouncedSearch = useDebounce(searchText, 300);

// Query only fires when debouncedSearch changes (300ms after typing stops)
const { data } = useTasksQuery({ search: debouncedSearch });

return (
  <Input
    value={searchText}
    onChangeText={setSearchText}  // immediate local update
    placeholder="Search tasks..."
  />
);
```

**Why debounce:** Without it, every keystroke triggers an API request. Typing "design" fires 6 requests: "d", "de", "des", "desi", "desig", "design". With 300ms debounce, only one request fires (300ms after the user stops typing "design").

---

## 82. Filter & Sort

Filters are stored in Zustand (so they persist when navigating away and back) and passed to TanStack Query (included in the query key, so different filter combinations have separate caches).

```typescript
// src/features/tasks/store/tasks-filters.store.ts
import { create } from 'zustand';
import type { TaskFilters } from '../types';

interface TasksFiltersState {
  filters: TaskFilters;
  setFilter: <K extends keyof TaskFilters>(key: K, value: TaskFilters[K]) => void;
  resetFilters: () => void;
}

export const useTasksFiltersStore = create<TasksFiltersState>((set) => ({
  filters: {},
  setFilter: (key, value) =>
    set((state) => ({ filters: { ...state.filters, [key]: value } })),
  resetFilters: () => set({ filters: {} }),
}));
```

**Usage:**
```tsx
const { filters, setFilter, resetFilters } = useTasksFiltersStore();
const { data } = useTasksQuery(filters);

// Changing a filter automatically triggers a refetch with the new filter
// (because filters are part of the query key)
<Button label="Show Completed" onPress={() => setFilter('status', 'completed')} />
<Button label="High Priority"  onPress={() => setFilter('priority', 'high')} />
<Button label="Clear Filters"  onPress={resetFilters} />
```

---

## 83. Infinite Scroll — useInfiniteQuery

Load tasks in pages. Load next page when user scrolls near the bottom.

```typescript
// src/features/tasks/hooks/useInfiniteTasksQuery.ts
import { useInfiniteQuery } from '@tanstack/react-query';
import { queryKeys } from '@/services/api/query-keys';
import { tasksApi } from '../services/tasks-api';
import type { TaskFilters } from '../types';

const PAGE_SIZE = 20;

export function useInfiniteTasksQuery(filters: TaskFilters = {}) {
  return useInfiniteQuery({
    queryKey: queryKeys.tasks.list(filters),
    queryFn: ({ pageParam = 1 }) => tasksApi.getAll(filters, pageParam, PAGE_SIZE),
    initialPageParam: 1,
    getNextPageParam: (lastPage, allPages) => {
      const totalLoaded = allPages.length * PAGE_SIZE;
      return totalLoaded < lastPage.count ? allPages.length + 1 : undefined;
      // undefined = no more pages
    },
  });
}
```

**In FlashList:**
```tsx
const {
  data,
  fetchNextPage,
  hasNextPage,
  isFetchingNextPage,
} = useInfiniteTasksQuery(filters);

// Flatten pages into a single array
const tasks = data?.pages.flatMap((page) => page.data) ?? [];

<FlashList
  data={tasks}
  onEndReached={() => {
    if (hasNextPage && !isFetchingNextPage) fetchNextPage();
  }}
  onEndReachedThreshold={0.3}
  ListFooterComponent={
    isFetchingNextPage ? <Spinner className="my-4" /> : null
  }
/>
```

**`onEndReachedThreshold={0.3}`** — triggers `onEndReached` when the user has scrolled to within 30% of the bottom. Starts loading the next page before they actually reach the end, creating a seamless experience.

---

## 84. Permissions Handling Pattern

Native features require user permission. Denying permission should show helpful UI, not a silent failure.

```typescript
// src/lib/permissions.ts
import { Alert, Linking } from 'react-native';

export async function requestPermissionWithFallback(
  requestFn: () => Promise<{ status: string }>,
  permissionName: string,
): Promise<boolean> {
  const { status } = await requestFn();

  if (status === 'granted') return true;

  if (status === 'denied') {
    // First denial — can ask again on next attempt
    return false;
  }

  // 'blocked' — user denied and selected "Don't ask again"
  // Must direct to Settings
  Alert.alert(
    `${permissionName} Required`,
    `Please enable ${permissionName} access in your Settings to use this feature.`,
    [
      { text: 'Cancel', style: 'cancel' },
      {
        text: 'Open Settings',
        onPress: () => Linking.openSettings(),
      },
    ],
  );
  return false;
}
```

**Permission states:**
- `granted` — user allowed. Proceed.
- `denied` — user said no this time. Can ask again.
- `blocked` — user said "Don't ask again". Can only direct to Settings.

---

## 85. Image Picker — Gallery

```typescript
// src/features/tasks/hooks/useImagePicker.ts
import * as ImagePicker from 'expo-image-picker';
import { requestPermissionWithFallback } from '@/lib/permissions';

export function useImagePicker() {
  const pickFromGallery = async () => {
    const granted = await requestPermissionWithFallback(
      () => ImagePicker.requestMediaLibraryPermissionsAsync(),
      'Photo Library',
    );

    if (!granted) return null;

    const result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ImagePicker.MediaTypeOptions.Images,
      allowsEditing: true,
      aspect: [4, 3],
      quality: 0.8,         // 80% quality — good balance of size vs clarity
      allowsMultipleSelection: false,
    });

    if (result.canceled) return null;
    return result.assets[0];
  };

  return { pickFromGallery };
}
```

```bash
npx expo install expo-image-picker
```

**Add to `app.json`:**
```json
{
  "plugins": [
    ["expo-image-picker", {
      "photosPermission": "TaskFlow uses your photo library to attach images to tasks.",
      "cameraPermission": "TaskFlow uses your camera to capture images for tasks."
    }]
  ]
}
```

---

## 86. Camera Capture

```typescript
const captureFromCamera = async () => {
  const granted = await requestPermissionWithFallback(
    () => ImagePicker.requestCameraPermissionsAsync(),
    'Camera',
  );

  if (!granted) return null;

  const result = await ImagePicker.launchCameraAsync({
    allowsEditing: true,
    aspect: [4, 3],
    quality: 0.8,
  });

  if (result.canceled) return null;
  return result.assets[0];
};
```

**Unified picker UI:**
```tsx
const handleAttach = () => {
  Alert.alert(
    'Add Attachment',
    'Choose a source',
    [
      { text: 'Camera', onPress: captureFromCamera },
      { text: 'Gallery', onPress: pickFromGallery },
      { text: 'Cancel', style: 'cancel' },
    ],
  );
};
```

---

## 87. Image Compression Before Upload

Large images slow uploads and waste storage. Compress before uploading.

```typescript
import * as ImageManipulator from 'expo-image-manipulator';

export async function compressImage(uri: string): Promise<string> {
  const result = await ImageManipulator.manipulateAsync(
    uri,
    [{ resize: { width: 1200 } }], // resize to max 1200px wide
    {
      compress: 0.8,  // 80% quality
      format: ImageManipulator.SaveFormat.JPEG,
    },
  );
  return result.uri;
}
```

```bash
npx expo install expo-image-manipulator
```

**Usage in upload flow:**
```typescript
const pickedImage = await pickFromGallery();
if (!pickedImage) return;

const compressedUri = await compressImage(pickedImage.uri);
// Upload compressedUri instead of pickedImage.uri
```

---

## 88. File Upload to Supabase Storage

```typescript
// src/features/tasks/services/attachments-api.ts
import { supabase } from '@/services/api/supabase';
import * as FileSystem from 'expo-file-system';

export async function uploadAttachment(
  taskId: string,
  localUri: string,
  onProgress?: (progress: number) => void,
): Promise<string> {
  const fileName = `${taskId}/${Date.now()}.jpg`;

  // Read file as base64
  const base64 = await FileSystem.readAsStringAsync(localUri, {
    encoding: FileSystem.EncodingType.Base64,
  });

  const { error } = await supabase.storage
    .from('attachments')
    .upload(fileName, decode(base64), {
      contentType: 'image/jpeg',
      upsert: false,
    });

  if (error) throw new Error(error.message);

  const { data: urlData } = supabase.storage
    .from('attachments')
    .getPublicUrl(fileName);

  return urlData.publicUrl;
}

// base64 string → Uint8Array
function decode(base64: string): Uint8Array {
  const binaryStr = atob(base64);
  const bytes = new Uint8Array(binaryStr.length);
  for (let i = 0; i < binaryStr.length; i++) {
    bytes[i] = binaryStr.charCodeAt(i);
  }
  return bytes;
}
```

```bash
npx expo install expo-file-system
```

---

## 89. Upload Progress Tracking

Supabase's JS client doesn't expose upload progress directly. Use `fetch` with a custom upload:

```typescript
export async function uploadWithProgress(
  file: { uri: string; type: string; name: string },
  onProgress: (percent: number) => void,
): Promise<string> {
  const formData = new FormData();
  formData.append('file', {
    uri: file.uri,
    type: file.type,
    name: file.name,
  } as any);

  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();

    xhr.upload.addEventListener('progress', (event) => {
      if (event.lengthComputable) {
        onProgress(Math.round((event.loaded / event.total) * 100));
      }
    });

    xhr.onload = () => resolve(JSON.parse(xhr.responseText).url);
    xhr.onerror = () => reject(new Error('Upload failed'));

    xhr.open('POST', `${env.supabaseUrl}/storage/v1/object/attachments/${file.name}`);
    xhr.setRequestHeader('Authorization', `Bearer ${env.supabaseAnonKey}`);
    xhr.send(formData);
  });
}
```

**Progress UI:**
```tsx
const [progress, setProgress] = useState(0);
const [uploading, setUploading] = useState(false);

// In upload handler:
setUploading(true);
await uploadWithProgress(file, setProgress);
setUploading(false);

// In JSX:
{uploading && (
  <View className="h-1 w-full rounded-full bg-muted">
    <View
      className="h-1 rounded-full bg-primary-500"
      style={{ width: `${progress}%` }}
    />
  </View>
)}
```

---

## 90. expo-image for Rendering Attachments

```tsx
import { Image } from 'expo-image';

function TaskAttachment({ url, blurhash }: { url: string; blurhash?: string }) {
  return (
    <Image
      source={url}
      placeholder={{ blurhash }}   // colored blur while loading
      contentFit="cover"
      transition={300}              // 300ms fade-in
      priority="normal"
      style={{ width: '100%', height: 200, borderRadius: 8 }}
      accessibilityLabel="Task attachment"
    />
  );
}
```

**`blurhash`** — a compact string that encodes a blurred preview of an image. Generate it on the server when the image is uploaded. Store alongside the URL. On the client, `expo-image` renders the blurred preview instantly from the string, then fades to the real image when it loads.

---

## 91. MMKV Storage Swap

MMKV is a C++ key-value store with JS bindings. ~10x faster than AsyncStorage for read operations.

```bash
npx expo install react-native-mmkv
```

Update `src/services/storage/persist-storage.ts`:
```typescript
import { MMKV } from 'react-native-mmkv';
import { createJSONStorage } from 'zustand/middleware';

// Create a dedicated MMKV instance for Zustand state
const mmkvStore = new MMKV({ id: 'taskflow-store' });

// Adapter matching the interface Zustand's persist middleware expects
export const persistStorage = createJSONStorage(() => ({
  setItem: (key: string, value: string) => {
    mmkvStore.set(key, value);
  },
  getItem: (key: string) => {
    const value = mmkvStore.getString(key);
    return value ?? null;
  },
  removeItem: (key: string) => {
    mmkvStore.delete(key);
  },
}));
```

**No other changes needed.** All Zustand stores use `persistStorage`. Swapping the adapter here updates all stores at once. This is why we centralized the adapter in a single file.

**When to use MMKV vs SecureStore:**
- MMKV — fast, non-sensitive data. Theme preferences, cached responses, non-auth settings. Data is encrypted at rest (MMKV supports encryption) but accessible to the app process.
- SecureStore — auth tokens, credentials. Hardware-backed Keychain (iOS) / Keystore (Android). Survives app reinstall. Only accessible to your app.

---

## 92. TanStack Query Persistence

Persist TanStack Query's cache to MMKV so data loads instantly on app restart (before re-fetching from server).

```bash
npm install @tanstack/query-async-storage-persister @tanstack/react-query-persist-client
```

Update `app/_layout.tsx`:
```tsx
import { PersistQueryClientProvider } from '@tanstack/react-query-persist-client';
import { createAsyncStoragePersister } from '@tanstack/query-async-storage-persister';
import { MMKV } from 'react-native-mmkv';

const queryMmkv = new MMKV({ id: 'taskflow-query-cache' });

const persister = createAsyncStoragePersister({
  storage: {
    getItem: (key) => queryMmkv.getString(key) ?? null,
    setItem: (key, value) => queryMmkv.set(key, value),
    removeItem: (key) => queryMmkv.delete(key),
  },
});

// Replace QueryClientProvider with PersistQueryClientProvider
<PersistQueryClientProvider
  client={queryClient}
  persistOptions={{ persister, maxAge: 1000 * 60 * 60 * 24 }} // 24 hours
>
```

**`maxAge: 24 hours`** — cached data older than 24 hours is discarded on restart. After that, the app fetches fresh data on next mount.

**Result:** User opens the app → immediately sees their task list (from cache) → data refetches in background → list updates silently if anything changed.

---

## 93. Network Status Hook

`src/hooks/useNetworkStatus.ts`:
```typescript
import NetInfo, { type NetInfoState } from '@react-native-community/netinfo';
import { useEffect, useState } from 'react';
import { useUIStore } from '@/store';

export function useNetworkStatus() {
  const setOffline = useUIStore((s) => s.setOffline);

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener((state: NetInfoState) => {
      const isOffline = !state.isConnected || !state.isInternetReachable;
      setOffline(isOffline ?? false);
    });

    return unsubscribe;
  }, [setOffline]);
}
```

```bash
npx expo install @react-native-community/netinfo
```

**Mount in root layout** to monitor network status app-wide:
```tsx
// In app/_layout.tsx
import { useNetworkStatus } from '@/hooks';

export default function RootLayout() {
  useNetworkStatus(); // starts listening immediately
  // ...
}
```

---

## 94. Offline Banner Component

`src/components/common/OfflineBanner.tsx`:
```tsx
import { View } from 'react-native';
import Animated, { useSharedValue, useAnimatedStyle, withTiming } from 'react-native-reanimated';
import { useEffect } from 'react';
import { Text } from '@/components/ui';
import { WifiOff } from 'lucide-react-native';
import { useUIStore } from '@/store';

export function OfflineBanner() {
  const isOffline = useUIStore((s) => s.isOffline);
  const height = useSharedValue(0);

  useEffect(() => {
    height.value = withTiming(isOffline ? 44 : 0, { duration: 300 });
  }, [isOffline]);

  const animatedStyle = useAnimatedStyle(() => ({
    height: height.value,
    overflow: 'hidden',
  }));

  return (
    <Animated.View style={animatedStyle}>
      <View className="flex-row items-center justify-center gap-2 bg-warning px-4 py-2">
        <WifiOff size={16} color="#000" />
        <Text variant="label" className="text-foreground">
          You're offline. Changes will sync when reconnected.
        </Text>
      </View>
    </Animated.View>
  );
}
```

**Mount between root layout and content.** When offline, the banner smoothly slides down. When reconnected, it slides back up.

---

## 95. Offline Mutation Queue

TanStack Query automatically queues mutations when offline and retries them when reconnected.

```typescript
// Configure in QueryClient
export const queryClient = new QueryClient({
  defaultOptions: {
    mutations: {
      retry: 3,
      retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 30000), // exponential backoff
      networkMode: 'offlineFirst', // queue mutations when offline
    },
  },
});
```

**`networkMode: 'offlineFirst'`** — mutations are queued (not failed) when offline. When the device reconnects, TanStack Query automatically retries them in order.

**Manual queue observation:**
```typescript
import { onlineManager } from '@tanstack/react-query';

// When app foregrounds or network returns
const unsubscribe = NetInfo.addEventListener((state) => {
  onlineManager.setOnline(state.isConnected ?? false);
});
```

---

## 96. Cache Invalidation on Reconnect

When the device comes back online, stale queries should refetch immediately.

```typescript
import { focusManager } from '@tanstack/react-query';
import { AppState } from 'react-native';

// Refetch when app comes to foreground
AppState.addEventListener('change', (status) => {
  if (status === 'active') {
    focusManager.setFocused(true);
  } else {
    focusManager.setFocused(false);
  }
});

// Refetch when network returns
NetInfo.addEventListener((state) => {
  if (state.isConnected) {
    queryClient.invalidateQueries();
  }
});
```

---

## 97. FCM Setup — Android

Firebase Cloud Messaging (FCM) sends push notifications to Android devices.

1. In Firebase Console → Project Settings → Cloud Messaging → Server Key → copy
2. `google-services.json` already placed in project root (from Topic 55)
3. Expo config plugin already added (`@react-native-firebase/app`)

Install messaging:
```bash
npx expo install @react-native-firebase/messaging
```

Add to plugins in `app.json`:
```json
"@react-native-firebase/messaging"
```

**Android requires no additional configuration** — `google-services.json` contains all necessary keys. The Firebase config plugin handles the native Android setup.

---

## 98. APNs Setup — iOS

Apple Push Notification Service (APNs) delivers push notifications to iOS devices.

1. Go to [developer.apple.com](https://developer.apple.com) → Certificates, Identifiers & Profiles
2. Create an APNs Auth Key (`.p8` file) — one key works for all your apps
3. In Firebase Console → Project Settings → Cloud Messaging → iOS app → Upload APNs Auth Key

EAS handles the rest — it configures provisioning profiles with push notification entitlements automatically when you build.

**Add push notification capability to `app.json`:**
```json
{
  "ios": {
    "bundleIdentifier": "com.taskflow.app",
    "entitlements": {
      "aps-environment": "production"
    }
  }
}
```

---

## 99. Request Notification Permission

```typescript
// src/services/notifications/permissions.ts
import messaging from '@react-native-firebase/messaging';
import { Alert, Linking, Platform } from 'react-native';

export async function requestNotificationPermission(): Promise<boolean> {
  if (Platform.OS === 'ios') {
    const status = await messaging().requestPermission();
    return (
      status === messaging.AuthorizationStatus.AUTHORIZED ||
      status === messaging.AuthorizationStatus.PROVISIONAL
    );
  }

  if (Platform.OS === 'android') {
    // Android 13+ requires explicit permission
    const { status } = await messaging().requestPermission();
    return status === messaging.AuthorizationStatus.AUTHORIZED;
  }

  return false;
}
```

**When to request:** Don't ask for notifications on first app launch. Ask in context — when the user enables a feature that uses notifications (reminders, team mentions, due date alerts). Higher acceptance rate than a cold permission dialog on launch.

---

## 100. FCM Token Registration & Refresh

```typescript
// src/services/notifications/token.ts
import messaging from '@react-native-firebase/messaging';
import { supabase } from '@/services/api/supabase';

export async function registerFCMToken(userId: string): Promise<void> {
  const token = await messaging().getToken();
  await saveFCMToken(userId, token);
}

async function saveFCMToken(userId: string, token: string): Promise<void> {
  await supabase
    .from('device_tokens')
    .upsert({ user_id: userId, token, platform: Platform.OS, updated_at: new Date().toISOString() });
}

// Listen for token refreshes (token can change after reinstall, app restore)
export function listenForTokenRefresh(userId: string): () => void {
  return messaging().onTokenRefresh((newToken) => {
    saveFCMToken(userId, newToken);
  });
}
```

**Call on sign in:**
```typescript
// In auth success handler:
await registerFCMToken(user.id);
const unsubscribe = listenForTokenRefresh(user.id);
// Store unsubscribe — call on sign out to stop listening
```

**Create device_tokens table in Supabase:**
```sql
CREATE TABLE device_tokens (
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  token TEXT NOT NULL,
  platform TEXT NOT NULL,
  updated_at TIMESTAMPTZ DEFAULT now(),
  PRIMARY KEY (user_id, token)
);
```

---

## 101. Foreground Notification Handler

When the app is open, incoming notifications are not shown by default. Handle them manually.

```typescript
// In app/_layout.tsx
import messaging from '@react-native-firebase/messaging';

// Handle foreground messages
useEffect(() => {
  const unsubscribe = messaging().onMessage(async (remoteMessage) => {
    const { showSnackbar } = useUIStore.getState();

    // Show in-app notification using the snackbar
    showSnackbar({
      message: remoteMessage.notification?.body ?? 'New notification',
      variant: 'default',
      duration: 4000,
    });
  });

  return unsubscribe;
}, []);
```

---

## 102. Background Notification Handler

When the app is in the background or closed, notifications are shown in the system tray automatically. You just need to handle what happens when the user taps one.

```typescript
// Must be outside any component — module level
messaging().setBackgroundMessageHandler(async (remoteMessage) => {
  // This runs in a headless JS environment
  // Only perform simple tasks here (no navigation, no UI updates)
  console.log('Background message:', remoteMessage);
});
```

---

## 103. Notification Tap — Deep Link Navigation

When a user taps a notification, open the relevant screen.

```typescript
// In app/_layout.tsx
import messaging from '@react-native-firebase/messaging';

useEffect(() => {
  // App opened from notification (was in background)
  messaging().onNotificationOpenedApp((remoteMessage) => {
    handleNotificationTap(remoteMessage);
  });

  // App opened from notification (was completely closed)
  messaging()
    .getInitialNotification()
    .then((remoteMessage) => {
      if (remoteMessage) handleNotificationTap(remoteMessage);
    });
}, []);

function handleNotificationTap(remoteMessage: FirebaseMessagingTypes.RemoteMessage) {
  const taskId = remoteMessage.data?.taskId as string | undefined;
  if (taskId) {
    router.push(ROUTES.TASK_DETAIL(taskId));
  }
}
```

**Send notifications with data payload from server:**
```json
{
  "notification": { "title": "Task due", "body": "Fix login bug is due today" },
  "data": { "taskId": "abc123", "type": "task_due" }
}
```

---

## 104. Local Notifications & Scheduling

```bash
npx expo install expo-notifications
```

```typescript
// src/services/notifications/local.ts
import * as Notifications from 'expo-notifications';

Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldShowBanner: true,
    shouldPlaySound: false,
    shouldSetBadge: false,
  }),
});

export async function scheduleTaskReminder(
  taskId: string,
  taskTitle: string,
  dueDate: Date,
): Promise<string> {
  const reminderDate = new Date(dueDate);
  reminderDate.setHours(reminderDate.getHours() - 1); // 1 hour before due

  const notificationId = await Notifications.scheduleNotificationAsync({
    content: {
      title: 'Task Due Soon',
      body: `"${taskTitle}" is due in 1 hour`,
      data: { taskId },
    },
    trigger: {
      type: Notifications.SchedulableTriggerInputTypes.DATE,
      date: reminderDate,
    },
  });

  return notificationId; // save this to cancel later
}
```

---

## 105. Cancel Scheduled Notifications

```typescript
export async function cancelTaskReminder(notificationId: string): Promise<void> {
  await Notifications.cancelScheduledNotificationAsync(notificationId);
}

export async function cancelAllReminders(): Promise<void> {
  await Notifications.cancelAllScheduledNotificationsAsync();
}
```

**When to cancel:**
- Task completed → `cancelTaskReminder(task.notificationId)`
- Task deleted → `cancelTaskReminder(task.notificationId)`
- User disables reminders → `cancelAllReminders()`

Store `notificationId` in the task record in Supabase so you can cancel it even after app restarts.

---

## 106. Reanimated v3 Fundamentals

Reanimated runs animations on the **UI thread** (the native thread that draws pixels). Normal JavaScript animations run on the **JS thread** — if your app is busy loading data, the animation stutters. Reanimated animations are smooth regardless.

**Core API:**

```typescript
import Animated, {
  useSharedValue,    // mutable value that lives on UI thread
  useAnimatedStyle,  // derives CSS from shared values (runs on UI thread)
  withSpring,        // spring physics animation
  withTiming,        // linear/eased animation
  withSequence,      // run animations in sequence
  withRepeat,        // repeat an animation
  runOnJS,           // call a JS function from UI thread
  runOnUI,           // call a UI thread function from JS
} from 'react-native-reanimated';

function FadeInView({ children }) {
  const opacity = useSharedValue(0); // starts at 0 (invisible)

  useEffect(() => {
    opacity.value = withTiming(1, { duration: 400 }); // animate to 1 (visible)
  }, []);

  const animatedStyle = useAnimatedStyle(() => ({
    opacity: opacity.value,
  }));

  // Use Animated.View instead of View — it accepts Reanimated's animated styles
  return (
    <Animated.View style={animatedStyle}>
      {children}
    </Animated.View>
  );
}
```

**`useSharedValue`** — creates a value that lives on the UI thread. Mutating it doesn't trigger React re-renders.

**`useAnimatedStyle`** — derives a style object from shared values. This runs on the UI thread. Must not access regular JavaScript variables or call JS functions directly.

---

## 107. Worklets — Running on UI Thread

Functions that run on the UI thread are called **worklets**. They must be marked with the `'worklet'` directive.

```typescript
// ✅ Worklet — runs on UI thread
function clamp(value: number, min: number, max: number): number {
  'worklet';
  return Math.max(min, Math.min(value, max));
}

// Use in animated style
const animatedStyle = useAnimatedStyle(() => ({
  transform: [{ translateX: clamp(translateX.value, -100, 0) }],
});

// ❌ Can't call regular JS from UI thread
const animatedStyle = useAnimatedStyle(() => {
  console.log(translateX.value); // ERROR — console.log is not a worklet
  return { transform: [{ translateX: translateX.value }] };
});

// ✅ Use runOnJS to call JS from UI thread
const animatedStyle = useAnimatedStyle(() => {
  if (translateX.value < -80) {
    runOnJS(onDelete)(); // calls onDelete on JS thread when threshold crossed
  }
  return { transform: [{ translateX: translateX.value }] };
});
```

**What can't be used in worklets:**
- `console.log` (JS-only)
- State setters (`setCount`)
- React hooks (`useState`, `useRef`)
- Any third-party library not marked as worklet

**`runOnJS`** bridges from UI thread back to JS. Use it to trigger React state updates, navigation, or other JS operations in response to gesture/animation events.

---

## 108. Swipe-to-Delete with Gesture Handler

```bash
npx expo install react-native-gesture-handler
```

```tsx
import { GestureDetector, Gesture } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  runOnJS,
} from 'react-native-reanimated';

interface SwipeableTaskCardProps {
  task: Task;
  onDelete: (id: string) => void;
}

const DELETE_THRESHOLD = -100;

export function SwipeableTaskCard({ task, onDelete }: SwipeableTaskCardProps) {
  const translateX = useSharedValue(0);

  const panGesture = Gesture.Pan()
    .activeOffsetX([-10, 10])     // start tracking after 10px horizontal movement
    .failOffsetY([-5, 5])          // cancel if vertical movement detected
    .onUpdate((event) => {
      // Only allow left swipe
      translateX.value = Math.min(0, event.translationX);
    })
    .onEnd(() => {
      if (translateX.value < DELETE_THRESHOLD) {
        // Crossed threshold — trigger delete
        runOnJS(onDelete)(task.id);
      }
      // Snap back
      translateX.value = withSpring(0);
    });

  const cardStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }],
  }));

  const deleteIndicatorStyle = useAnimatedStyle(() => ({
    opacity: Math.min(1, Math.abs(translateX.value) / Math.abs(DELETE_THRESHOLD)),
  }));

  return (
    <View className="relative overflow-hidden rounded-lg">
      {/* Delete indicator behind the card */}
      <Animated.View
        style={deleteIndicatorStyle}
        className="absolute right-4 top-0 bottom-0 items-center justify-center"
      >
        <Trash2 size={24} color="white" />
      </Animated.View>

      <GestureDetector gesture={panGesture}>
        <Animated.View style={cardStyle}>
          <TaskCard task={task} />
        </Animated.View>
      </GestureDetector>
    </View>
  );
}
```

**Wrap app in `GestureHandlerRootView`** in `app/_layout.tsx`:
```tsx
import { GestureHandlerRootView } from 'react-native-gesture-handler';

// Wrap SafeAreaProvider:
<GestureHandlerRootView style={{ flex: 1 }}>
  <SafeAreaProvider>
    {/* ... */}
  </SafeAreaProvider>
</GestureHandlerRootView>
```

---

## 109. List Item Enter/Exit Animation

```tsx
import Animated, { FadeInDown, FadeOutLeft, Layout } from 'react-native-reanimated';

// Animated wrapper for TaskCard
export function AnimatedTaskCard({ task, index }: { task: Task; index: number }) {
  return (
    <Animated.View
      entering={FadeInDown.delay(index * 50).springify()} // stagger based on index
      exiting={FadeOutLeft}
      layout={Layout.springify()}                          // smooth layout shifts
    >
      <TaskCard task={task} />
    </Animated.View>
  );
}
```

**`entering`** — animation when the component mounts.

**`exiting`** — animation when the component unmounts (e.g., after delete). The component holds position while animating out before being removed.

**`layout`** — when other items shift position (e.g., an item above is removed), this animates the shift smoothly. Without it, items jump to their new positions instantly.

**`delay(index * 50)`** — stagger. First item: 0ms delay, second: 50ms, third: 100ms. Creates a cascade effect when the list initially appears.

---

## 110. Button Press Spring Animation

Micro-interaction — button scales down slightly when pressed, springs back on release.

```tsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
} from 'react-native-reanimated';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

export function SpringButton({ onPress, children, ...props }) {
  const scale = useSharedValue(1);

  const tapGesture = Gesture.Tap()
    .onBegin(() => {
      scale.value = withSpring(0.95); // press in
    })
    .onFinalize(() => {
      scale.value = withSpring(1);    // release
      runOnJS(onPress)();
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  return (
    <GestureDetector gesture={tapGesture}>
      <Animated.View style={animatedStyle} {...props}>
        {children}
      </Animated.View>
    </GestureDetector>
  );
}
```

**`withSpring`** — uses physics-based spring animation. The button snaps back with a natural feel, like pressing a physical button. No `duration` needed — spring physics determines timing.

---

## 111. Accessibility Audit

Run through the entire app with screen readers before every release.

**iOS VoiceOver:**
- Settings → Accessibility → VoiceOver → On
- Swipe right to move to next element
- Double-tap to activate
- Triple-click side button to toggle

**Android TalkBack:**
- Settings → Accessibility → TalkBack → On
- Swipe right to move to next element
- Double-tap to activate

**What to verify for each interactive element:**

| Prop | Purpose | Example |
|---|---|---|
| `accessibilityRole` | What is this? | `"button"`, `"link"`, `"header"`, `"image"` |
| `accessibilityLabel` | What does it say? | `"Delete task: Fix login bug"` |
| `accessibilityHint` | What happens? | `"Navigates to task details"` |
| `accessibilityState` | Current state? | `{ disabled: true }`, `{ selected: true }` |

**Common issues found in audits:**

```tsx
// ❌ Icon button with no label
<Pressable onPress={handleDelete}>
  <Trash2 />
</Pressable>

// ✅ Icon button with descriptive label
<Pressable
  onPress={handleDelete}
  accessibilityRole="button"
  accessibilityLabel={`Delete task: ${task.title}`}
  accessibilityHint="Permanently removes this task"
>
  <Trash2 />
</Pressable>
```

```tsx
// ❌ Image with no label
<Image source={url} />

// ✅ Image with description
<Image source={url} accessibilityLabel="Profile photo" accessibilityRole="image" />

// Or if decorative:
<Image source={decorativeImage} accessibilityElementsHidden importantForAccessibility="no" />
```

**Dynamic font sizes:** Test with largest accessibility font size (iOS: Settings → Accessibility → Display & Text Size → Larger Text). Text should remain readable. No truncation that hides important content.

**WCAG AA contrast:** All text must have a 4.5:1 contrast ratio against its background. Verify using a contrast checker against both light and dark theme colors.

---

## 112. Multi-Environment Setup

Three environments: `development`, `staging`, `production`. Each has separate backends, bundle IDs, and app names so multiple versions can be installed side by side on one device.

`.env.development`:
```
EXPO_PUBLIC_API_URL=https://dev-api.taskflow.app
EXPO_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
EXPO_PUBLIC_SUPABASE_ANON_KEY=eyJxxx
EXPO_PUBLIC_ENV=development
```

`.env.staging`:
```
EXPO_PUBLIC_API_URL=https://staging-api.taskflow.app
EXPO_PUBLIC_ENV=staging
```

`.env.production`:
```
EXPO_PUBLIC_API_URL=https://api.taskflow.app
EXPO_PUBLIC_ENV=production
```

**`EXPO_PUBLIC_` prefix:** Required for variables to be bundled into the app. Variables without this prefix are never sent to the device. Never prefix secrets (Firebase admin keys, payment keys) — they'd be visible to anyone who decompiles the APK.

---

## 113. app.config.ts — Dynamic Config

Convert `app.json` to `app.config.ts` for environment-specific config:

```typescript
import type { ConfigContext, ExpoConfig } from 'expo/config';

const ENV = process.env.EXPO_PUBLIC_ENV ?? 'development';

const appNames = {
  development: 'TaskFlow Dev',
  staging:     'TaskFlow Staging',
  production:  'TaskFlow',
};

const bundleIds = {
  development: 'com.taskflow.app.dev',
  staging:     'com.taskflow.app.staging',
  production:  'com.taskflow.app',
};

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  name: appNames[ENV as keyof typeof appNames] ?? 'TaskFlow',
  scheme: 'taskflow',
  ios: {
    bundleIdentifier: bundleIds[ENV as keyof typeof bundleIds],
    supportsTablet: true,
  },
  android: {
    package: bundleIds[ENV as keyof typeof bundleIds],
  },
  plugins: [
    'expo-router',
    '@react-native-firebase/app',
    '@react-native-firebase/auth',
    '@react-native-firebase/messaging',
    ['expo-image-picker', {
      photosPermission: 'TaskFlow needs access to your photos to attach images to tasks.',
      cameraPermission: 'TaskFlow needs camera access to capture images for tasks.',
    }],
  ],
  experiments: { typedRoutes: true },
});
```

**Three apps on one device:** Development, staging, and production all have different bundle IDs — iOS and Android treat them as separate apps. You can have all three installed simultaneously for testing.

---

## 114. Typed Environment Variables

```typescript
// src/lib/env.ts
function requireEnv(key: string): string {
  const value = process.env[key];
  if (!value) throw new Error(`Missing required environment variable: ${key}`);
  return value;
}

export const env = {
  apiUrl:           requireEnv('EXPO_PUBLIC_API_URL'),
  supabaseUrl:      requireEnv('EXPO_PUBLIC_SUPABASE_URL'),
  supabaseAnonKey:  requireEnv('EXPO_PUBLIC_SUPABASE_ANON_KEY'),
  environment:      (process.env.EXPO_PUBLIC_ENV ?? 'development') as
                    'development' | 'staging' | 'production',

  // Derived helpers
  isDev:  process.env.EXPO_PUBLIC_ENV === 'development',
  isProd: process.env.EXPO_PUBLIC_ENV === 'production',
} as const;
```

`requireEnv` throws at startup if a variable is missing — better than a cryptic runtime error deep in the app when the variable is first used.

---

## 115. Sentry Integration

```bash
npx expo install @sentry/react-native
```

`src/lib/sentry.ts`:
```typescript
import * as Sentry from '@sentry/react-native';
import { env } from './env';

export function initSentry() {
  Sentry.init({
    dsn: process.env.EXPO_PUBLIC_SENTRY_DSN,
    environment: env.environment,
    enabled: env.isProd,         // only in production
    tracesSampleRate: 0.2,       // 20% of transactions traced
    beforeSend: (event) => {
      // Remove PII from error reports
      if (event.user) {
        delete event.user.email;
        delete event.user.ip_address;
      }
      return event;
    },
  });
}

// Add user context after sign-in
export function setSentryUser(userId: string) {
  Sentry.setUser({ id: userId });
}

// Clear on sign-out
export function clearSentryUser() {
  Sentry.setUser(null);
}
```

**Wrap root in Sentry:**
```tsx
// app/_layout.tsx
import { initSentry } from '@/lib/sentry';
initSentry();

// Wrap the return value:
export default Sentry.wrap(RootLayout);
```

**Source maps** — upload during EAS Build so stack traces show your TypeScript code, not minified JS:
```json
// eas.json
{
  "build": {
    "production": {
      "env": {
        "SENTRY_AUTH_TOKEN": "your-token"
      }
    }
  }
}
```

---

## 116. Error Boundaries

Error boundaries catch JavaScript errors in the component tree and show a fallback UI instead of crashing the app.

`src/components/common/ErrorBoundary.tsx`:
```tsx
import React from 'react';
import { View } from 'react-native';
import { Text, Button } from '@/components/ui';

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends React.Component<
  { children: React.ReactNode; fallback?: React.ReactNode },
  ErrorBoundaryState
> {
  constructor(props: any) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    // Log to Sentry
    Sentry.captureException(error, { extra: { componentStack: info.componentStack } });
  }

  reset() {
    this.setState({ hasError: false, error: null });
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? (
        <View className="flex-1 items-center justify-center gap-4 px-8">
          <Text variant="h3">Something went wrong</Text>
          <Text color="muted" className="text-center">
            An unexpected error occurred. Our team has been notified.
          </Text>
          <Button label="Try Again" onPress={() => this.reset()} />
        </View>
      );
    }
    return this.props.children;
  }
}
```

**Wrap each major screen group:**
```tsx
// In app/(app)/_layout.tsx
<ErrorBoundary>
  <Stack>{/* screens */}</Stack>
</ErrorBoundary>
```

One error boundary around the entire app means a bug on one screen crashes the whole app. Per-screen boundaries allow other screens to continue working.

---

## 117. Analytics with PostHog

```bash
npm install posthog-react-native
```

`src/lib/analytics.ts`:
```typescript
import PostHog from 'posthog-react-native';
import { env } from './env';

export const analytics = new PostHog(
  process.env.EXPO_PUBLIC_POSTHOG_KEY ?? '',
  {
    host: 'https://app.posthog.com',
    disabled: !env.isProd,   // no analytics in dev
  },
);

export function identifyUser(userId: string, traits?: Record<string, unknown>) {
  analytics.identify(userId, traits);
}

export function trackEvent(event: string, properties?: Record<string, unknown>) {
  analytics.capture(event, properties);
}

export function resetAnalytics() {
  analytics.reset();
}
```

**Events to track (minimum):**
```typescript
// Auth
trackEvent('user_signed_up', { method: 'email' });
trackEvent('user_signed_in');
trackEvent('user_signed_out');

// Core feature
trackEvent('task_created', { priority: task.priority });
trackEvent('task_completed', { timeToComplete: daysSinceCreation });
trackEvent('task_deleted');
trackEvent('search_performed', { resultsCount: data.count });
```

---

## 118. Jest & React Native Testing Library

```bash
npm install --save-dev jest jest-expo @testing-library/react-native @testing-library/jest-native
```

`jest.config.js`:
```javascript
module.exports = {
  preset: 'jest-expo',
  setupFilesAfterFramework: ['@testing-library/jest-native/extend-expect'],
  transformIgnorePatterns: [
    'node_modules/(?!(jest-)?react-native|@react-native|expo|@expo|@unimodules|unimodules|sentry-expo|native-base|@sentry)',
  ],
};
```

**Testing a component:**
```typescript
import { render, fireEvent, screen } from '@testing-library/react-native';
import { Button } from '@/components/ui';

describe('Button', () => {
  it('renders the label', () => {
    render(<Button label="Save" onPress={() => {}} />);
    expect(screen.getByText('Save')).toBeTruthy();
  });

  it('shows loading indicator when loading', () => {
    render(<Button label="Save" loading onPress={() => {}} />);
    expect(screen.queryByText('Save')).toBeNull();
    expect(screen.getByRole('progressbar')).toBeTruthy();
  });

  it('calls onPress when tapped', () => {
    const onPress = jest.fn();
    render(<Button label="Save" onPress={onPress} />);
    fireEvent.press(screen.getByRole('button'));
    expect(onPress).toHaveBeenCalledTimes(1);
  });

  it('does not call onPress when disabled', () => {
    const onPress = jest.fn();
    render(<Button label="Save" disabled onPress={onPress} />);
    fireEvent.press(screen.getByRole('button'));
    expect(onPress).not.toHaveBeenCalled();
  });
});
```

---

## 119. Maestro E2E Testing

Maestro is a mobile UI testing tool that drives real devices/simulators via YAML flows.

```bash
curl -Ls "https://get.maestro.mobile.dev" | bash
```

`e2e/flows/login.yaml`:
```yaml
appId: com.taskflow.app.dev
---
- launchApp
- assertVisible: "Welcome back"
- tapOn:
    text: "Email"
- inputText: "test@example.com"
- tapOn:
    text: "Password"
- inputText: "Password123"
- tapOn:
    text: "Sign In"
- assertVisible: "Tasks"
- assertVisible: "Projects"
```

`e2e/flows/create-task.yaml`:
```yaml
appId: com.taskflow.app.dev
---
- launchApp
- tapOn: "New Task (Modal)"
- assertVisible: "New Task"
- tapOn:
    text: "Title"
- inputText: "Test task from Maestro"
- tapOn: "Save"
- assertVisible: "Test task from Maestro"
```

**Run tests:**
```bash
maestro test e2e/flows/login.yaml
maestro test e2e/flows/     # run all flows in directory
```

---

## 120. Performance Audit

**Tools:**
- React DevTools Profiler — see which components re-render and why
- Flashlight — measures JS FPS and UI FPS on real Android devices
- Expo's performance monitor overlay — enable with `Cmd+D → Performance Monitor`

**What to measure:**
1. Cold start time (launch → first interactive frame): target < 3 seconds
2. FlatList/FlashList scroll FPS: target 60 FPS with 500 items
3. JS FPS during navigation transitions: target 60 FPS

**Common performance issues and fixes:**

| Issue | Symptom | Fix |
|---|---|---|
| Unnecessary re-renders | Component re-renders without prop/state change | Add `memo()`, fix selector |
| Large bundle | Slow cold start | Check bundle with `npx expo export`, lazy load screens |
| Missing `keyExtractor` | Jumpy list scroll | Add stable, unique key |
| Missing `estimatedItemSize` | FlashList scroll position wrong | Measure item, provide accurate estimate |
| `useCallback` missing | `renderItem` recreated on each render | Wrap in `useCallback` |
| Image not cached | Images flash/reload | Switch to `expo-image` |

---

## 121. EAS Build Profiles

`eas.json`:
```json
{
  "cli": { "version": ">= 10.0.0" },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "ios": { "simulator": true },
      "env": { "EXPO_PUBLIC_ENV": "development" }
    },
    "preview": {
      "distribution": "internal",
      "android": { "buildType": "apk" },
      "env": { "EXPO_PUBLIC_ENV": "staging" }
    },
    "production": {
      "autoIncrement": true,
      "env": { "EXPO_PUBLIC_ENV": "production" }
    }
  },
  "submit": {
    "production": {
      "ios": {
        "appleId": "your@email.com",
        "ascAppId": "your-app-store-connect-app-id"
      },
      "android": {
        "serviceAccountKeyPath": "./play-store-service-account.json",
        "track": "internal"
      }
    }
  }
}
```

**Build commands:**
```bash
# Development build (with dev client, fast iteration)
eas build --profile development --platform all

# Internal testing build (shares as APK / ad-hoc)
eas build --profile preview --platform all

# App store production build
eas build --profile production --platform all
```

**Three profiles — three purposes:**
- `development` — replaces Expo Go. Has dev menu, hot reload, all native modules.
- `preview` — shareable test build. Has production config but distributes internally. For QA.
- `production` — store submission. Optimized, no dev tools.

---

## 122. EAS Update — OTA

OTA (Over-the-Air) updates push JavaScript changes to users without App Store review.

```bash
# Push update to production users
eas update --branch production --message "Fix task sorting bug"

# Push update to staging
eas update --branch staging --message "Testing new filter UI"
```

**What CAN be updated OTA:**
- JavaScript logic changes
- UI layout changes (NativeWind classes)
- Bug fixes in TypeScript/JavaScript
- New screens (if they use only existing native modules)

**What CANNOT be updated OTA:**
- New native modules (new Expo plugins)
- Changes to `app.json`/`app.config.ts`
- New Expo SDK version
- Permission changes

**Update channels** — each EAS Build subscribes to a channel. Production builds listen to `production`, staging builds to `staging`. Push to wrong channel = wrong users get it.

**Rollback:**
```bash
eas update rollback --branch production
```

---

## 123. Store Submission Prep

### iOS — App Store

**Required assets:**
- App icon: 1024×1024 PNG, no alpha channel, no rounded corners (Apple adds them)
- Screenshots: 6.7" (iPhone 15 Pro Max), 6.5" (iPhone 11 Pro Max), iPad Pro 12.9" 3rd gen
- Privacy policy URL (must be publicly accessible)

**Required before submission:**
- Apple Developer Account ($99/year)
- Privacy Manifest file (`PrivacyInfo.xcprivacy`) — required from iOS 17
- App Store Connect listing (name, description, keywords, category)
- Age rating (complete the IARC questionnaire)

**Submit with EAS:**
```bash
eas submit --platform ios --profile production
```

### Android — Google Play

**Required assets:**
- Feature graphic: 1024×500 PNG
- Screenshots: at least 2 phone screenshots
- App icon: 512×512 PNG

**Required before submission:**
- Google Play Developer Account ($25 one-time)
- Data Safety form (declare what data you collect)
- Content rating (IARC questionnaire)
- Target SDK 35 (2026 requirement)

**Submit with EAS:**
```bash
eas submit --platform android --profile production
```

### Final Pre-submission Checklist

- [ ] App tested on real physical devices (not just simulator)
- [ ] All crashes fixed — target 99.5%+ crash-free sessions
- [ ] TestFlight beta test run (iOS) / Internal testing track test (Android)
- [ ] All required permissions explained in app store description
- [ ] Privacy policy accessible and current
- [ ] Version number and build number incremented
- [ ] Sentry configured and receiving real crash reports
- [ ] Analytics verified in PostHog dashboard
- [ ] Deep links tested (notification taps open correct screens)
- [ ] Offline mode tested (airplane mode, then reconnect)
- [ ] Dark mode tested on all major screens
- [ ] Accessibility tested with VoiceOver / TalkBack
