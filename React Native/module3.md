# Module 3 — Styling in React Native

> React Native has no CSS. Styles are JavaScript objects, layout is Flexbox-only, and every value is a number. This module covers everything from basics to theming.

---

## Table of Contents

1. [StyleSheet Fundamentals](#1-stylesheet-fundamentals)
   - 1.1 [StyleSheet.create](#11-stylesheetcreate)
   - 1.2 [Inline vs StyleSheet Styles](#12-inline-vs-stylesheet-styles)
   - 1.3 [Style Composition and Arrays](#13-style-composition-and-arrays)
   - 1.4 [camelCase Properties](#14-camelcase-properties)

2. [Flexbox in React Native](#2-flexbox-in-react-native)
   - 2.1 [flexDirection (default column)](#21-flexdirection-default-column)
   - 2.2 [justifyContent](#22-justifycontent)
   - 2.3 [alignItems](#23-alignitems)
   - 2.4 [alignSelf](#24-alignself)
   - 2.5 [flex, flexGrow, flexShrink, flexBasis](#25-flex-flexgrow-flexshrink-flexbasis)
   - 2.6 [gap property](#26-gap-property)

3. [Responsive Design](#3-responsive-design)
   - 3.1 [Dimensions API](#31-dimensions-api)
   - 3.2 [useWindowDimensions Hook](#32-usewindowdimensions-hook)
   - 3.3 [PixelRatio](#33-pixelratio)
   - 3.4 [Percentage-Based Sizing](#34-percentage-based-sizing)
   - 3.5 [Aspect Ratio](#35-aspect-ratio)
   - 3.6 [Handling Tablets and Foldables](#36-handling-tablets-and-foldables)
   - 3.7 [Orientation Handling](#37-orientation-handling)

4. [Safe Areas & Notches](#4-safe-areas--notches)
   - 4.1 [SafeAreaView](#41-safeareaview)
   - 4.2 [react-native-safe-area-context](#42-react-native-safe-area-context)
   - 4.3 [useSafeAreaInsets](#43-usesafeareainsets)
   - 4.4 [StatusBar Component](#44-statusbar-component)

5. [Styling Libraries](#5-styling-libraries)
   - 5.1 [NativeWind v4 (Tailwind for RN)](#51-nativewind-v4-tailwind-for-rn)
   - 5.2 [Unistyles 3.0](#52-unistyles-30)
   - 5.3 [StyleSheet with Theme Context](#53-stylesheet-with-theme-context)

6. [Theming](#6-theming)
   - 6.1 [Light and Dark Mode](#61-light-and-dark-mode)
   - 6.2 [useColorScheme Hook](#62-usecolorscheme-hook)
   - 6.3 [Theme Provider Pattern](#63-theme-provider-pattern)
   - 6.4 [Dynamic Color Schemes](#64-dynamic-color-schemes)

7. [Platform-Specific Styling](#7-platform-specific-styling)
   - 7.1 [Platform.select for Styles](#71-platformselect-for-styles)
   - 7.2 [Shadow vs Elevation](#72-shadow-vs-elevation)
   - 7.3 [Font Handling Across Platforms](#73-font-handling-across-platforms)

---

## 1. StyleSheet Fundamentals

### 1.1 StyleSheet.create

`StyleSheet.create` validates your style objects at dev time and optimizes them for the bridge. It is the standard way to define styles in React Native.

```tsx
import { View, Text, StyleSheet } from 'react-native';

export default function Card() {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Hello</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    padding: 16,
    backgroundColor: '#fff',
    borderRadius: 8,
  },
  title: {
    fontSize: 18,
    fontWeight: 'bold',
    color: '#111',
  },
});
```

**Why use it over plain objects?**
- Validates property names and values in development (typos throw errors)
- Converts the style object to an ID, reducing serialization overhead across the JS–native bridge
- Groups styles away from JSX, keeping components readable

---

### 1.2 Inline vs StyleSheet Styles

```tsx
// Inline — fine for truly dynamic values
<View style={{ backgroundColor: isActive ? '#007AFF' : '#ccc', padding: 8 }}>

// StyleSheet — for static styles
<View style={styles.box}>
```

**When to use inline:**
- The style value depends on runtime state/props
- One-off dynamic values (e.g., `{ width: progress * 100 }`)

**When to use StyleSheet:**
- Everything else — static styles should never be inline

> Inline style objects are recreated on every render. `StyleSheet.create` registers styles once. For performance-sensitive lists, always use `StyleSheet`.

---

### 1.3 Style Composition and Arrays

Pass an **array of styles** to the `style` prop. React Native merges them left to right — later styles win on conflicts.

```tsx
// Merging styles
<View style={[styles.base, styles.rounded, { backgroundColor: 'red' }]} />

// Conditional styles
<Text style={[styles.text, isError && styles.errorText]}>
  Message
</Text>

// Props-based overrides — allow consumers to extend your component's style
function Badge({ style }: { style?: object }) {
  return <View style={[styles.badge, style]} />;
}
```

```tsx
const styles = StyleSheet.create({
  base: { padding: 12, backgroundColor: '#eee' },
  rounded: { borderRadius: 8 },
  text: { fontSize: 14, color: '#333' },
  errorText: { color: 'red', fontWeight: 'bold' },
  badge: { width: 24, height: 24, borderRadius: 12, backgroundColor: 'blue' },
});
```

**`StyleSheet.flatten`** — collapses an array into a single object (useful for passing to third-party components that don't accept arrays):

```tsx
const merged = StyleSheet.flatten([styles.base, styles.rounded]);
// → { padding: 12, backgroundColor: '#eee', borderRadius: 8 }
```

---

### 1.4 camelCase Properties

React Native uses camelCase for all CSS properties. There are no shorthand properties like `border`, `margin`, or `font`.

| CSS (Web) | React Native |
|---|---|
| `background-color` | `backgroundColor` |
| `font-size` | `fontSize` |
| `font-weight` | `fontWeight` |
| `border-radius` | `borderRadius` |
| `margin: 8 16` | `marginVertical: 8, marginHorizontal: 16` |
| `padding: 10px 20px 10px 20px` | `paddingTop/Right/Bottom/Left` |
| `border: 1px solid red` | `borderWidth: 1, borderStyle: 'solid', borderColor: 'red'` |
| `box-shadow` | `shadow*` (iOS) / `elevation` (Android) |

**Margin / Padding shorthand equivalents:**

```tsx
// Instead of margin: '8px 16px'
{ marginVertical: 8, marginHorizontal: 16 }

// Instead of padding: '10px 20px 5px 15px'
{ paddingTop: 10, paddingRight: 20, paddingBottom: 5, paddingLeft: 15 }

// Shorthand for all four sides
{ margin: 8 }      // all sides
{ padding: 16 }    // all sides
```

---

## 2. Flexbox in React Native

React Native uses Flexbox for all layout. Key difference from web: **`flexDirection` defaults to `'column'`**, not `'row'`.

### 2.1 flexDirection (default column)

Controls the **main axis** direction.

```tsx
// Default: column — children stack vertically
<View style={{ flexDirection: 'column' }}>
  <Text>First</Text>   {/* top */}
  <Text>Second</Text>  {/* below first */}
</View>

// Row — children sit horizontally side by side
<View style={{ flexDirection: 'row' }}>
  <Text>Left</Text>
  <Text>Right</Text>
</View>
```

| Value | Direction |
|---|---|
| `'column'` (default) | Top → Bottom |
| `'column-reverse'` | Bottom → Top |
| `'row'` | Left → Right |
| `'row-reverse'` | Right → Left |

---

### 2.2 justifyContent

Aligns children along the **main axis** (same direction as `flexDirection`).

```tsx
<View style={{ flex: 1, flexDirection: 'column', justifyContent: 'center' }}>
  {/* children centered vertically (since default is column) */}
</View>

<View style={{ flex: 1, flexDirection: 'row', justifyContent: 'space-between' }}>
  {/* children spread horizontally */}
</View>
```

| Value | Behavior |
|---|---|
| `'flex-start'` (default) | Pack children at start |
| `'flex-end'` | Pack children at end |
| `'center'` | Center children |
| `'space-between'` | Equal space between children, no edge space |
| `'space-around'` | Equal space around each child |
| `'space-evenly'` | Equal space between all children and edges |

---

### 2.3 alignItems

Aligns children along the **cross axis** (perpendicular to `flexDirection`).

```tsx
// column direction → alignItems controls horizontal alignment
<View style={{ flex: 1, flexDirection: 'column', alignItems: 'center' }}>
  {/* children centered horizontally */}
</View>

// row direction → alignItems controls vertical alignment
<View style={{ flex: 1, flexDirection: 'row', alignItems: 'center' }}>
  {/* children centered vertically */}
</View>
```

| Value | Behavior |
|---|---|
| `'stretch'` (default) | Children fill cross axis |
| `'flex-start'` | Children at start of cross axis |
| `'flex-end'` | Children at end of cross axis |
| `'center'` | Children centered on cross axis |
| `'baseline'` | Children aligned by text baseline |

**Center everything (common pattern):**
```tsx
<View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
  <Text>I am centered</Text>
</View>
```

---

### 2.4 alignSelf

Overrides `alignItems` for a **single child**. Same values as `alignItems`, but applied to the child, not the parent.

```tsx
<View style={{ flexDirection: 'row', alignItems: 'flex-start', height: 100 }}>
  <Text>Normal</Text>
  <Text style={{ alignSelf: 'center' }}>I'm centered</Text>
  <Text style={{ alignSelf: 'flex-end' }}>I'm at the bottom</Text>
</View>
```

---

### 2.5 flex, flexGrow, flexShrink, flexBasis

These control how children **share available space**.

**`flex`** — shorthand. `flex: 1` means "take all remaining space proportionally."

```tsx
<View style={{ flex: 1, flexDirection: 'row' }}>
  <View style={{ flex: 1, backgroundColor: 'red' }} />   {/* 1/3 of width */}
  <View style={{ flex: 2, backgroundColor: 'blue' }} />  {/* 2/3 of width */}
</View>
```

**`flexBasis`** — sets the initial size of a child before remaining space is distributed (equivalent to `width` in a row, `height` in a column).

```tsx
<View style={{ flexDirection: 'row' }}>
  <View style={{ flexBasis: 100, backgroundColor: 'red' }} />   {/* always 100px */}
  <View style={{ flex: 1, backgroundColor: 'blue' }} />         {/* fills the rest */}
</View>
```

**`flexGrow`** — how much a child grows to fill extra space (0 = don't grow).

**`flexShrink`** — how much a child shrinks when space is tight (0 = don't shrink).

```tsx
// Sidebar (fixed) + content (grows)
<View style={{ flex: 1, flexDirection: 'row' }}>
  <View style={{ flexBasis: 80, flexShrink: 0 }} />  {/* sidebar — never shrinks */}
  <View style={{ flexGrow: 1 }} />                   {/* content — grows to fill */}
</View>
```

---

### 2.6 gap property

`gap` adds spacing between children without adding margin to each one. Available since React Native 0.71.

```tsx
<View style={{ flexDirection: 'row', gap: 12 }}>
  <View style={styles.box} />
  <View style={styles.box} />
  <View style={styles.box} />
</View>

// Different row and column gaps
<View style={{ flexDirection: 'row', flexWrap: 'wrap', rowGap: 8, columnGap: 16 }}>
  {items.map(item => <Card key={item.id} />)}
</View>
```

> Before `gap` existed, developers added `marginRight` to each child and removed it from the last one — a common source of bugs. Use `gap` instead.

---

## 3. Responsive Design

### 3.1 Dimensions API

Returns the device screen dimensions as a static snapshot (does not update on rotation).

```tsx
import { Dimensions, StyleSheet } from 'react-native';

const { width, height } = Dimensions.get('window');
// 'window' = app's visible area (excludes status bar on Android)
// 'screen' = full physical screen

const styles = StyleSheet.create({
  halfScreen: {
    width: width / 2,
    height: height * 0.3,
  },
});
```

> Dimensions is fine for values used in `StyleSheet.create` (computed once), but for anything that changes on rotation, use `useWindowDimensions`.

---

### 3.2 useWindowDimensions Hook

Reactive version of the Dimensions API — re-renders the component when dimensions change (orientation change, foldable unfold, etc.).

```tsx
import { useWindowDimensions, View, Text } from 'react-native';

export default function ResponsiveLayout() {
  const { width, height, scale, fontScale } = useWindowDimensions();

  const columns = width > 600 ? 3 : 2;

  return (
    <View style={{ flexDirection: 'row', flexWrap: 'wrap' }}>
      {items.map(item => (
        <View key={item.id} style={{ width: width / columns, padding: 8 }}>
          <Text>{item.title}</Text>
        </View>
      ))}
    </View>
  );
}
```

| Property | Description |
|---|---|
| `width` | Window width in dp |
| `height` | Window height in dp |
| `scale` | Pixel density ratio |
| `fontScale` | User's font size accessibility setting |

---

### 3.3 PixelRatio

Maps between logical pixels (dp) and physical pixels. Important for sharp images and hairline borders.

```tsx
import { PixelRatio } from 'react-native';

// Get pixel density
PixelRatio.get();        // e.g., 3.0 on iPhone 14 Pro (3x screen)

// Convert dp to physical pixels
PixelRatio.getPixelSizeForLayoutSize(24); // 24dp → 72px on a 3x screen

// Hairline — thinnest possible visible line (1 physical pixel)
const HAIRLINE = StyleSheet.hairlineWidth; // = 1 / PixelRatio.get()

const styles = StyleSheet.create({
  divider: {
    height: StyleSheet.hairlineWidth,
    backgroundColor: '#ccc',
  },
});
```

**Loading the right image resolution:**
```tsx
function getImageSource(base: string) {
  const ratio = PixelRatio.get();
  if (ratio >= 3) return `${base}@3x.png`;
  if (ratio >= 2) return `${base}@2x.png`;
  return `${base}.png`;
}
```

> In practice, React Native automatically picks `@2x`/`@3x` image assets from your `assets/` folder — you only need `PixelRatio` for dynamic or server-side image URLs.

---

### 3.4 Percentage-Based Sizing

React Native supports percentage strings for `width`, `height`, `top`, `left`, `right`, `bottom`.

```tsx
<View style={{ width: '100%', height: '50%' }}>
  <View style={{ width: '80%', alignSelf: 'center' }}>
    <Text>Centered content at 80% width</Text>
  </View>
</View>
```

> Percentages are relative to the **parent container**, not the screen. Combine with `useWindowDimensions` when you need screen-relative sizing.

---

### 3.5 Aspect Ratio

Forces a component to maintain a width-to-height ratio regardless of its actual size.

```tsx
// Always a square
<View style={{ width: '100%', aspectRatio: 1 }} />

// 16:9 video thumbnail
<View style={{ width: '100%', aspectRatio: 16 / 9, backgroundColor: '#000' }} />

// Portrait card (3:4)
<View style={{ width: 120, aspectRatio: 3 / 4 }} />
```

> `aspectRatio` is particularly useful for images and media — set one dimension, let RN compute the other.

---

### 3.6 Handling Tablets and Foldables

Detect tablet form factor and adjust layout:

```tsx
import { useWindowDimensions } from 'react-native';

function useDeviceType() {
  const { width } = useWindowDimensions();
  return {
    isTablet: width >= 768,
    isPhone: width < 768,
  };
}

// Usage
export default function ProductGrid() {
  const { isTablet } = useDeviceType();

  return (
    <FlatList
      data={products}
      numColumns={isTablet ? 4 : 2}   // 4 columns on tablet, 2 on phone
      key={isTablet ? 'tablet' : 'phone'}  // key forces re-mount when columns change
      renderItem={({ item }) => <ProductCard item={item} />}
    />
  );
}
```

**Foldables** — devices like Samsung Galaxy Z Fold change their `width` dramatically when folded/unfolded. Since `useWindowDimensions` is reactive, the layout above handles foldables automatically.

> The `key` prop on `FlatList` is required when `numColumns` changes — RN cannot dynamically change column count without remounting the list.

---

### 3.7 Orientation Handling

```tsx
import { useWindowDimensions, StyleSheet, View, Text } from 'react-native';

export default function OrientationAware() {
  const { width, height } = useWindowDimensions();
  const isLandscape = width > height;

  return (
    <View style={[styles.container, isLandscape && styles.landscape]}>
      <Text>Mode: {isLandscape ? 'Landscape' : 'Portrait'}</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    flexDirection: 'column',
    padding: 16,
  },
  landscape: {
    flexDirection: 'row',  // switch to row layout in landscape
  },
});
```

**Lock orientation in `app.json` (Expo):**
```json
{
  "expo": {
    "orientation": "portrait"
  }
}
```

Values: `"portrait"` | `"landscape"` | `"default"` (both)

---

## 4. Safe Areas & Notches

Devices have notches, Dynamic Islands, home indicators, and rounded corners that can overlap your UI. Safe area handling prevents content from being hidden behind these.

### 4.1 SafeAreaView

The simplest solution — a `View` that automatically adds padding to avoid system UI intrusions.

```tsx
import { SafeAreaView, Text, StyleSheet } from 'react-native';

export default function App() {
  return (
    <SafeAreaView style={styles.container}>
      <Text>This text won't hide behind the notch or home bar</Text>
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#fff',
  },
});
```

**Limitation:** The built-in `SafeAreaView` from `react-native` only works on iOS. Use `react-native-safe-area-context` (below) for cross-platform support.

---

### 4.2 react-native-safe-area-context

The community standard for safe area handling. Works on both iOS and Android.

```bash
npx expo install react-native-safe-area-context
```

Wrap your entire app with the provider once (Expo Router does this in `_layout.tsx`):

```tsx
// app/_layout.tsx
import { SafeAreaProvider } from 'react-native-safe-area-context';

export default function RootLayout() {
  return (
    <SafeAreaProvider>
      {/* rest of your app */}
    </SafeAreaProvider>
  );
}
```

Use the safe-area-context version of `SafeAreaView`:

```tsx
import { SafeAreaView } from 'react-native-safe-area-context';

export default function HomeScreen() {
  return (
    <SafeAreaView style={{ flex: 1 }} edges={['top', 'left', 'right']}>
      {/* content */}
    </SafeAreaView>
  );
}
```

**`edges` prop** — control which edges get safe area padding:
```tsx
edges={['top']}              // only top (common for screens with a custom bottom nav)
edges={['top', 'bottom']}    // both (common for full-screen modal)
edges={['left', 'right']}    // only sides (for landscape content)
```

---

### 4.3 useSafeAreaInsets

When you need **precise inset values** to manually position elements — e.g., floating buttons above the home indicator.

```tsx
import { useSafeAreaInsets } from 'react-native-safe-area-context';
import { View, StyleSheet } from 'react-native';

export default function Screen() {
  const insets = useSafeAreaInsets();
  // insets.top    → space for status bar / notch
  // insets.bottom → space for home indicator
  // insets.left   → space for left edge (landscape on some devices)
  // insets.right  → space for right edge

  return (
    <View style={{ flex: 1, paddingTop: insets.top }}>
      {/* Custom header that respects the status bar */}

      {/* Floating action button above home indicator */}
      <View style={[styles.fab, { bottom: insets.bottom + 16 }]}>
        {/* FAB content */}
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  fab: {
    position: 'absolute',
    right: 16,
    width: 56,
    height: 56,
    borderRadius: 28,
    backgroundColor: '#007AFF',
    justifyContent: 'center',
    alignItems: 'center',
  },
});
```

---

### 4.4 StatusBar Component

Controls the appearance of the system status bar (battery, time, signal icons).

```tsx
import { StatusBar } from 'expo-status-bar';  // Expo version (recommended)

export default function App() {
  return (
    <>
      <StatusBar style="dark" />   {/* dark icons on light background */}
      {/* your app content */}
    </>
  );
}
```

| Prop | Values | Effect |
|---|---|---|
| `style` | `'auto'`, `'light'`, `'dark'` | Icon color |
| `backgroundColor` | color string | Android background color |
| `translucent` | boolean | Android: content draws under status bar |
| `hidden` | boolean | Hide status bar entirely |

```tsx
// Per-screen status bar — each screen can set its own
import { StatusBar } from 'expo-status-bar';

export default function DarkScreen() {
  return (
    <View style={{ flex: 1, backgroundColor: '#000' }}>
      <StatusBar style="light" />   {/* white icons for dark background */}
    </View>
  );
}
```

---

## 5. Styling Libraries

### 5.1 NativeWind v4 (Tailwind for RN)

NativeWind brings Tailwind CSS utility classes to React Native via the `className` prop. Best choice if your team already knows Tailwind.

```bash
npx expo install nativewind tailwindcss
npx tailwindcss init
```

**`tailwind.config.js`:**
```js
module.exports = {
  content: ['./app/**/*.{tsx,ts}', './src/**/*.{tsx,ts}'],
  presets: [require('nativewind/preset')],
  theme: { extend: {} },
  plugins: [],
};
```

**`babel.config.js`:**
```js
module.exports = {
  presets: [
    ['babel-preset-expo', { jsxImportSource: 'nativewind' }],
    'nativewind/babel',
  ],
};
```

**Usage:**
```tsx
import { View, Text, Pressable } from 'react-native';

export default function Card() {
  return (
    <View className="bg-white rounded-xl p-4 shadow-md">
      <Text className="text-lg font-bold text-gray-900">Title</Text>
      <Text className="text-sm text-gray-500 mt-1">Subtitle text here</Text>
      <Pressable className="mt-4 bg-blue-500 rounded-lg py-3 items-center active:opacity-70">
        <Text className="text-white font-semibold">Action</Text>
      </Pressable>
    </View>
  );
}
```

**Dark mode with NativeWind:**
```tsx
<View className="bg-white dark:bg-gray-900">
  <Text className="text-black dark:text-white">Adapts to system theme</Text>
</View>
```

**Pros:** Familiar to web devs, fast to write, automatic dark mode
**Cons:** Requires build-time compilation, class strings have no TypeScript autocomplete by default

---

### 5.2 Unistyles 3.0

A type-safe styling library with theming, breakpoints, and variants. No Tailwind knowledge needed.

```bash
npx expo install react-native-unistyles
```

**Setup theme:**
```ts
// src/styles/theme.ts
import { UnistylesRegistry } from 'react-native-unistyles';

const lightTheme = {
  colors: { background: '#fff', text: '#111', primary: '#007AFF' },
  spacing: { sm: 8, md: 16, lg: 24 },
};

const darkTheme = {
  colors: { background: '#111', text: '#fff', primary: '#0A84FF' },
  spacing: { sm: 8, md: 16, lg: 24 },
};

type AppThemes = {
  light: typeof lightTheme;
  dark: typeof darkTheme;
};

UnistylesRegistry
  .addThemes({ light: lightTheme, dark: darkTheme })
  .addConfig({ adaptiveThemes: true }); // auto light/dark
```

**Usage:**
```tsx
import { StyleSheet } from 'react-native-unistyles';
import { View, Text } from 'react-native';

export default function Card() {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Hello</Text>
    </View>
  );
}

const styles = StyleSheet.create(theme => ({
  container: {
    padding: theme.spacing.md,
    backgroundColor: theme.colors.background,
    borderRadius: 8,
  },
  title: {
    fontSize: 18,
    color: theme.colors.text,
  },
}));
```

**Breakpoints:**
```ts
UnistylesRegistry.addBreakpoints({
  sm: 0,
  md: 576,
  lg: 768,
  xl: 1024,
});

// In styles
const styles = StyleSheet.create(theme => ({
  container: {
    padding: { sm: 8, md: 16, lg: 24 },  // responsive padding
  },
}));
```

**Pros:** Fully type-safe, first-class theming, breakpoints, no build step
**Cons:** Smaller community than NativeWind, new API surface to learn

---

### 5.3 StyleSheet with Theme Context

The zero-dependency approach — build your own theme system with `useContext` and `StyleSheet`.

```tsx
// src/theme/ThemeContext.tsx
import { createContext, useContext } from 'react';

export const lightTheme = {
  colors: { background: '#fff', text: '#111', primary: '#007AFF', border: '#e5e5e5' },
  spacing: { xs: 4, sm: 8, md: 16, lg: 24, xl: 32 },
  fontSize: { sm: 12, md: 14, lg: 16, xl: 20, xxl: 24 },
  radius: { sm: 4, md: 8, lg: 16, full: 9999 },
};

export const darkTheme: typeof lightTheme = {
  colors: { background: '#111', text: '#f5f5f5', primary: '#0A84FF', border: '#333' },
  spacing: lightTheme.spacing,
  fontSize: lightTheme.fontSize,
  radius: lightTheme.radius,
};

type Theme = typeof lightTheme;

const ThemeContext = createContext<Theme>(lightTheme);

export function useTheme() {
  return useContext(ThemeContext);
}

export { ThemeContext, type Theme };
```

```tsx
// src/theme/ThemeProvider.tsx
import { useColorScheme } from 'react-native';
import { ThemeContext, lightTheme, darkTheme } from './ThemeContext';

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const scheme = useColorScheme();
  const theme = scheme === 'dark' ? darkTheme : lightTheme;

  return (
    <ThemeContext.Provider value={theme}>
      {children}
    </ThemeContext.Provider>
  );
}
```

```tsx
// Usage in a component
import { useTheme } from '@/theme/ThemeContext';
import { StyleSheet, View, Text } from 'react-native';

export default function Card() {
  const theme = useTheme();
  const styles = makeStyles(theme);

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Hello</Text>
    </View>
  );
}

function makeStyles(theme: ReturnType<typeof useTheme>) {
  return StyleSheet.create({
    container: {
      padding: theme.spacing.md,
      backgroundColor: theme.colors.background,
      borderRadius: theme.radius.md,
      borderWidth: 1,
      borderColor: theme.colors.border,
    },
    title: {
      fontSize: theme.fontSize.lg,
      color: theme.colors.text,
    },
  });
}
```

**Pros:** No extra dependencies, full control, easy to understand
**Cons:** Verbose — `makeStyles(theme)` in every component, no breakpoints built in

---

## 6. Theming

### 6.1 Light and Dark Mode

React Native apps automatically receive the device's color scheme preference. You respond to it using `useColorScheme` or by configuring your styling library.

Three strategies:

1. **`useColorScheme` hook** — detect and switch manually
2. **`Appearance` API** — imperative override (force light/dark regardless of system)
3. **Styling library** — NativeWind's `dark:` prefix or Unistyles' `adaptiveThemes`

---

### 6.2 useColorScheme Hook

Returns `'light'`, `'dark'`, or `null` (null means the device hasn't reported a preference yet).

```tsx
import { useColorScheme, View, Text, StyleSheet } from 'react-native';

export default function App() {
  const colorScheme = useColorScheme(); // 'light' | 'dark' | null

  const isDark = colorScheme === 'dark';

  return (
    <View style={[styles.container, isDark && styles.darkContainer]}>
      <Text style={[styles.text, isDark && styles.darkText]}>
        Current mode: {colorScheme}
      </Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#fff' },
  darkContainer: { backgroundColor: '#111' },
  text: { color: '#111' },
  darkText: { color: '#fff' },
});
```

---

### 6.3 Theme Provider Pattern

See [Section 5.3](#53-stylesheet-with-theme-context) for the full implementation. Here's how to wire it up at the app root:

```tsx
// app/_layout.tsx
import { ThemeProvider } from '@/theme/ThemeProvider';
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <ThemeProvider>
      <Stack />
    </ThemeProvider>
  );
}
```

Now every screen and component can call `useTheme()` to get the current theme object.

---

### 6.4 Dynamic Color Schemes

Override the system preference programmatically using the `Appearance` API:

```tsx
import { Appearance } from 'react-native';

// Force dark mode regardless of system setting
Appearance.setColorScheme('dark');

// Force light mode
Appearance.setColorScheme('light');

// Go back to following the system
Appearance.setColorScheme(null);
```

**User-controlled theme toggle:**
```tsx
import { useState } from 'react';
import { Appearance, Switch, View, Text } from 'react-native';

export default function ThemeToggle() {
  const [isDark, setIsDark] = useState(false);

  const toggle = (value: boolean) => {
    setIsDark(value);
    Appearance.setColorScheme(value ? 'dark' : 'light');
  };

  return (
    <View style={{ flexDirection: 'row', alignItems: 'center', gap: 8 }}>
      <Text>Dark Mode</Text>
      <Switch value={isDark} onValueChange={toggle} />
    </View>
  );
}
```

> Persist the user's choice in `AsyncStorage` or `expo-secure-store` so it survives app restarts.

---

## 7. Platform-Specific Styling

### 7.1 Platform.select for Styles

`Platform.select` returns the value matching the current OS — cleaner than `if/else` chains inside styles.

```tsx
import { Platform, StyleSheet } from 'react-native';

const styles = StyleSheet.create({
  header: {
    paddingTop: Platform.select({
      ios: 50,       // account for Dynamic Island / notch
      android: 24,   // account for status bar
      default: 0,    // web or other
    }),
    backgroundColor: Platform.select({
      ios: '#f8f8f8',
      android: '#ffffff',
    }),
  },
  text: {
    fontFamily: Platform.select({
      ios: 'San Francisco',  // iOS system font
      android: 'Roboto',     // Android system font
    }),
  },
});
```

---

### 7.2 Shadow vs Elevation

iOS and Android use completely different shadow APIs.

**iOS — 4 shadow properties:**
```tsx
const styles = StyleSheet.create({
  card: {
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },  // x and y offset
    shadowOpacity: 0.15,                     // 0–1
    shadowRadius: 4,                         // blur radius
    backgroundColor: '#fff',                // required for shadow to show
  },
});
```

**Android — single `elevation` value:**
```tsx
const styles = StyleSheet.create({
  card: {
    elevation: 4,           // 0–24, controls depth and shadow size
    backgroundColor: '#fff', // required
  },
});
```

**Cross-platform shadow utility:**
```tsx
// src/utils/shadow.ts
import { Platform } from 'react-native';

export function shadow(depth: 1 | 2 | 3 | 4) {
  const map = {
    1: { ios: { shadowOpacity: 0.08, shadowRadius: 2, shadowOffset: { width: 0, height: 1 } }, android: 2 },
    2: { ios: { shadowOpacity: 0.12, shadowRadius: 4, shadowOffset: { width: 0, height: 2 } }, android: 4 },
    3: { ios: { shadowOpacity: 0.16, shadowRadius: 6, shadowOffset: { width: 0, height: 3 } }, android: 6 },
    4: { ios: { shadowOpacity: 0.20, shadowRadius: 8, shadowOffset: { width: 0, height: 4 } }, android: 8 },
  };

  return Platform.select({
    ios: { shadowColor: '#000', ...map[depth].ios },
    android: { elevation: map[depth].android },
    default: {},
  });
}

// Usage
<View style={[styles.card, shadow(2)]}>
```

---

### 7.3 Font Handling Across Platforms

**Step 1 — Add font files to `assets/fonts/`:**
```
assets/
  fonts/
    Inter-Regular.ttf
    Inter-Medium.ttf
    Inter-Bold.ttf
```

**Step 2 — Load fonts with `expo-font`:**
```tsx
// app/_layout.tsx
import { useFonts } from 'expo-font';
import * as SplashScreen from 'expo-splash-screen';
import { useEffect } from 'react';

SplashScreen.preventAutoHideAsync();

export default function RootLayout() {
  const [loaded, error] = useFonts({
    'Inter-Regular': require('../assets/fonts/Inter-Regular.ttf'),
    'Inter-Medium': require('../assets/fonts/Inter-Medium.ttf'),
    'Inter-Bold': require('../assets/fonts/Inter-Bold.ttf'),
  });

  useEffect(() => {
    if (loaded || error) SplashScreen.hideAsync();
  }, [loaded, error]);

  if (!loaded && !error) return null;

  return <Stack />;
}
```

**Step 3 — Use by font name (not file name):**
```tsx
const styles = StyleSheet.create({
  title: {
    fontFamily: 'Inter-Bold',
    fontSize: 24,
  },
  body: {
    fontFamily: 'Inter-Regular',
    fontSize: 14,
  },
});
```

**Platform behavior differences:**

| Behavior | iOS | Android |
|---|---|---|
| Default system font | SF Pro | Roboto |
| `fontWeight` without custom font | Works | Sometimes ignored — use separate font files |
| `letterSpacing` | Works | Works (called `letterSpacing` in RN, not `letter-spacing`) |
| `fontStyle: 'italic'` | Works | Needs a separate italic `.ttf` file |

> On Android, avoid relying on `fontWeight: 'bold'` with custom fonts — Android often ignores it. Instead, load a separate bold variant (e.g., `Inter-Bold.ttf`) and set `fontFamily: 'Inter-Bold'` directly.

---

*End of Module 3*
