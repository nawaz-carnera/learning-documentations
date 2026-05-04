# Module 2 — Expo Setup & Development Tooling

> We will be using **Expo** throughout this course. This module explains why, how to set it up, and the tools you'll use daily.

---

## Table of Contents

1. [Expo vs Bare React Native CLI](#1-expo-vs-bare-react-native-cli)
   - 1.1 [When to Choose Which](#11-when-to-choose-which)
   - 1.2 [Expo SDK Versions](#12-expo-sdk-versions)
   - 1.3 [Expo Dev Client vs Expo Go](#13-expo-dev-client-vs-expo-go)

2. [Project Initialization](#2-project-initialization)
   - 2.1 [create-expo-app](#21-create-expo-app)
   - 2.2 [TypeScript Setup](#22-typescript-setup)
   - 2.3 [ESLint and Prettier Configuration](#23-eslint-and-prettier-configuration)
   - 2.4 [Folder Structure](#24-folder-structure)

3. [Development Tools](#3-development-tools)
   - 3.1 [Metro Bundler Basics](#31-metro-bundler-basics)
   - 3.2 [React Native DevTools](#32-react-native-devtools)
   - 3.3 [Flipper / React DevTools Profiler](#33-flipper--react-devtools-profiler)
   - 3.4 [Expo CLI Commands](#34-expo-cli-commands)
   - 3.5 [Simulator (iOS) and Emulator (Android) Setup](#35-simulator-ios-and-emulator-android-setup)

---

## 1. Expo vs Bare React Native CLI

### 1.1 When to Choose Which

Think of **Expo** as "Create React App for React Native" — it handles the native build complexity so you focus on JS/TS code. **Bare React Native CLI** gives you full native control but requires Xcode/Android Studio knowledge.

| | Expo (Managed) | Bare React Native CLI |
|---|---|---|
| Setup time | Minutes | Hours |
| Native code access | Via Expo SDK / plugins | Full access |
| Custom native modules | Limited (use config plugins) | Unrestricted |
| OTA updates | Built-in (EAS Update) | Manual setup |
| Build service | EAS Build (cloud) | Local machine |
| Good for | Most apps, startups, learning | Deep native integrations (e.g., Bluetooth, custom SDKs) |
| Escape hatch | `expo prebuild` → ejects to bare | N/A |

**Choose Expo when:**
- Building a new app without specialized native requirements
- You want fast iteration and OTA updates
- Your team is JS/TS focused, not native developers

**Choose Bare CLI when:**
- Integrating a third-party native SDK with no Expo plugin
- The app requires deep OS-level customization
- You already have a native iOS/Android codebase to bridge

> You can always **eject** from Expo Managed to Bare using `npx expo prebuild`. The reverse is not straightforward, so start Managed and eject only if needed.

---

### 1.2 Expo SDK Versions

Expo releases a new **SDK version** roughly every quarter, tied to a specific React Native version.

```
Expo SDK 52  →  React Native 0.76
Expo SDK 51  →  React Native 0.74
Expo SDK 50  →  React Native 0.73
```

**Key points:**

- Each SDK version has a **support window** (~1 year). After that, Expo Go drops support for older SDKs.
- You upgrade the SDK by bumping the version in `package.json` and running the upgrade command — it is not automatic.
- Always check the [Expo changelog](https://expo.dev/changelog) before upgrading — some SDK bumps include breaking changes.

**Upgrading SDK:**
```bash
npx expo install expo@^52
npx expo install --fix        # fixes peer dependency versions
```

**Check current SDK version:**
```bash
npx expo --version
# or look at package.json → "expo": "~52.0.0"
```

---

### 1.3 Expo Dev Client vs Expo Go

Both let you run your app on a real device during development — but they work differently.

| | Expo Go | Expo Dev Client |
|---|---|---|
| What it is | Pre-built app from the App Store | A custom dev build you compile yourself |
| Setup | Zero — just install & scan QR | Requires one `eas build` or local build |
| Supports custom native modules | No | Yes |
| Supports all Expo SDK APIs | Only bundled ones | All + your own |
| Good for | Quick prototyping, learning | Real production development |

**Expo Go workflow:**
```
Install "Expo Go" from App Store / Play Store
  → run `npx expo start`
  → scan QR code
  → app loads instantly
```

**Expo Dev Client workflow:**
```
Add your custom native module
  → run `npx expo run:ios` (or `eas build --profile development`)
  → install the generated .ipa/.apk on your device once
  → from then on, use the same QR scan workflow
```

> **For this course:** Expo Go is sufficient to start. You'll switch to Dev Client only when you add a library that requires native code not bundled in Expo Go (e.g., `react-native-mmkv`, `react-native-vision-camera`).

---

## 2. Project Initialization

### 2.1 create-expo-app

The official way to scaffold a new Expo project.

```bash
# Create a new project (TypeScript template, Expo Router)
npx create-expo-app@latest MyApp

# With a specific template
npx create-expo-app@latest MyApp --template blank-typescript
```

**Available templates:**

| Template | Description |
|---|---|
| `default` | Expo Router + TypeScript (recommended) |
| `blank` | Minimal JS setup |
| `blank-typescript` | Minimal TypeScript setup |
| `tabs` | Expo Router with bottom tabs |

```bash
cd MyApp
npx expo start        # starts the dev server
```

**What gets created:**
```
MyApp/
├── app/              # screens (Expo Router file-based routing)
├── assets/           # images, fonts
├── components/       # reusable components
├── constants/        # theme colors, sizes
├── hooks/            # custom hooks
├── app.json          # Expo config (app name, icon, splash)
├── package.json
├── tsconfig.json
└── expo-env.d.ts     # Expo TypeScript ambient declarations
```

---

### 2.2 TypeScript Setup

`create-expo-app` with the default template sets up TypeScript automatically. Here's what to know:

**`tsconfig.json` (Expo default):**
```json
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "paths": {
      "@/*": ["./*"]    // path alias: import from '@/components/Button'
    }
  }
}
```

**`expo/tsconfig.base` already includes:**
- `"jsx": "react-native"`
- `"moduleResolution": "bundler"`
- `"allowJs": true`
- `"noEmit": true` (Metro does the actual transpilation)

**Path aliases** — avoids `../../..` imports:
```tsx
// Without alias
import Button from '../../../components/Button';

// With @/* alias
import Button from '@/components/Button';
```

To enable path aliases, ensure your `tsconfig.json` has the `paths` entry above, and install the babel plugin:
```bash
npx expo install babel-plugin-module-resolver
```

```js
// babel.config.js
module.exports = {
  presets: ['babel-preset-expo'],
  plugins: [
    ['module-resolver', {
      alias: { '@': './' },
    }],
  ],
};
```

---

### 2.3 ESLint and Prettier Configuration

**Install dependencies:**
```bash
npx expo install -- --save-dev eslint prettier eslint-config-expo eslint-config-prettier eslint-plugin-prettier
```

**`.eslintrc.js`:**
```js
module.exports = {
  extends: [
    'expo',                  // Expo's recommended config (includes react, react-native, typescript rules)
    'prettier',              // disables ESLint rules that conflict with Prettier
  ],
  plugins: ['prettier'],
  rules: {
    'prettier/prettier': 'error',
    'no-console': 'warn',    // warn on console.log left in code
  },
};
```

**`.prettierrc`:**
```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "bracketSameLine": false
}
```

**`.prettierignore`:**
```
node_modules/
.expo/
dist/
android/
ios/
```

**Add scripts to `package.json`:**
```json
{
  "scripts": {
    "lint": "eslint . --ext .ts,.tsx",
    "lint:fix": "eslint . --ext .ts,.tsx --fix",
    "format": "prettier --write \"**/*.{ts,tsx,json}\""
  }
}
```

**VSCode settings (`.vscode/settings.json`):**
```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  }
}
```

---

### 2.4 Folder Structure

A clean folder structure for a real Expo project:

```
MyApp/
├── app/                        # Expo Router — every file = a screen/route
│   ├── (tabs)/
│   │   ├── index.tsx           # Home tab
│   │   ├── profile.tsx         # Profile tab
│   │   └── _layout.tsx         # Tab navigator config
│   ├── _layout.tsx             # Root layout (providers, global styles)
│   └── +not-found.tsx          # 404 screen
│
├── src/                        # All non-route source code
│   ├── components/             # Reusable UI components
│   │   ├── common/             # Button, Card, Input, etc.
│   │   └── screens/            # Screen-specific components
│   ├── hooks/                  # Custom hooks
│   ├── store/                  # State management (Zustand / Redux)
│   ├── services/               # API calls, third-party service wrappers
│   ├── utils/                  # Pure helper functions
│   ├── constants/              # Colors, spacing, typography, config
│   └── types/                  # Shared TypeScript interfaces/types
│
├── assets/
│   ├── images/
│   └── fonts/
│
├── .expo/                      # Auto-generated, add to .gitignore
├── app.json                    # Expo app config
├── babel.config.js
├── tsconfig.json
├── .eslintrc.js
├── .prettierrc
└── package.json
```

> Keep `app/` only for routing. Business logic, components, and services go inside `src/`. This keeps your route files thin and readable.

---

## 3. Development Tools

### 3.1 Metro Bundler Basics

**Metro** is React Native's JavaScript bundler (equivalent to Webpack/Vite for the web). It:
- Watches your source files for changes
- Bundles JS/TS into a single file the native runtime can execute
- Serves the bundle to the device/simulator over a local port (default: `8081`)

```bash
npx expo start         # starts Metro + Expo dev server
npx expo start --clear # clears Metro cache (fixes most "weird" build errors)
```

**Metro cache** — Metro caches transformed files to speed up rebuilds. When you install a new native library or change `babel.config.js`, always clear the cache:
```bash
npx expo start --clear
# or
npx react-native start --reset-cache
```

**How it works:**
```
Your .tsx files
      ↓
   Metro (transpile + bundle)
      ↓
   Bundle served on localhost:8081
      ↓
   Native runtime (JS engine: Hermes) executes the bundle
      ↓
   Native bridge renders UI components
```

**Hermes** — the default JS engine in React Native (replaces JavaScriptCore). It pre-compiles JS to bytecode, giving faster startup and lower memory usage. Enabled by default in all new Expo projects.

---

### 3.2 React Native DevTools

The built-in debugger launched from the Expo dev server. As of React Native 0.73+, it replaced the old Chrome debugger with a purpose-built tool.

**How to open:**
```
Press j in the Metro terminal
# or
Press m → "Open DevTools" in Expo Go menu
# or
Shake device → "Open Debugger"
```

**What you get:**

| Panel | Use |
|---|---|
| Console | `console.log`, errors, warnings |
| Sources | Breakpoints, step-through debugging in your TS source |
| Network | Inspect HTTP requests (fetch, axios) |
| Components | React component tree inspection (like React DevTools) |
| Profiler | Flame graph of component renders |

**Setting a breakpoint:**
```tsx
// Option 1: in DevTools Sources panel — click line number
// Option 2: in code
debugger; // pauses execution in DevTools when hit
```

---

### 3.3 Flipper / React DevTools Profiler

#### Flipper (mostly legacy now)

Flipper was the standard native debugger for React Native but is **no longer recommended** for new projects as of RN 0.73+. React Native DevTools (above) has replaced most of its use cases.

Still useful for:
- Inspecting SQLite / MMKV storage via plugins
- Native network inspection at the OS level
- Custom native debugging plugins

#### React DevTools (Standalone)

A standalone version of React DevTools that connects to your running app for component inspection and profiling.

```bash
# Install globally
npm install -g react-devtools

# Run it
react-devtools
```

Then shake your device → "Open React DevTools" (or it auto-connects).

**Profiler tab — identifying performance issues:**

```
Start profiling → interact with your app → stop profiling
→ Flame graph shows which components rendered and how long each took
→ "Ranked" tab sorts by render time — find the slowest components
```

**Reading the flame graph:**
- Each bar = one component render
- Wide bar = slow render (investigate with `React.memo`, `useMemo`, `useCallback`)
- Gray bar = component did not re-render in that commit (good)
- Yellow/Orange bar = re-rendered (check if it was necessary)

---

### 3.4 Expo CLI Commands

Essential commands you'll use daily:

```bash
# Start dev server
npx expo start

# Start with cache cleared
npx expo start --clear

# Run directly on iOS simulator
npx expo run:ios

# Run directly on Android emulator
npx expo run:android

# Install an Expo-compatible package (resolves correct version for your SDK)
npx expo install package-name
# Example:
npx expo install expo-camera expo-location

# Check for outdated/incompatible packages
npx expo install --check

# Fix incompatible package versions
npx expo install --fix

# Build for production (via EAS)
eas build --platform ios
eas build --platform android
eas build --platform all

# Preview build on device (EAS)
eas build --profile preview --platform android

# Submit to App Store / Play Store
eas submit

# OTA update (push JS changes without a new build)
eas update --branch production

# Upgrade Expo SDK
npx expo install expo@latest
npx expo install --fix

# Show project info
npx expo config
npx expo diagnostics      # prints environment info for bug reports

# Login to Expo account (needed for EAS)
npx expo login
npx expo whoami
```

> **Always use `npx expo install`** instead of `npm install` for Expo/RN packages. It picks the version compatible with your current Expo SDK — plain `npm install` can grab a version that breaks your build.

---

### 3.5 Simulator (iOS) and Emulator (Android) Setup

#### iOS Simulator (macOS only)

**Requirements:** Xcode (from Mac App Store)

```bash
# Install Xcode Command Line Tools
xcode-select --install

# Open Xcode → Settings → Platforms → download iOS Simulator runtime

# Run on simulator from Expo
npx expo start
# Press i  → opens iOS Simulator automatically
```

**Managing simulators:**
```bash
# List available simulators
xcrun simctl list devices

# Open Simulator app directly
open -a Simulator

# Install a .app build manually
xcrun simctl install booted path/to/MyApp.app
```

#### Android Emulator

**Requirements:** Android Studio

**Setup steps:**
1. Download Android Studio from developer.android.com
2. Open Android Studio → `More Actions` → `Virtual Device Manager`
3. Click `Create Device` → choose a phone (e.g., Pixel 8) → select a system image (API 34 recommended)
4. Click `Finish` → press ▶ to start the emulator

```bash
# Run on emulator from Expo
npx expo start
# Press a  → opens Android Emulator automatically

# List running emulators
adb devices

# Check if adb is installed
adb --version

# If 'a' doesn't work, ensure ANDROID_HOME is set:
export ANDROID_HOME=$HOME/Library/Android/sdk
export PATH=$PATH:$ANDROID_HOME/emulator:$ANDROID_HOME/platform-tools
# Add these to your ~/.zshrc
```

**Useful emulator shortcuts:**

| Shortcut | Action |
|---|---|
| `Cmd + Shift + H` (iOS) | Go to Home |
| `Cmd + Left/Right` (iOS) | Rotate |
| `Ctrl + M` (Android) | Open dev menu |
| `Cmd + R` (iOS sim) | Reload app |

#### Testing on a Real Device

**iOS (via Expo Go):**
1. Install Expo Go from App Store
2. Connect to the same WiFi as your machine
3. Run `npx expo start` → scan QR with Camera app

**Android (via Expo Go):**
1. Install Expo Go from Play Store
2. Connect to the same WiFi
3. Run `npx expo start` → scan QR inside Expo Go app

**Android USB debugging:**
```bash
# Enable Developer Options on device: Settings → About Phone → tap Build Number 7 times
# Enable USB Debugging in Developer Options
adb devices   # should show your device
npx expo run:android  # deploys directly to device
```

---

*End of Module 2*
