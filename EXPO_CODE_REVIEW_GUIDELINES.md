# React Native + Expo Code Guidelines

> Code review standards for React Native + Expo projects.
> Based on [expo/skills](https://github.com/expo/skills), [callstackincubator/agent-skills](https://github.com/callstackincubator/agent-skills), and Callstack's "Ultimate Guide to React Native Optimization".

**Target versions:** Expo SDK 54 | React Native 0.81 | React 19.1.0

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
11. [Accessibility](#11-accessibility)
12. [Code Style & Conventions](#12-code-style--conventions)
13. [Library Preferences](#13-library-preferences)

---

## 1. Performance - Lists & Rendering

### 1.1 Use FlashList for Lists

**What:** Use FlashList instead of ScrollView with .map() for lists with more than 10-20 items.

**Why:** ScrollView renders ALL items at once, causing freezes, FPS drops to 0, and high memory usage. FlashList only renders visible items and is more performant than FlatList.

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
| > 20 items | FlashList |
| Complex item layouts | FlashList with `getItemType` |

### 1.2 Proper keyExtractor

**What:** Use unique item IDs for keyExtractor, not array indices.

**Why:** Using indices causes incorrect recycling when items are added/removed/reordered, leading to visual bugs and performance issues.

```tsx
// BAD - Index-based keys cause bugs
keyExtractor={(item, index) => index.toString()}

// GOOD - Unique ID-based keys
keyExtractor={(item) => item.id.toString()}
```

### 1.3 React Compiler (Expo SDK 52+)

**What:** Enable React Compiler v1.0 for automatic memoization instead of manual memo/useMemo/useCallback.

**Why:** React Compiler is now production-ready (v1.0). It eliminates boilerplate and ensures consistent optimization across the codebase. Manual memoization becomes optional when enabled.

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

**What:** Import specific functions from libraries instead of the entire library, OR enable tree shaking.

**Why:** Reduces bundle size significantly, especially with large libraries like lodash.

```tsx
// BAD - Imports entire lodash library (~70kb)
import { debounce } from "lodash";

// GOOD - Direct import (~2kb)
import debounce from "lodash/debounce";
```

**Note:** Modern libraries like `date-fns` support tree shaking out of the box. With tree shaking enabled (see 2.3), named imports work efficiently:

```tsx
// OK with tree shaking enabled
import { format, addDays } from "date-fns";
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

### 4.1 Use React Query for Data Management

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

### 4.2 Environment Variables

**What:** Use `EXPO_PUBLIC_` prefix only for non-sensitive client-side configuration.

**Why:** Variables with `EXPO_PUBLIC_` prefix are embedded in the JavaScript bundle and visible to anyone who downloads the app. Never use this prefix for secrets or sensitive credentials.

```bash
# .env

# OK - Public configuration (safe to expose)
EXPO_PUBLIC_API_URL=https://api.example.com
EXPO_PUBLIC_ENVIRONMENT=production
EXPO_PUBLIC_SENTRY_DSN=https://abc123@sentry.io/123
EXPO_PUBLIC_POSTHOG_KEY=phc_abc123

# NEVER use EXPO_PUBLIC_ for:
# - Secret keys (STRIPE_SECRET_KEY, OPENAI_API_KEY)
# - Auth secrets (JWT_SECRET, AUTH0_CLIENT_SECRET)
# - Database credentials (DATABASE_URL, SUPABASE_SERVICE_KEY)
# - Private API keys that grant write/admin access
```

**Note:** Some services (Firebase, Sentry, PostHog) have client-side keys that are safe to expose—security is enforced server-side. Check each service's documentation.

---

## 5. Authentication & Security

### 5.1 Use SecureStore for Tokens

**What:** Use expo-secure-store for storing authentication tokens, not MMKV or AsyncStorage.

**Why:** SecureStore uses the device's secure keychain/keystore with hardware-backed encryption. MMKV encryption is software-based and less secure for sensitive credentials.

```tsx
// BAD - Not secure enough for tokens
storage.set("token", token);

// GOOD - Hardware-backed encryption
import * as SecureStore from "expo-secure-store";
await SecureStore.setItemAsync("token", token);
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

### 6.2 Screen Content Organization

**What:** Route files in `/app` should only import and export screens. The actual screen content lives in `/src/screens/`.

**Why:** Keeps the `/app` directory focused on routing structure. Screen logic, styles, and related components stay organized in `/src/screens/`, making them easier to test and maintain.

```
// Route file (app/user-profile.tsx)
export { default } from '@/screens/user-profile';

// Screen content (src/screens/user-profile/index.tsx)
export default function UserProfileScreen() {
  return (
    // Screen implementation
  );
}
```

```
// Project structure
app/
  _layout.tsx
  index.tsx              <- exports from @/screens/home
  user-profile.tsx       <- exports from @/screens/user-profile
src/
  screens/
    home/
      index.tsx          <- actual screen content
    user-profile/
      index.tsx          <- actual screen content
      components/        <- screen-specific components (optional)
```

### 6.3 Always Use \_layout.tsx for Stacks

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

### 6.4 Use Link with asChild for Navigation

**What:** Use Link from expo-router with asChild to wrap Pressable components.

```tsx
import { Link } from "expo-router";

<Link href="/profile" asChild>
  <Pressable>
    <Text>Profile</Text>
  </Pressable>
</Link>;
```

### 6.5 Use useRouter for Programmatic Navigation

**What:** Use the useRouter hook for programmatic navigation inside components.

**Why:** The hook is reactive to navigation context and handles edge cases (modals, tabs) correctly.

```tsx
import { useRouter } from 'expo-router';

function MyComponent() {
  const router = useRouter();

  const handleSubmit = async () => {
    await saveData();
    router.push('/success');
  };

  const handleCancel = () => {
    router.back();
  };

  // With params
  const goToUser = (id: string) => {
    router.push({
      pathname: '/user/[id]',
      params: { id }
    });
  };

  return (/* ... */);
}
```

**Note:** Only use direct `import { router } from 'expo-router'` outside of React components (utils, services).

### 6.6 Ensure Root Route Exists

**What:** Always have a route that matches "/" so the app is never blank.

---

## 7. Styling & UI

### 7.1 Safe Area Handling

**What:** Use `react-native-safe-area-context` for safe area handling. The approach depends on your screen type.

**Why:** SafeAreaView from react-native is deprecated in RN 0.81. The context-based approach provides more flexibility and works correctly with animations.

**Setup (required in root layout):**

```tsx
// app/_layout.tsx
import { SafeAreaProvider } from "react-native-safe-area-context";

export default function RootLayout() {
  return (
    <SafeAreaProvider>
      <Stack />
    </SafeAreaProvider>
  );
}
```

**Decision Matrix:**

| Scenario                           | Solution                                                    |
| ---------------------------------- | ----------------------------------------------------------- |
| Scrollable content                 | `ScrollView` + `contentInsetAdjustmentBehavior="automatic"` |
| Static screen (no scroll)          | `useSafeAreaInsets()` hook                                  |
| Fixed bottom element (FAB, button) | `useSafeAreaInsets()` for bottom padding                    |
| ❌ Avoid                           | `SafeAreaView` from react-native (deprecated)               |

```tsx
// CASE 1: Scrollable content
<ScrollView contentInsetAdjustmentBehavior="automatic">
  {/* long content */}
</ScrollView>;

// CASE 2: Static screen without scroll
import { useSafeAreaInsets } from "react-native-safe-area-context";

function StaticScreen() {
  const insets = useSafeAreaInsets();

  return (
    <View
      style={{
        flex: 1,
        paddingTop: insets.top,
        paddingBottom: insets.bottom,
      }}
    >
      {/* static content */}
    </View>
  );
}

// CASE 3: Screen with fixed bottom button
function ScreenWithBottomButton() {
  const insets = useSafeAreaInsets();

  return (
    <View style={{ flex: 1 }}>
      <ScrollView contentInsetAdjustmentBehavior="automatic">
        {/* content */}
      </ScrollView>

      <View style={{ paddingBottom: insets.bottom }}>
        <Button title="Submit" />
      </View>
    </View>
  );
}
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

### 7.3 Use StyleSheet.create for Static Styles

**What:** Define static styles in StyleSheet.create. Inline styles are acceptable for dynamic values computed at runtime.

**Why:** Inline styles are not bad per se, but extracting static styles to StyleSheet.create improves consistency and keeps the codebase clean across the entire app. Dynamic values (from props, hooks, or calculations) are fine inline since they can't be defined statically.

```tsx
// BAD - Static values inline
<View style={{ flex: 1, padding: 16, backgroundColor: '#fff' }}>
  <Text style={{ fontSize: 18, marginTop: 20 }}>Title</Text>
</View>

// GOOD - Static styles extracted
<View style={styles.container}>
  <Text style={styles.title}>Title</Text>
</View>

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16, backgroundColor: '#fff' },
  title: { fontSize: 18, marginTop: 20 },
});

// GOOD - Dynamic values inline are acceptable
const insets = useSafeAreaInsets();

<View style={[styles.container, { paddingTop: insets.top }]}>
  <Text style={{ opacity: isVisible ? 1 : 0 }}>Title</Text>
</View>
```

### 7.4 Flexbox Over Absolute Positioning

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

### 7.5 Selectable Text for Important Data

**What:** Add the selectable prop to Text elements displaying important data or error messages.

```tsx
<Text selectable>{errorMessage}</Text>
<Text selectable>{transactionId}</Text>
```

---

## 8. Components & Architecture

### 8.1 Use expo-image

**What:** Use expo-image Image component instead of React Native's Image.

**Why:** expo-image provides better caching, placeholder support, and performance optimizations.

```tsx
// BAD
import { Image } from "react-native";

// GOOD
import { Image } from "expo-image";
```

### 8.2 Never Use Deprecated Modules

**What:** Never use deprecated React Native modules.

| Deprecated                     | Use Instead                            |
| ------------------------------ | -------------------------------------- |
| Picker from react-native       | @react-native-picker/picker            |
| WebView from react-native      | react-native-webview                   |
| SafeAreaView from react-native | react-native-safe-area-context         |
| AsyncStorage from react-native | react-native-mmkv or expo-secure-store |

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

| Use Case                           | Solution            |
| ---------------------------------- | ------------------- |
| Settings, preferences, cache       | `react-native-mmkv` |
| Sensitive data (tokens, passwords) | `expo-secure-store` |

### 10.2 Use MMKV for Local Storage

**What:** Use react-native-mmkv for general key-value storage instead of AsyncStorage.

**Why:** MMKV is ~30x faster than AsyncStorage, has a synchronous API, supports encryption, and is battle-tested by WeChat (billions of users).

```tsx
// BAD - Slow, async API
import AsyncStorage from "@react-native-async-storage/async-storage";
await AsyncStorage.setItem("theme", "dark");
const theme = await AsyncStorage.getItem("theme");

// GOOD - Fast, sync API
import { MMKV } from "react-native-mmkv";

const storage = new MMKV();

storage.set("theme", "dark");
const theme = storage.getString("theme");

// Supports native types
storage.set("count", 42);
storage.set("enabled", true);
storage.set("user", JSON.stringify(user));
```

### 10.3 MMKV with Encryption (Non-sensitive Data)

**What:** Use MMKV's built-in encryption for data that needs protection but isn't highly sensitive.

**Why:** Software-based encryption is suitable for preferences and cached data, but use SecureStore for tokens and passwords.

```tsx
const encryptedStorage = new MMKV({
  id: "encrypted-storage",
  encryptionKey: "your-encryption-key",
});
```

### 10.4 Typed Storage Helper

**What:** Create a typed wrapper for type-safe storage access.

```tsx
import { MMKV } from "react-native-mmkv";

const storage = new MMKV();

export const appStorage = {
  getTheme: () => storage.getString("theme") as "light" | "dark" | undefined,
  setTheme: (theme: "light" | "dark") => storage.set("theme", theme),

  getOnboardingComplete: () => storage.getBoolean("onboarding") ?? false,
  setOnboardingComplete: (value: boolean) => storage.set("onboarding", value),

  getUser: () => {
    const json = storage.getString("user");
    return json ? (JSON.parse(json) as User) : null;
  },
  setUser: (user: User) => storage.set("user", JSON.stringify(user)),
  clearUser: () => storage.delete("user"),
};
```

---

## 11. Accessibility

### 11.1 Always Add accessibilityLabel to Interactive Elements

**What:** Add accessibilityLabel to all Pressable, TouchableOpacity, Button, and icon-only elements.

**Why:** Screen readers (VoiceOver, TalkBack) read this label aloud. Without it, users hear nothing or unhelpful text like "button".

```tsx
// BAD - Screen reader says "button" or nothing
<Pressable onPress={handleClose}>
  <CloseIcon />
</Pressable>

// GOOD - Screen reader says "Close modal"
<Pressable
  onPress={handleClose}
  accessibilityLabel="Close modal"
>
  <CloseIcon />
</Pressable>

// BAD - Screen reader reads "heart"
<Pressable onPress={handleLike}>
  <HeartIcon />
</Pressable>

// GOOD - Describes the action
<Pressable
  onPress={handleLike}
  accessibilityLabel={isLiked ? "Remove from favorites" : "Add to favorites"}
>
  <HeartIcon filled={isLiked} />
</Pressable>
```

### 11.2 Use accessibilityRole

**What:** Specify the semantic role of interactive elements.

**Why:** Tells assistive technology what type of element it is, enabling proper interaction patterns.

```tsx
// Common roles
<Pressable accessibilityRole="button" />
<Pressable accessibilityRole="link" />
<Switch accessibilityRole="switch" />
<TextInput accessibilityRole="search" />
<Image accessibilityRole="image" accessibilityLabel="Product photo" />

// Headers for screen structure
<Text accessibilityRole="header">Settings</Text>
```

**Common Roles:**

| Role       | Use Case                              |
| ---------- | ------------------------------------- |
| `button`   | Pressable actions                     |
| `link`     | Navigation to other screens/URLs      |
| `header`   | Section titles (helps navigation)     |
| `image`    | Images (pair with accessibilityLabel) |
| `search`   | Search input fields                   |
| `switch`   | Toggle controls                       |
| `checkbox` | Multi-select options                  |
| `radio`    | Single-select options                 |

### 11.3 Use accessibilityState for Dynamic States

**What:** Communicate element states to assistive technology.

**Why:** Screen readers announce state changes, helping users understand the current UI state.

```tsx
// Toggle button
<Pressable
  accessibilityRole="button"
  accessibilityLabel="Dark mode"
  accessibilityState={{ checked: isDarkMode }}
  onPress={toggleDarkMode}
>
  <Text>{isDarkMode ? "On" : "Off"}</Text>
</Pressable>

// Disabled button
<Pressable
  accessibilityRole="button"
  accessibilityLabel="Submit form"
  accessibilityState={{ disabled: !isFormValid }}
  disabled={!isFormValid}
  onPress={handleSubmit}
>
  <Text>Submit</Text>
</Pressable>

// Selected item in a list
<Pressable
  accessibilityRole="button"
  accessibilityLabel={item.name}
  accessibilityState={{ selected: item.id === selectedId }}
  onPress={() => select(item.id)}
>
  <Text>{item.name}</Text>
</Pressable>
```

### 11.4 Minimum Touch Target Size

**What:** Ensure all interactive elements are at least 44x44 points.

**Why:** Small touch targets are difficult for users with motor impairments. This is also an Apple/Google requirement.

```tsx
// BAD - Icon too small to tap reliably
<Pressable onPress={handlePress}>
  <Icon size={24} />
</Pressable>

// GOOD - Adequate touch target with hitSlop
<Pressable
  onPress={handlePress}
  hitSlop={{ top: 10, bottom: 10, left: 10, right: 10 }}
>
  <Icon size={24} />
</Pressable>

// GOOD - Minimum size enforced
<Pressable
  onPress={handlePress}
  style={{ minWidth: 44, minHeight: 44, justifyContent: 'center', alignItems: 'center' }}
>
  <Icon size={24} />
</Pressable>
```

### 11.5 Respect Reduced Motion

**What:** Check user's motion preferences and reduce/disable animations accordingly.

**Why:** Some users experience motion sickness, vestibular disorders, or simply prefer reduced motion.

```tsx
import { useReducedMotion } from "react-native-reanimated";

function AnimatedComponent() {
  const reducedMotion = useReducedMotion();

  return (
    <Animated.View
      entering={reducedMotion ? FadeIn : FadeInDown.springify()}
      exiting={reducedMotion ? FadeOut : FadeOutUp}
    >
      {content}
    </Animated.View>
  );
}

// For custom animations
function CustomAnimation() {
  const reducedMotion = useReducedMotion();

  const animatedStyle = useAnimatedStyle(() => ({
    transform: reducedMotion ? [] : [{ translateY: withSpring(offset.value) }],
    opacity: withTiming(opacity.value),
  }));

  return <Animated.View style={animatedStyle} />;
}
```

### 11.6 Group Related Elements

**What:** Use the accessible prop to group related elements as a single focusable unit.

**Why:** Reduces the number of swipes needed to navigate and provides context.

```tsx
// BAD - Each element is separately focusable (3 swipes)
<View>
  <Image source={user.avatar} />
  <Text>{user.name}</Text>
  <Text>{user.role}</Text>
</View>

// GOOD - Single focusable unit (1 swipe)
<View
  accessible={true}
  accessibilityLabel={`${user.name}, ${user.role}`}
>
  <Image source={user.avatar} />
  <Text>{user.name}</Text>
  <Text>{user.role}</Text>
</View>

// Card example
<Pressable
  accessible={true}
  accessibilityRole="button"
  accessibilityLabel={`${product.name}, ${product.price}. Double tap to view details`}
  onPress={() => navigateToProduct(product.id)}
>
  <Image source={product.image} />
  <Text>{product.name}</Text>
  <Text>{product.price}</Text>
</Pressable>
```

### 11.7 Announce Dynamic Content

**What:** Use accessibilityLiveRegion to announce content changes.

**Why:** Screen readers don't automatically announce dynamic updates. Users need to know when content changes.

```tsx
// Toast/Snackbar notifications
<Animated.View
  accessibilityLiveRegion="polite"
  accessibilityRole="alert"
  entering={SlideInUp}
>
  <Text>{message}</Text>
</Animated.View>

// Loading states
<View accessibilityLiveRegion="polite">
  {isLoading ? (
    <Text accessibilityLabel="Loading content">Loading...</Text>
  ) : (
    <Text>{content}</Text>
  )}
</View>

// Error messages
<Text
  accessibilityLiveRegion="assertive"
  accessibilityRole="alert"
  style={styles.error}
>
  {errorMessage}
</Text>
```

**Live Region Types:**

| Type        | Use Case                                           |
| ----------- | -------------------------------------------------- |
| `polite`    | Non-urgent updates (loading complete, new content) |
| `assertive` | Urgent updates (errors, time-sensitive alerts)     |
| `none`      | Don't announce (default)                           |

### 11.8 Form Accessibility

**What:** Associate labels with inputs and provide error feedback.

```tsx
// Text input with label
<View>
  <Text nativeID="emailLabel">Email address</Text>
  <TextInput
    accessibilityLabelledBy="emailLabel"
    accessibilityRole="none"
    keyboardType="email-address"
    autoComplete="email"
    textContentType="emailAddress"
  />
</View>

// Input with error
<View>
  <Text nativeID="passwordLabel">Password</Text>
  <TextInput
    accessibilityLabelledBy="passwordLabel"
    accessibilityState={{ invalid: !!error }}
    accessibilityHint={error || "Enter your password"}
  />
  {error && (
    <Text
      accessibilityLiveRegion="polite"
      style={styles.error}
    >
      {error}
    </Text>
  )}
</View>
```

---

## 12. Code Style & Conventions

### 12.1 File Naming

**What:** Use different naming conventions based on location:
- **Routes & Screens** (in `/app` and `/src/screens`): kebab-case → `user-profile.tsx`, `user-profile/`
- **Components** (in `/src/components`): PascalCase → `UserProfile.tsx`

**Why:** Routes map to URLs which use kebab-case. Screen folders mirror the route structure for consistency. Reusable components use PascalCase to match the exported component name (React convention).

```
// Routes (app/) - kebab-case
app/
  user-profile.tsx      ✓ kebab-case
  settings.tsx          ✓ kebab-case
  _layout.tsx           ✓ kebab-case

// Screens (src/screens/) - kebab-case folders, mirrors /app structure
src/screens/
  user-profile/
    index.tsx           ✓ screen content
    components/         ✓ screen-specific components
  settings/
    index.tsx

// Reusable Components (src/components/) - PascalCase
src/components/
  UserAvatar.tsx        ✓ PascalCase
  CommentCard.tsx       ✓ PascalCase
  Button.tsx            ✓ PascalCase
```

### 12.2 Path Aliases

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

### 12.3 Escape Strings Properly

**What:** Be cautious of unterminated strings. Ensure nested backticks are escaped correctly.

---

## 13. Library Preferences

### Preferred Libraries

| Category               | Use                            | Avoid                |
| ---------------------- | ------------------------------ | -------------------- |
| Images                 | expo-image                     | react-native Image   |
| Icons (iOS native)     | expo-symbols                   | -                    |
| Icons (cross-platform) | @expo/vector-icons             | -                    |
| Audio                  | expo-audio                     | expo-av              |
| Video                  | expo-video                     | expo-av              |
| Storage (general)      | react-native-mmkv              | AsyncStorage         |
| Storage (sensitive)    | expo-secure-store              | MMKV, AsyncStorage   |
| Networking             | React Query                    | -                    |
| Animations             | react-native-reanimated        | Animated from RN     |
| Lists                  | @shopify/flash-list            | FlatList, ScrollView + map |
| Safe Area              | react-native-safe-area-context | SafeAreaView from RN |

**Note on Icons:** `expo-symbols` provides native SF Symbols on iOS only. For cross-platform apps, use `@expo/vector-icons` or combine both with platform checks.

---

## Quick Reference Checklist

### Before Opening a PR

- [ ] Lists with >20 items use FlashList
- [ ] No barrel imports (import directly from source)
- [ ] All useEffect have cleanup functions for subscriptions/timers
- [ ] Tokens stored in SecureStore, not MMKV or AsyncStorage
- [ ] HTTP responses check .ok before parsing
- [ ] No deprecated modules (Picker, WebView, SafeAreaView from RN)
- [ ] File names use kebab-case
- [ ] No co-located files in app/ directory
- [ ] Using expo-image, not RN Image
- [ ] Animations use Reanimated, not Animated
- [ ] SafeAreaProvider in root layout
- [ ] Safe areas handled with useSafeAreaInsets or contentInsetAdjustmentBehavior
- [ ] Using MMKV for local storage, not AsyncStorage
- [ ] useRouter hook for programmatic navigation
- [ ] Interactive elements have accessibilityLabel
- [ ] Touch targets are at least 44x44 points
- [ ] Animations respect useReducedMotion

---

_Sources: [Expo Docs](https://docs.expo.dev/), [Expo YouTube](https://www.youtube.com/@ExpoDevelopers), [Code with Beto](https://www.youtube.com/@codewithbeto), [expo/skills](https://github.com/expo/skills), [callstackincubator/agent-skills](https://github.com/callstackincubator/agent-skills)_
