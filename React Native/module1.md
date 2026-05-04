# Module 1 — React Native Fundamentals

> Targeted at developers with React JS experience. Concepts shared with React JS are kept concise; RN-specific differences are highlighted.

---

## Table of Contents

1. [Hooks Essentials](#1-hooks-essentials)
   - 1.1 [useState](#11-usestate)
   - 1.2 [useEffect and Cleanup Functions](#12-useeffect-and-cleanup-functions)
   - 1.3 [useRef](#13-useref)
   - 1.4 [useMemo](#14-usememo)
   - 1.5 [useCallback](#15-usecallback)
   - 1.6 [useContext](#16-usecontext)
   - 1.7 [useReducer](#17-usereducer)
   - 1.8 [Custom Hooks](#18-custom-hooks)

2. [Component Patterns](#2-component-patterns)
   - 2.1 [Functional Components](#21-functional-components)
   - 2.2 [Props and Prop Drilling](#22-props-and-prop-drilling)
   - 2.3 [Children and Composition](#23-children-and-composition)
   - 2.4 [Conditional Rendering](#24-conditional-rendering)
   - 2.5 [Lists and Keys](#25-lists-and-keys)
   - 2.6 [Lifting State Up](#26-lifting-state-up)
   - 2.7 [Controlled vs Uncontrolled Components](#27-controlled-vs-uncontrolled-components)

3. [React Native Specifics](#3-react-native-specifics)
   - 3.1 [Core Components](#31-core-components)
   - 3.2 [Differences from React DOM](#32-differences-from-react-dom)
   - 3.3 [JSX Rules Unique to RN](#33-jsx-rules-unique-to-rn)
   - 3.4 [Platform Module](#34-platform-module)
   - 3.5 [Dimensions API and useWindowDimensions](#35-dimensions-api-and-usewindowdimensions)
   - 3.6 [Platform-Specific File Extensions](#36-platform-specific-file-extensions)

---

## 1. Hooks Essentials

### 1.1 useState

Same as React JS. Manages local component state.

```tsx
import { useState } from 'react';
import { View, Text, Button } from 'react-native';

export default function Counter() {
  const [count, setCount] = useState(0);

  return (
    <View>
      <Text>{count}</Text>
      <Button title="Increment" onPress={() => setCount(count + 1)} />
    </View>
  );
}
```

> Note: `onClick` becomes `onPress` in React Native — more on this in Section 3.

---

### 1.2 useEffect and Cleanup Functions

Runs side effects after render. Cleanup function runs before the effect re-runs or on unmount — critical for subscriptions and timers.

```tsx
import { useState, useEffect } from 'react';
import { Text } from 'react-native';

export default function Timer() {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      setSeconds(prev => prev + 1);
    }, 1000);

    // Cleanup: clears the interval when component unmounts
    return () => clearInterval(interval);
  }, []); // Empty array = run once on mount

  return <Text>Elapsed: {seconds}s</Text>;
}
```

**Dependency array rules:**
| Array | Behavior |
|-------|----------|
| No array | Runs after every render |
| `[]` | Runs once on mount |
| `[value]` | Runs when `value` changes |

---

### 1.3 useRef

Two common uses:
1. **Persist a mutable value** across renders without triggering re-render.
2. **Reference a native element** (like focusing a TextInput).

```tsx
import { useRef } from 'react';
import { TextInput, Button, View } from 'react-native';

export default function FocusInput() {
  const inputRef = useRef<TextInput>(null);

  return (
    <View>
      <TextInput ref={inputRef} placeholder="Type here..." />
      <Button title="Focus" onPress={() => inputRef.current?.focus()} />
    </View>
  );
}
```

> In React JS you'd reference a `<input>` DOM element; in RN you reference a native `TextInput` component.

---

### 1.4 useMemo

Memoizes the **result of a computation** — recalculates only when dependencies change. Use it to avoid expensive recalculations on every render.

```tsx
import { useState, useMemo } from 'react';
import { Text, Button, View } from 'react-native';

function expensiveCalc(n: number) {
  // Simulate heavy work
  return n * n;
}

export default function App() {
  const [num, setNum] = useState(5);
  const [other, setOther] = useState(0);

  // Only recalculates when `num` changes, not when `other` changes
  const result = useMemo(() => expensiveCalc(num), [num]);

  return (
    <View>
      <Text>Result: {result}</Text>
      <Button title="Change Num" onPress={() => setNum(n => n + 1)} />
      <Button title="Change Other" onPress={() => setOther(o => o + 1)} />
    </View>
  );
}
```

---

### 1.5 useCallback

Memoizes a **function reference** — returns the same function instance across renders unless dependencies change. Prevents child re-renders when passing callbacks as props.

```tsx
import { useState, useCallback } from 'react';
import { Button, View } from 'react-native';

// Without useCallback, handlePress would be a NEW function on every render,
// causing ChildButton to re-render unnecessarily.
const ChildButton = React.memo(({ onPress }: { onPress: () => void }) => (
  <Button title="Press Me" onPress={onPress} />
));

export default function Parent() {
  const [count, setCount] = useState(0);

  const handlePress = useCallback(() => {
    setCount(c => c + 1);
  }, []); // stable reference — no dependencies

  return (
    <View>
      <ChildButton onPress={handlePress} />
    </View>
  );
}
```

> **Rule of thumb:** use `useCallback` when passing a function to a `React.memo` child or as a dependency in another hook.

---

### 1.6 useContext

Share data across the component tree without prop drilling.

```tsx
import { createContext, useContext, useState } from 'react';
import { Text, View } from 'react-native';

// 1. Create context
const ThemeContext = createContext<'light' | 'dark'>('light');

// 2. Provide it at a parent level
export default function App() {
  const [theme] = useState<'light' | 'dark'>('dark');

  return (
    <ThemeContext.Provider value={theme}>
      <Screen />
    </ThemeContext.Provider>
  );
}

// 3. Consume it anywhere in the tree
function Screen() {
  const theme = useContext(ThemeContext);
  return (
    <View style={{ backgroundColor: theme === 'dark' ? '#000' : '#fff' }}>
      <Text style={{ color: theme === 'dark' ? '#fff' : '#000' }}>
        Current theme: {theme}
      </Text>
    </View>
  );
}
```

---

### 1.7 useReducer

Alternative to `useState` for **complex state logic** — similar to Redux but local to a component.

```tsx
import { useReducer } from 'react';
import { Text, Button, View } from 'react-native';

type State = { count: number };
type Action = { type: 'increment' } | { type: 'decrement' } | { type: 'reset' };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment': return { count: state.count + 1 };
    case 'decrement': return { count: state.count - 1 };
    case 'reset':     return { count: 0 };
  }
}

export default function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });

  return (
    <View>
      <Text>Count: {state.count}</Text>
      <Button title="+" onPress={() => dispatch({ type: 'increment' })} />
      <Button title="-" onPress={() => dispatch({ type: 'decrement' })} />
      <Button title="Reset" onPress={() => dispatch({ type: 'reset' })} />
    </View>
  );
}
```

> Use `useReducer` over `useState` when state has multiple sub-values or next state depends on the previous one.

---

### 1.8 Custom Hooks

Extract reusable stateful logic into a function prefixed with `use`.

```tsx
// hooks/useOnlineStatus.ts
import { useState, useEffect } from 'react';
import NetInfo from '@react-native-community/netinfo';

export function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(true);

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener(state => {
      setIsOnline(state.isConnected ?? false);
    });
    return unsubscribe; // cleanup
  }, []);

  return isOnline;
}

// Usage in any component
import { useOnlineStatus } from '../hooks/useOnlineStatus';

function Banner() {
  const isOnline = useOnlineStatus();
  return <Text>{isOnline ? 'Online' : 'Offline'}</Text>;
}
```

> Custom hooks let you share logic, not UI — the same way you'd extract utility functions, but for stateful behavior.

---

## 2. Component Patterns

### 2.1 Functional Components

React Native uses functional components exclusively in modern code. Syntax is identical to React JS, just with RN primitives instead of HTML.

```tsx
import { View, Text } from 'react-native';

type Props = {
  name: string;
};

export default function Greeting({ name }: Props) {
  return (
    <View>
      <Text>Hello, {name}!</Text>
    </View>
  );
}
```

---

### 2.2 Props and Prop Drilling

Passing data from parent → child → grandchild. Same concept as React JS.

```tsx
// Parent
<ProfileCard userId="42" username="Nawaz" />

// Child
function ProfileCard({ userId, username }: { userId: string; username: string }) {
  return <Avatar userId={userId} username={username} />;
}

// Grandchild — receives props it may not even use just to pass them down
function Avatar({ userId, username }: { userId: string; username: string }) {
  return <Text>{username}</Text>;
}
```

**Problem:** When the tree is deep, intermediate components carry props they don't use.
**Solution:** `useContext` (see 1.6) or a state manager like Zustand/Redux.

---

### 2.3 Children and Composition

Use `children` to build flexible wrapper components — same as React JS.

```tsx
import { View, StyleSheet } from 'react-native';

type Props = {
  children: React.ReactNode;
};

function Card({ children }: Props) {
  return <View style={styles.card}>{children}</View>;
}

// Usage
<Card>
  <Text>Title</Text>
  <Text>Subtitle</Text>
</Card>

const styles = StyleSheet.create({
  card: {
    padding: 16,
    borderRadius: 8,
    backgroundColor: '#fff',
    elevation: 2, // Android shadow
    shadowColor: '#000', // iOS shadow
    shadowOpacity: 0.1,
    shadowRadius: 4,
  },
});
```

---

### 2.4 Conditional Rendering

Same as React JS — use ternary or `&&` operator.

```tsx
function StatusBadge({ isActive }: { isActive: boolean }) {
  return (
    <View>
      {isActive ? (
        <Text style={{ color: 'green' }}>Active</Text>
      ) : (
        <Text style={{ color: 'red' }}>Inactive</Text>
      )}

      {/* Short-circuit: only renders when isActive is true */}
      {isActive && <Text>Welcome back!</Text>}
    </View>
  );
}
```

---

### 2.5 Lists and Keys

In React JS you use `.map()` over an array. In React Native, **always prefer `FlatList`** for long lists — it virtualizes rendering (only renders visible items).

```tsx
import { FlatList, Text, View } from 'react-native';

const DATA = [
  { id: '1', title: 'Item One' },
  { id: '2', title: 'Item Two' },
  { id: '3', title: 'Item Three' },
];

export default function MyList() {
  return (
    <FlatList
      data={DATA}
      keyExtractor={item => item.id}       // equivalent to key prop
      renderItem={({ item }) => (
        <View>
          <Text>{item.title}</Text>
        </View>
      )}
    />
  );
}
```

> Using `.map()` inside a `ScrollView` works for short static lists but will cause performance issues for large datasets. Always use `FlatList` for dynamic data.

---

### 2.6 Lifting State Up

When two sibling components need to share state, move the state to their closest common parent.

```tsx
export default function Parent() {
  const [text, setText] = useState('');

  return (
    <View>
      {/* Input writes to parent's state */}
      <SearchInput value={text} onChange={setText} />
      {/* Results reads from parent's state */}
      <ResultList query={text} />
    </View>
  );
}

function SearchInput({ value, onChange }: { value: string; onChange: (v: string) => void }) {
  return <TextInput value={value} onChangeText={onChange} />;
}

function ResultList({ query }: { query: string }) {
  return <Text>Searching for: {query}</Text>;
}
```

---

### 2.7 Controlled vs Uncontrolled Components

| | Controlled | Uncontrolled |
|---|---|---|
| State lives in | React (useState) | Native element (ref) |
| How to read value | From state variable | Via `ref.current` |
| When to use | Forms needing validation, instant feedback | Simple cases, third-party integrations |

**Controlled (recommended in RN):**
```tsx
const [email, setEmail] = useState('');

<TextInput
  value={email}
  onChangeText={setEmail}
  placeholder="Enter email"
/>
```

**Uncontrolled:**
```tsx
const inputRef = useRef<TextInput>(null);

// Read value on submit
const handleSubmit = () => {
  // Note: RN TextInput doesn't expose .value directly via ref
  // Uncontrolled is less common in RN — controlled is the standard pattern
};

<TextInput ref={inputRef} placeholder="Enter email" />
```

> In React Native, **controlled components are the standard**. Unlike the web, there's no direct DOM `.value` access, so uncontrolled inputs are rarely used.

---

## 3. React Native Specifics

### 3.1 Core Components

React Native has no HTML — it maps to native UI components on iOS and Android. Here are the essential ones:

| RN Component | Web Equivalent | Purpose |
|---|---|---|
| `View` | `div` | Container / layout |
| `Text` | `p`, `span`, `h1` | All text rendering |
| `Image` | `img` | Display images |
| `ScrollView` | `div` with overflow | Scrollable container |
| `TextInput` | `input` | Text input field |
| `Pressable` | `button` | Touchable/clickable area |
| `FlatList` | virtualized list | Performant long lists |
| `Modal` | modal dialog | Overlay screens |
| `ActivityIndicator` | loading spinner | Loading state |

```tsx
import { View, Text, Image, ScrollView, Pressable, StyleSheet } from 'react-native';

export default function ProfileScreen() {
  return (
    <ScrollView>
      <View style={styles.container}>
        <Image
          source={{ uri: 'https://example.com/avatar.png' }}
          style={styles.avatar}
        />
        <Text style={styles.name}>Nawaz Mujawar</Text>

        <Pressable
          style={({ pressed }) => [styles.button, pressed && styles.buttonPressed]}
          onPress={() => console.log('Pressed!')}
        >
          <Text style={styles.buttonText}>Follow</Text>
        </Pressable>
      </View>
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  container: { alignItems: 'center', padding: 20 },
  avatar: { width: 80, height: 80, borderRadius: 40 },
  name: { fontSize: 18, fontWeight: 'bold', marginTop: 8 },
  button: { backgroundColor: '#007AFF', padding: 12, borderRadius: 8, marginTop: 12 },
  buttonPressed: { opacity: 0.7 },
  buttonText: { color: '#fff', fontWeight: '600' },
});
```

---

### 3.2 Differences from React DOM

| Feature | React (Web) | React Native |
|---|---|---|
| Styling | CSS classes / inline styles (strings) | StyleSheet objects (JS numbers/objects) |
| Layout | CSS Flexbox (default row) | Flexbox (default **column**) |
| Events | `onClick`, `onChange` | `onPress`, `onChangeText` |
| Navigation | React Router, browser history | React Navigation (stack, tabs) |
| Animations | CSS transitions | Animated API / Reanimated |
| Fonts | Any web font | Must link native fonts |
| No CSS units | `px`, `em`, `rem` | Numbers only (density-independent pixels) |
| No `class` | `className` | `style` prop only |

**Flexbox difference example:**

```tsx
// Web: default flexDirection is 'row'
// React Native: default flexDirection is 'column'

<View style={{ flexDirection: 'row', gap: 8 }}>
  <View style={{ flex: 1, backgroundColor: 'red' }} />
  <View style={{ flex: 2, backgroundColor: 'blue' }} />
</View>
```

---

### 3.3 JSX Rules Unique to RN

**Rule 1: All text MUST be inside a `<Text>` component.**

```tsx
// WRONG — will throw an error
<View>
  Hello World
</View>

// CORRECT
<View>
  <Text>Hello World</Text>
</View>
```

**Rule 2: No HTML elements — everything is RN components.**

```tsx
// WRONG
<div style={{ padding: 10 }}>
  <h1>Title</h1>
  <p>Paragraph</p>
  <img src="..." />
</div>

// CORRECT
<View style={{ padding: 10 }}>
  <Text style={{ fontSize: 24, fontWeight: 'bold' }}>Title</Text>
  <Text>Paragraph</Text>
  <Image source={{ uri: '...' }} style={{ width: 100, height: 100 }} />
</View>
```

**Rule 3: `StyleSheet.create` instead of CSS strings.**

```tsx
import { StyleSheet } from 'react-native';

// WRONG
<View style="padding: 10px; background-color: red;" />

// CORRECT
<View style={styles.box} />

const styles = StyleSheet.create({
  box: {
    padding: 10,
    backgroundColor: 'red',
  },
});
```

**Rule 4: No `px` — use plain numbers (they are density-independent pixels).**

```tsx
// WRONG
{ fontSize: '16px', margin: '8px' }

// CORRECT
{ fontSize: 16, margin: 8 }
```

---

### 3.4 Platform Module

Lets you write platform-specific code within a single file.

```tsx
import { Platform, StyleSheet, Text, View } from 'react-native';

export default function Header() {
  return (
    <View style={styles.header}>
      <Text style={styles.title}>My App</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  header: {
    // Platform.OS returns 'ios' | 'android' | 'web'
    paddingTop: Platform.OS === 'ios' ? 50 : 30,
    backgroundColor: '#007AFF',
  },
  title: {
    // Platform.select returns the matching value for current OS
    fontSize: Platform.select({
      ios: 18,
      android: 16,
      default: 17,
    }),
    color: '#fff',
  },
});
```

**`Platform.OS`** — returns a string: `'ios'`, `'android'`, or `'web'`.

**`Platform.select`** — cleaner for multiple branches, accepts an object with OS keys.

```tsx
const shadow = Platform.select({
  ios: {
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.2,
    shadowRadius: 4,
  },
  android: {
    elevation: 4,
  },
});

<View style={[styles.card, shadow]}>
```

---

### 3.5 Dimensions API and useWindowDimensions

Get the screen's width and height for responsive layouts.

**`Dimensions` API (static — does not update on rotation):**

```tsx
import { Dimensions } from 'react-native';

const { width, height } = Dimensions.get('window');
// 'window' = visible app area  |  'screen' = full device screen (includes status bar)

const styles = StyleSheet.create({
  halfWidth: {
    width: width / 2,
  },
});
```

**`useWindowDimensions` hook (reactive — updates on rotation/resize):**

```tsx
import { useWindowDimensions, View, Text } from 'react-native';

export default function ResponsiveCard() {
  const { width, height } = useWindowDimensions();

  const isLandscape = width > height;

  return (
    <View style={{ flexDirection: isLandscape ? 'row' : 'column' }}>
      <Text>Width: {width}</Text>
      <Text>Height: {height}</Text>
    </View>
  );
}
```

> **Prefer `useWindowDimensions`** over `Dimensions` — it automatically re-renders when the screen orientation changes.

---

### 3.6 Platform-Specific File Extensions

React Native's bundler (Metro) automatically picks the right file based on the platform. No imports need to change.

**File naming convention:**

```
components/
  Button.tsx           ← shared / fallback
  Button.ios.tsx       ← used on iOS only
  Button.android.tsx   ← used on Android only
```

**Import stays the same regardless of platform:**

```tsx
import Button from './components/Button';
// Metro automatically resolves to Button.ios.tsx on iOS
// and Button.android.tsx on Android
```

**When to use this:**

```tsx
// Button.ios.tsx — iOS has a native segmented control feel
import { TouchableOpacity, Text } from 'react-native';

export default function Button({ title, onPress }: { title: string; onPress: () => void }) {
  return (
    <TouchableOpacity
      style={{ backgroundColor: '#007AFF', padding: 12, borderRadius: 10 }}
      onPress={onPress}
    >
      <Text style={{ color: '#fff' }}>{title}</Text>
    </TouchableOpacity>
  );
}

// Button.android.tsx — Android uses Material-style ripple
import { Pressable, Text } from 'react-native';
import { android_ripple } from './utils';

export default function Button({ title, onPress }: { title: string; onPress: () => void }) {
  return (
    <Pressable
      android_ripple={{ color: '#rgba(0,0,0,0.2)' }}
      style={{ backgroundColor: '#6200EE', padding: 12, borderRadius: 4 }}
      onPress={onPress}
    >
      <Text style={{ color: '#fff' }}>{title}</Text>
    </Pressable>
  );
}
```

> Use platform files for **significant UI differences**. For minor tweaks (padding, font size), `Platform.select` inside a single file is simpler and more readable.

---

*End of Module 1*
