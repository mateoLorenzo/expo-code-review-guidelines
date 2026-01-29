# React Native + Expo Code Guidelines

> Code review standards for React Native + Expo projects.
> Based on [expo/skills](https://github.com/expo/skills), [callstackincubator/agent-skills](https://github.com/callstackincubator/agent-skills), and Callstack's "Ultimate Guide to React Native Optimization".

---

## Table of Contents

1. [Performance - Lists & Rendering](#1-performance---lists--rendering)
2. [Performance - Bundle Size](#2-performance---bundle-size)
3. [Performance - Memory Management](#3-performance---memory-management)
4. [Networking & Data Fetching](#4-networking--data-fetching)
5. [Authentication & Security](#5-authentication--security)
6. [Navigation & Routing](#6-navigation--routing)
7. [Styling & UI](#7-styling--ui)
8. [Components & Architecture](#8-components--architecture)
9. [Animations](#9-animations)
10. [Storage](#10-storage)
11. [Code Style & Conventions](#11-code-style--conventions)
12. [Library Preferences](#12-library-preferences)

---

## 1. Performance - Lists & Rendering

### 1.1 Use FlashList/FlatList for Lists

**What:** Use FlashList or FlatList instead of ScrollView with .map() for lists with more than 10-20 items.

**Why:** ScrollView renders ALL items at once, causing freezes, FPS drops to 0, and high memory usage. Virtualized lists only render visible items.

```tsx
// BAD - Renders all items at once, causes freezes
<ScrollView>
  {items.map((item) => <Item key={item.id} {...item} />)}
</ScrollView>

// GOOD - Virtualized rendering
<FlashList
  data={items}
  renderItem={({ item }) => <Item {...item} />}
  estimatedItemSize={50}
/>
```

**Decision Matrix:**
| Scenario | Recommendation |
|----------|---------------|
| < 20 static items | ScrollView OK |
| 20-100 items | FlatList minimum |
| > 100 items | FlashList |
| Complex item layouts | FlashList with `getItemType` |

### 1.2 Proper keyExtractor

**What:** Use unique item IDs for keyExtractor, not array indices.

**Why:** Using indices causes incorrect recycling when items are added/removed/reordered, leading to visual bugs and performance issues.

```tsx
// BAD - Index-based keys cause bugs
keyExtractor={(item, index) => index.toString()}

// GOOD - Unique ID-based keys
keyExtractor={(item) => item.id}
```

### 1.3 Memoize renderItem Functions

**What:** Define renderItem functions outside the component or use useCallback.

**Why:** Inline functions cause re-renders on every parent update.

```tsx
// BAD - Inline function recreated every render
<FlatList data={items} renderItem={({ item }) => <Item {...item} />} />;

// GOOD - Stable function reference
const renderItem = useCallback(({ item }) => <Item {...item} />, []);

<FlatList data={items} renderItem={renderItem} />;
```

### 1.4 React Compiler (Expo SDK 52+)

**What:** Enable React Compiler for automatic memoization instead of manual memo/useMemo/useCallback.

**Why:** Eliminates boilerplate and ensures consistent optimization across the codebase.

```json
// app.json
{
  "expo": {
    "experiments": {
      "reactCompiler": true
    }
  }
}
```

---

## 2. Performance - Bundle Size

### 2.1 Avoid Barrel Exports

**What:** Import directly from source files instead of barrel exports (index.ts).

**Why:** Barrel imports load ALL exports even if you use only one, increasing bundle size and slowing TTI. Also causes circular dependency issues.

```tsx
// BAD - Loads ALL exports from components/index.ts
import { Button } from "./components";

// GOOD - Loads only Button
import Button from "./components/Button";
```

### 2.2 Direct Library Imports

**What:** Import specific functions from libraries instead of the entire library.

**Why:** Reduces bundle size significantly, especially with large libraries like date-fns, lodash.

```tsx
// BAD - Imports entire library
import { format, addDays, isToday } from "date-fns";

// GOOD - Direct imports
import format from "date-fns/format";
import addDays from "date-fns/addDays";
import isToday from "date-fns/isToday";
```

### 2.3 Enable Tree Shaking (Expo SDK 52+)

**What:** Enable experimental tree shaking for automatic dead code elimination.

```bash
# .env
EXPO_UNSTABLE_METRO_OPTIMIZE_GRAPH=1
EXPO_UNSTABLE_TREE_SHAKING=1
```

---

## 3. Performance - Memory Management

### 3.1 useEffect Cleanup

**What:** Always return a cleanup function in useEffect for subscriptions, timers, and listeners.

**Why:** Missing cleanup causes memory leaks that grow over time and can crash the app.

```tsx
// BAD - Memory leak, listener never removed
useEffect(() => {
  const sub = EventEmitter.addListener("event", handler);
}, []);

// GOOD - Proper cleanup
useEffect(() => {
  const sub = EventEmitter.addListener("event", handler);
  return () => sub.remove();
}, []);
```

### 3.2 Timer Cleanup

**What:** Always clear intervals and timeouts in useEffect cleanup.

```tsx
// BAD - Timer continues after unmount
useEffect(() => {
  const timer = setInterval(() => {
    setCount((prev) => prev + 1);
  }, 1000);
}, []);

// GOOD - Timer cleared on unmount
useEffect(() => {
  const timer = setInterval(() => {
    setCount((prev) => prev + 1);
  }, 1000);
  return () => clearInterval(timer);
}, []);
```

### 3.3 Avoid Closure Memory Leaks

**What:** Don't capture large objects in closures when only a small value is needed.

```tsx
// BAD - Closure captures entire array
class BadExample {
  private largeData = new Array(1000000).fill("data");

  createFunction() {
    return () => this.largeData.length; // Captures this.largeData
  }
}

// GOOD - Only capture what's needed
class GoodExample {
  private largeData = new Array(1000000).fill("data");

  createFunction() {
    const length = this.largeData.length;
    return () => length; // Only captures primitive
  }
}
```

---

## 4. Networking & Data Fetching

### 4.1 Prefer fetch over axios

**What:** Use the native fetch API instead of axios.

**Why:** fetch is built-in, has a smaller footprint, and Expo provides optimized networking.

```tsx
// BAD
import axios from "axios";
const response = await axios.get(url);

// GOOD
const response = await fetch(url);
const data = await response.json();
```

### 4.2 Always Check response.ok

**What:** Always check response.ok before parsing JSON in fetch requests.

**Why:** fetch doesn't throw on HTTP errors (4xx, 5xx). Without checking, you'll try to parse error pages as JSON.

```tsx
// BAD - No error handling
const data = await fetch(url).then((r) => r.json());

// GOOD - Proper error handling
const response = await fetch(url);
if (!response.ok) {
  throw new Error(`HTTP error! status: ${response.status}`);
}
const data = await response.json();
```

### 4.3 Use React Query for Data Management

**What:** Use TanStack Query (React Query) for server state management.

**Why:** Provides caching, deduplication, background refetching, and error handling out of the box.

```tsx
// Setup in _layout.tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 minutes
      retry: 2,
    },
  },
});

// Usage
const { data, isLoading, error } = useQuery({
  queryKey: ["user", userId],
  queryFn: () => fetchUser(userId),
});
```

### 4.4 Request Cancellation

**What:** Cancel requests on component unmount using AbortController.

```tsx
useEffect(() => {
  const controller = new AbortController();

  fetch(url, { signal: controller.signal })
    .then((response) => response.json())
    .then(setData)
    .catch((error) => {
      if (error.name !== "AbortError") {
        setError(error);
      }
    });

  return () => controller.abort();
}, [url]);
```

### 4.5 Environment Variables

**What:** Use EXPO*PUBLIC* prefix for client-side environment variables.

**Why:** Only EXPO*PUBLIC* prefixed variables are exposed to the client bundle. Never put secrets in these variables.

```bash
# .env
EXPO_PUBLIC_API_URL=https://api.example.com

# Usage
const API_URL = process.env.EXPO_PUBLIC_API_URL;
```

---

## 5. Authentication & Security

### 5.1 Use SecureStore for Tokens

**What:** Use expo-secure-store for storing authentication tokens, not AsyncStorage.

**Why:** AsyncStorage is not encrypted and stores data in plain text. SecureStore uses the device's secure keychain/keystore.

```tsx
// BAD - Not secure, plain text storage
await AsyncStorage.setItem("token", token);

// GOOD - Encrypted storage
import * as SecureStore from "expo-secure-store";
await SecureStore.setItemAsync("token", token);
```

### 5.2 Token Refresh Pattern

**What:** Implement proper token refresh with race condition handling.

```tsx
let isRefreshing = false;
let refreshPromise: Promise<string> | null = null;

const getValidToken = async (): Promise<string> => {
  const token = await SecureStore.getItemAsync("token");

  if (!token || isTokenExpired(token)) {
    if (!isRefreshing) {
      isRefreshing = true;
      refreshPromise = refreshToken().finally(() => {
        isRefreshing = false;
        refreshPromise = null;
      });
    }
    return refreshPromise!;
  }

  return token;
};
```

---

## 6. Navigation & Routing

### 6.1 Route File Organization

**What:** Never co-locate components, types, or utilities in the app directory.

**Why:** The app directory should only contain route files. Co-location is an anti-pattern that makes navigation structure unclear.

```
// BAD
app/
  index.tsx
  components/    <- Wrong location
  utils/         <- Wrong location

// GOOD
app/
  _layout.tsx
  index.tsx
  (auth)/
    login.tsx
src/
  components/
  utils/
  types/
```

### 6.2 Always Use \_layout.tsx for Stacks

**What:** Define navigation stacks in \_layout.tsx files, not inline.

```tsx
// app/(main)/_layout.tsx
import { Stack } from "expo-router/stack";

export default function Layout() {
  return (
    <Stack screenOptions={{ headerLargeTitle: true }}>
      <Stack.Screen name="index" options={{ title: "Home" }} />
    </Stack>
  );
}
```

### 6.3 Use Native Navigation Titles

**What:** Use Stack.Screen options title instead of custom Text elements for page titles.

**Why:** Native navigation titles integrate with the system, support large titles, and handle safe areas automatically.

```tsx
// BAD - Custom title
<View>
  <Text style={styles.pageTitle}>Home</Text>
</View>

// GOOD - Native title
<Stack.Screen options={{ title: 'Home' }} />
```

### 6.4 Use Link for Navigation

**What:** Use Link from expo-router for navigation, not imperative navigation.

```tsx
import { Link } from 'expo-router';

// Navigation
<Link href="/settings">Settings</Link>

// With custom component
<Link href="/profile" asChild>
  <Pressable>
    <Text>Profile</Text>
  </Pressable>
</Link>
```

### 6.5 Ensure Root Route Exists

**What:** Always have a route that matches "/" so the app is never blank.

---

## 7. Styling & UI

### 7.1 Safe Area Handling

**What:** Use ScrollView with contentInsetAdjustmentBehavior="automatic" instead of SafeAreaView from react-native.

**Why:** Provides smarter safe area handling and works better with navigation headers.

```tsx
// BAD
import { SafeAreaView } from 'react-native';
<SafeAreaView>...</SafeAreaView>

// GOOD
<ScrollView contentInsetAdjustmentBehavior="automatic">
  ...
</ScrollView>
```

### 7.2 Use useWindowDimensions

**What:** Use useWindowDimensions hook instead of Dimensions.get().

**Why:** useWindowDimensions updates automatically on orientation/size changes, Dimensions.get() returns stale values.

```tsx
// BAD - Stale on rotation
const { width, height } = Dimensions.get("window");

// GOOD - Reactive to changes
const { width, height } = useWindowDimensions();
```

### 7.3 Modern Shadow Syntax

**What:** Use CSS boxShadow style instead of legacy React Native shadow props or elevation.

**Why:** boxShadow is the modern, cross-platform approach and supports more features including inset shadows.

```tsx
// BAD - Legacy shadow props
<View style={{
  shadowColor: '#000',
  shadowOffset: { width: 0, height: 1 },
  shadowOpacity: 0.2,
  elevation: 2
}} />

// GOOD - Modern boxShadow
<View style={{ boxShadow: '0 1px 2px rgba(0, 0, 0, 0.05)' }} />
```

### 7.4 Continuous Border Curves

**What:** Use borderCurve: 'continuous' for rounded corners (except capsule shapes).

**Why:** Continuous border curves provide smoother, more natural-looking corners following Apple's design guidelines.

```tsx
// Standard rounded corners
<View style={{ borderRadius: 12 }} />

// Better - Continuous curves
<View style={{ borderRadius: 12, borderCurve: 'continuous' }} />
```

### 7.5 Inline Styles Preferred

**What:** Use inline styles for one-time use, StyleSheet.create only when reusing styles.

```tsx
// GOOD for single use
<View style={{ flex: 1, padding: 16 }} />;

// GOOD for reusable styles
const styles = StyleSheet.create({
  container: { flex: 1, padding: 16 },
});
// Used in multiple places
```

### 7.6 Flexbox Over Absolute Positioning

**What:** Use flexbox for layouts, prefer gap over margin/padding for spacing.

```tsx
// BAD
<View>
  <View style={{ marginBottom: 16 }}>...</View>
  <View style={{ marginBottom: 16 }}>...</View>
</View>

// GOOD
<View style={{ gap: 16 }}>
  <View>...</View>
  <View>...</View>
</View>
```

### 7.7 Tabular Numbers for Counters

**What:** Use fontVariant: 'tabular-nums' for numbers that change frequently.

**Why:** Prevents layout shifts when numbers change.

```tsx
<Text style={{ fontVariant: ["tabular-nums"] }}>{count}</Text>
```

### 7.8 Selectable Text for Important Data

**What:** Add the selectable prop to Text elements displaying important data or error messages.

```tsx
<Text selectable>{errorMessage}</Text>
<Text selectable>{transactionId}</Text>
```

---

## 8. Components & Architecture

### 8.1 Platform Detection

**What:** Use process.env.EXPO_OS instead of Platform.OS.

**Why:** EXPO_OS is optimized for Expo apps and provides better tree-shaking capabilities.

```tsx
// BAD
import { Platform } from 'react-native';
if (Platform.OS === 'ios') { ... }

// GOOD
if (process.env.EXPO_OS === 'ios') { ... }
```

### 8.2 Use expo-image

**What:** Use expo-image Image component instead of React Native's Image.

**Why:** expo-image provides better caching, placeholder support, and performance optimizations.

```tsx
// BAD
import { Image } from "react-native";

// GOOD
import { Image } from "expo-image";
```

### 8.3 Use Modern Context API

**What:** Use React.use() instead of React.useContext() (React 19+).

**Why:** React.use() is the modern API that works with Suspense and provides better integration with concurrent features.

```tsx
// Legacy
const theme = React.useContext(ThemeContext);

// Modern (React 19+)
const theme = React.use(ThemeContext);
```

### 8.4 Never Use Deprecated Modules

**What:** Never use deprecated React Native modules.

| Deprecated                     | Use Instead                               |
| ------------------------------ | ----------------------------------------- |
| Picker from react-native       | @react-native-picker/picker               |
| WebView from react-native      | react-native-webview                      |
| SafeAreaView from react-native | react-native-safe-area-context            |
| AsyncStorage from react-native | @react-native-async-storage/async-storage |

---

## 9. Animations

### 9.1 Use Reanimated

**What:** Use Reanimated v4 for animations, not React Native's built-in Animated API.

**Why:** Reanimated runs on the UI thread, providing 60 FPS animations without blocking JS.

```tsx
// BAD - Blocks JS thread
import { Animated } from "react-native";

// GOOD - UI thread animations
import Animated, { FadeIn, FadeOut } from "react-native-reanimated";

<Animated.View entering={FadeIn} exiting={FadeOut}>
  ...
</Animated.View>;
```

### 9.2 Add Enter/Exit Animations

**What:** Add entering and exiting animations for state changes.

```tsx
<Animated.View
  entering={FadeInDown.duration(300)}
  exiting={FadeOut.duration(200)}
  layout={LinearTransition}
>
  {content}
</Animated.View>
```

### 9.3 Keep Animations Under 300ms

**What:** Keep animations under 300ms for responsive feel.

### 9.4 Prefer Transforms Over Layout

**What:** Avoid animating layout properties (width, height) when possible - prefer transforms.

```tsx
// BAD - Animates layout
const style = useAnimatedStyle(() => ({
  width: width.value,
}));

// GOOD - Animates transform
const style = useAnimatedStyle(() => ({
  transform: [{ scale: scale.value }],
}));
```

### 9.5 Use Haptics on iOS

**What:** Use expo-haptics conditionally on iOS for delightful interactions.

```tsx
import * as Haptics from "expo-haptics";

const handlePress = () => {
  if (process.env.EXPO_OS === "ios") {
    Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
  }
  // ... rest of handler
};
```

---

## 10. Storage

### 10.1 Storage Decision Matrix

| Use Case                                 | Solution                            |
| ---------------------------------------- | ----------------------------------- |
| Simple key-value (settings, preferences) | localStorage polyfill (expo-sqlite) |
| Large datasets, complex queries          | Full expo-sqlite                    |
| Sensitive data (tokens, passwords)       | expo-secure-store                   |

### 10.2 Never Use AsyncStorage Directly

**What:** Use expo-sqlite localStorage polyfill instead of AsyncStorage for key-value storage.

```tsx
// BAD
import AsyncStorage from "@react-native-async-storage/async-storage";

// GOOD
import "expo-sqlite/localStorage/install";

localStorage.setItem("key", "value");
localStorage.getItem("key");
```

---

## 11. Code Style & Conventions

### 11.1 File Naming

**What:** Use kebab-case for all file names.

```
// BAD
CommentCard.tsx
commentCard.tsx
comment_card.tsx

// GOOD
comment-card.tsx
user-profile.tsx
```

### 11.2 Path Aliases

**What:** Configure tsconfig.json with path aliases and prefer aliases over relative imports.

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

```tsx
// BAD
import { Button } from "../../../components/Button";

// GOOD
import { Button } from "@/components/Button";
```

### 11.3 No Intrinsic Elements

**What:** Never use intrinsic HTML elements like 'img' or 'div' unless in a webview or Expo DOM component.

```tsx
// BAD
<img src={url} />
<div>...</div>

// GOOD
import { Image } from 'expo-image';
import { View } from 'react-native';

<Image source={{ uri: url }} />
<View>...</View>
```

### 11.4 Escape Strings Properly

**What:** Be cautious of unterminated strings. Ensure nested backticks are escaped correctly.

---

## 12. Library Preferences

### Preferred Libraries

| Category   | Use                            | Avoid                |
| ---------- | ------------------------------ | -------------------- |
| Images     | expo-image                     | react-native Image   |
| Icons      | expo-symbols                   | @expo/vector-icons   |
| Audio      | expo-audio                     | expo-av              |
| Video      | expo-video                     | expo-av              |
| Storage    | expo-sqlite, expo-secure-store | AsyncStorage         |
| Networking | fetch, React Query             | axios                |
| Animations | react-native-reanimated        | Animated from RN     |
| Lists      | @shopify/flash-list            | ScrollView + map     |
| Safe Area  | react-native-safe-area-context | SafeAreaView from RN |

---

## Quick Reference Checklist

### Before Opening a PR

- [ ] Lists with >20 items use FlashList/FlatList
- [ ] No barrel imports (import directly from source)
- [ ] All useEffect have cleanup functions for subscriptions/timers
- [ ] Tokens stored in SecureStore, not AsyncStorage
- [ ] HTTP responses check .ok before parsing
- [ ] No deprecated modules (Picker, WebView, SafeAreaView from RN)
- [ ] File names use kebab-case
- [ ] No co-located files in app/ directory
- [ ] Using expo-image, not RN Image
- [ ] Animations use Reanimated, not Animated
- [ ] Safe areas handled with contentInsetAdjustmentBehavior

---

_Sources: [expo/skills](https://github.com/expo/skills), [callstackincubator/agent-skills](https://github.com/callstackincubator/agent-skills)_
