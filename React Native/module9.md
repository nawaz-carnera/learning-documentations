# Module 9 — Authentication & Security

> Authentication in React Native involves more than just an API call — secure token storage, refresh flows, biometric gating, and proper session management all live on the native side. This module covers everything from a basic login UI to enterprise SSO.

---

## Table of Contents

1. [Auth Flows](#1-auth-flows)
   - 1.1 [Login / Signup UI](#11-login--signup-ui)
   - 1.2 [Forgot Password Flow](#12-forgot-password-flow)
   - 1.3 [OTP / Email Verification](#13-otp--email-verification)
   - 1.4 [Protected Routes](#14-protected-routes)
   - 1.5 [Auth State Persistence](#15-auth-state-persistence)
   - 1.6 [Auto-Login](#16-auto-login)
   - 1.7 [Logout Flow](#17-logout-flow)

2. [Token-Based Auth](#2-token-based-auth)
   - 2.1 [JWT Basics](#21-jwt-basics)
   - 2.2 [Access and Refresh Tokens](#22-access-and-refresh-tokens)
   - 2.3 [Token Storage (SecureStore / Keychain)](#23-token-storage-securestore--keychain)
   - 2.4 [Token Refresh Interceptor](#24-token-refresh-interceptor)
   - 2.5 [Session Expiry Handling](#25-session-expiry-handling)

3. [Firebase Authentication](#3-firebase-authentication)
   - 3.1 [Firebase Project Setup](#31-firebase-project-setup)
   - 3.2 [@react-native-firebase/auth](#32-react-native-firebaseauth)
   - 3.3 [Email/Password Auth](#33-emailpassword-auth)
   - 3.4 [Google Sign-In](#34-google-sign-in)
   - 3.5 [Apple Sign-In (required for iOS)](#35-apple-sign-in-required-for-ios)
   - 3.6 [Phone Authentication](#36-phone-authentication)
   - 3.7 [Anonymous Auth](#37-anonymous-auth)

4. [OAuth / Social Login](#4-oauth--social-login)
   - 4.1 [expo-auth-session](#41-expo-auth-session)
   - 4.2 [Google OAuth](#42-google-oauth)
   - 4.3 [Facebook Login](#43-facebook-login)
   - 4.4 [Apple Sign-In](#44-apple-sign-in)
   - 4.5 [Microsoft / Azure AD](#45-microsoft--azure-ad)
   - 4.6 [PKCE Flow](#46-pkce-flow)

5. [Enterprise Auth](#5-enterprise-auth)
   - 5.1 [Azure AD / Entra ID](#51-azure-ad--entra-id)
   - 5.2 [SSO Integration](#52-sso-integration)
   - 5.3 [SAML Basics](#53-saml-basics)
   - 5.4 [MSAL Library](#54-msal-library)

6. [Security Best Practices](#6-security-best-practices)
   - 6.1 [Secure Token Storage](#61-secure-token-storage)
   - 6.2 [Certificate Pinning](#62-certificate-pinning)
   - 6.3 [Jailbreak / Root Detection](#63-jailbreak--root-detection)
   - 6.4 [Biometric Gating](#64-biometric-gating)
   - 6.5 [Session Timeout](#65-session-timeout)

---

## 1. Auth Flows

### 1.1 Login / Signup UI

A complete login screen using RHF + Zod from Module 7, wired to an auth store.

```tsx
// app/(auth)/login.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { View, Text, Pressable, StyleSheet, Alert } from 'react-native';
import { router, Link } from 'expo-router';
import { FormInput } from '@/components/FormInput';
import { useAuthStore } from '@/store/useAuthStore';

const loginSchema = z.object({
  email: z.string().min(1, 'Required').email('Invalid email'),
  password: z.string().min(1, 'Required').min(8, 'Min 8 characters'),
});

type LoginForm = z.infer<typeof loginSchema>;

export default function LoginScreen() {
  const login = useAuthStore(s => s.login);

  const {
    control,
    handleSubmit,
    setError,
    formState: { isSubmitting, isValid },
  } = useForm<LoginForm>({
    resolver: zodResolver(loginSchema),
    defaultValues: { email: '', password: '' },
    mode: 'onBlur',
  });

  const onSubmit = async (data: LoginForm) => {
    try {
      await login(data.email, data.password);
      router.replace('/(app)/(tabs)/');
    } catch (error: any) {
      const status = error?.response?.status;
      if (status === 401) {
        setError('password', { message: 'Incorrect email or password' });
      } else if (status === 429) {
        Alert.alert('Too many attempts', 'Please wait a moment before trying again.');
      } else {
        setError('root', { message: 'Something went wrong. Please try again.' });
      }
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Welcome back</Text>
      <Text style={styles.subtitle}>Sign in to your account</Text>

      <FormInput control={control} name="email" label="Email"
        keyboardType="email-address" autoCapitalize="none" autoCorrect={false} />

      <FormInput control={control} name="password" label="Password"
        secureTextEntry autoCapitalize="none" />

      <Link href="/(auth)/forgot-password" style={styles.forgotLink}>
        Forgot password?
      </Link>

      <Pressable
        onPress={handleSubmit(onSubmit)}
        disabled={!isValid || isSubmitting}
        style={[styles.button, (!isValid || isSubmitting) && styles.buttonDisabled]}
      >
        <Text style={styles.buttonText}>
          {isSubmitting ? 'Signing in...' : 'Sign In'}
        </Text>
      </Pressable>

      <View style={styles.footer}>
        <Text style={styles.footerText}>Don't have an account? </Text>
        <Link href="/(auth)/signup" style={styles.link}>Sign up</Link>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 24, justifyContent: 'center' },
  title: { fontSize: 28, fontWeight: '700', marginBottom: 4 },
  subtitle: { fontSize: 16, color: '#666', marginBottom: 32 },
  forgotLink: { alignSelf: 'flex-end', color: '#007AFF', marginTop: 4, marginBottom: 24 },
  button: { backgroundColor: '#007AFF', height: 52, borderRadius: 12, justifyContent: 'center', alignItems: 'center' },
  buttonDisabled: { opacity: 0.5 },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
  footer: { flexDirection: 'row', justifyContent: 'center', marginTop: 24 },
  footerText: { color: '#666' },
  link: { color: '#007AFF', fontWeight: '600' },
});
```

---

### 1.2 Forgot Password Flow

Three-step flow: enter email → receive reset link → confirm.

```tsx
// app/(auth)/forgot-password.tsx
import { useState } from 'react';
import { View, Text, Pressable, StyleSheet } from 'react-native';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { FormInput } from '@/components/FormInput';
import { router } from 'expo-router';

const schema = z.object({
  email: z.string().min(1, 'Required').email('Invalid email'),
});

type ForgotForm = z.infer<typeof schema>;

export default function ForgotPasswordScreen() {
  const [sent, setSent] = useState(false);
  const [email, setEmail] = useState('');

  const { control, handleSubmit, formState: { isSubmitting } } = useForm<ForgotForm>({
    resolver: zodResolver(schema),
  });

  const onSubmit = async (data: ForgotForm) => {
    await api.auth.sendPasswordReset(data.email);
    setEmail(data.email);
    setSent(true);
  };

  if (sent) {
    return (
      <View style={styles.container}>
        <Text style={styles.icon}>📬</Text>
        <Text style={styles.title}>Check your inbox</Text>
        <Text style={styles.body}>
          We sent a password reset link to{'\n'}
          <Text style={styles.emailText}>{email}</Text>
        </Text>
        <Pressable onPress={() => router.back()} style={styles.button}>
          <Text style={styles.buttonText}>Back to Sign In</Text>
        </Pressable>
        <Pressable onPress={() => onSubmit({ email })} style={styles.resendLink}>
          <Text style={styles.linkText}>Didn't receive it? Resend</Text>
        </Pressable>
      </View>
    );
  }

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Reset password</Text>
      <Text style={styles.body}>
        Enter your email and we'll send you a reset link.
      </Text>
      <FormInput control={control} name="email" label="Email"
        keyboardType="email-address" autoCapitalize="none" />
      <Pressable
        onPress={handleSubmit(onSubmit)}
        disabled={isSubmitting}
        style={styles.button}
      >
        <Text style={styles.buttonText}>
          {isSubmitting ? 'Sending...' : 'Send Reset Link'}
        </Text>
      </Pressable>
    </View>
  );
}
```

---

### 1.3 OTP / Email Verification

Two use cases: phone number OTP and email verification after signup.

```tsx
// app/(auth)/verify-otp.tsx
import { useState, useRef, useEffect } from 'react';
import { View, Text, Pressable, StyleSheet, Alert } from 'react-native';
import { router, useLocalSearchParams } from 'expo-router';
import { OTPInput } from '@/components/OTPInput'; // from Module 7

export default function OTPVerifyScreen() {
  const { phone, email } = useLocalSearchParams<{ phone?: string; email?: string }>();
  const [isVerifying, setIsVerifying] = useState(false);
  const [countdown, setCountdown] = useState(60);
  const intervalRef = useRef<ReturnType<typeof setInterval>>();

  // Countdown timer for resend
  useEffect(() => {
    intervalRef.current = setInterval(() => {
      setCountdown(prev => {
        if (prev <= 1) { clearInterval(intervalRef.current); return 0; }
        return prev - 1;
      });
    }, 1000);
    return () => clearInterval(intervalRef.current);
  }, []);

  const handleComplete = async (otp: string) => {
    setIsVerifying(true);
    try {
      if (phone) {
        await api.auth.verifyPhoneOtp(phone, otp);
      } else if (email) {
        await api.auth.verifyEmailOtp(email, otp);
      }
      router.replace('/(app)/(tabs)/');
    } catch {
      Alert.alert('Invalid Code', 'The code you entered is incorrect or expired.');
    } finally {
      setIsVerifying(false);
    }
  };

  const handleResend = async () => {
    await api.auth.resendOtp(phone ?? email ?? '');
    setCountdown(60);
    // restart countdown
    clearInterval(intervalRef.current);
    intervalRef.current = setInterval(() => {
      setCountdown(prev => {
        if (prev <= 1) { clearInterval(intervalRef.current); return 0; }
        return prev - 1;
      });
    }, 1000);
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Enter verification code</Text>
      <Text style={styles.subtitle}>
        We sent a 6-digit code to{'\n'}
        <Text style={styles.target}>{phone ?? email}</Text>
      </Text>

      <OTPInput length={6} onComplete={handleComplete} />

      {isVerifying && <ActivityIndicator style={{ marginTop: 24 }} />}

      <View style={styles.resendRow}>
        {countdown > 0 ? (
          <Text style={styles.countdown}>Resend in {countdown}s</Text>
        ) : (
          <Pressable onPress={handleResend}>
            <Text style={styles.resendLink}>Resend Code</Text>
          </Pressable>
        )}
      </View>
    </View>
  );
}
```

---

### 1.4 Protected Routes

Expo Router handles protected routes via `redirect` in `_layout.tsx`. Unauthenticated users are redirected to login; authenticated users are redirected away from auth screens.

```tsx
// app/_layout.tsx — root layout guards all routes
import { Stack, Redirect } from 'expo-router';
import { useAuthStore } from '@/store/useAuthStore';
import * as SplashScreen from 'expo-splash-screen';
import { useEffect } from 'react';

SplashScreen.preventAutoHideAsync();

export default function RootLayout() {
  const { isLoggedIn, isHydrating } = useAuthStore();

  useEffect(() => {
    if (!isHydrating) SplashScreen.hideAsync();
  }, [isHydrating]);

  // Keep splash visible while restoring session
  if (isHydrating) return null;

  return (
    <Stack screenOptions={{ headerShown: false }}>
      {/* redirect prop: if condition is true, skip this route group */}
      <Stack.Screen name="(auth)" redirect={isLoggedIn} />
      <Stack.Screen name="(app)" redirect={!isLoggedIn} />
    </Stack>
  );
}
```

```tsx
// app/index.tsx — smart landing: decides where to send the user
import { Redirect } from 'expo-router';
import { useAuthStore } from '@/store/useAuthStore';

export default function Index() {
  const isLoggedIn = useAuthStore(s => s.isLoggedIn);
  return isLoggedIn
    ? <Redirect href="/(app)/(tabs)/" />
    : <Redirect href="/(auth)/login" />;
}
```

**Protecting individual screens** (role-based):
```tsx
// app/(app)/admin.tsx
import { Redirect } from 'expo-router';
import { useAuthStore } from '@/store/useAuthStore';

export default function AdminScreen() {
  const user = useAuthStore(s => s.user);

  if (user?.role !== 'admin') {
    return <Redirect href="/(app)/(tabs)/" />;
  }

  return (/* admin UI */);
}
```

---

### 1.5 Auth State Persistence

Persist the auth state so users don't have to log in every time the app opens.

```tsx
// src/store/useAuthStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import * as SecureStore from 'expo-secure-store';

type User = { id: string; name: string; email: string; role: string };

type AuthState = {
  user: User | null;
  accessToken: string | null;
  isLoggedIn: boolean;
  isHydrating: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  setTokens: (access: string, refresh: string) => Promise<void>;
};

// Custom SecureStore adapter for Zustand persist
const secureStorage = {
  getItem: async (key: string) => SecureStore.getItemAsync(key),
  setItem: async (key: string, value: string) => SecureStore.setItemAsync(key, value),
  removeItem: async (key: string) => SecureStore.deleteItemAsync(key),
};

export const useAuthStore = create<AuthState>()(
  persist(
    (set, get) => ({
      user: null,
      accessToken: null,
      isLoggedIn: false,
      isHydrating: true,

      login: async (email, password) => {
        const { user, accessToken, refreshToken } = await api.auth.login(email, password);
        // Store refresh token in SecureStore (not in persisted state)
        await SecureStore.setItemAsync('refreshToken', refreshToken);
        set({ user, accessToken, isLoggedIn: true });
      },

      logout: async () => {
        await SecureStore.deleteItemAsync('refreshToken');
        await api.auth.logout().catch(() => {}); // best-effort server logout
        set({ user: null, accessToken: null, isLoggedIn: false });
      },

      setTokens: async (access, refresh) => {
        await SecureStore.setItemAsync('refreshToken', refresh);
        set({ accessToken: access });
      },
    }),
    {
      name: 'auth-store',
      storage: createJSONStorage(() => secureStorage),
      // Only persist non-sensitive data — tokens handled separately
      partialize: (state) => ({
        user: state.user,
        isLoggedIn: state.isLoggedIn,
      }),
      onRehydrateStorage: () => (state) => {
        // Called after hydration completes
        state?.setState?.({ isHydrating: false } as any);
      },
    }
  )
);
```

---

### 1.6 Auto-Login

Restore the user session silently on app launch using the stored refresh token.

```tsx
// src/hooks/useAutoLogin.ts
import { useEffect } from 'react';
import * as SecureStore from 'expo-secure-store';
import { useAuthStore } from '@/store/useAuthStore';

export function useAutoLogin() {
  const { setTokens, logout, isHydrating } = useAuthStore();

  useEffect(() => {
    if (isHydrating) return; // wait for persist hydration

    const tryAutoLogin = async () => {
      const refreshToken = await SecureStore.getItemAsync('refreshToken');
      if (!refreshToken) return; // no token — user must log in manually

      try {
        // Exchange refresh token for new access token
        const { accessToken, refreshToken: newRefresh } = await api.auth.refresh(refreshToken);
        await setTokens(accessToken, newRefresh);
      } catch {
        // Refresh token expired — clean up and send to login
        await logout();
      }
    };

    tryAutoLogin();
  }, [isHydrating]);
}

// Use in root layout
export default function RootLayout() {
  useAutoLogin(); // runs once, silently refreshes session

  return <Stack />;
}
```

---

### 1.7 Logout Flow

A complete logout clears all state, tokens, cache, and navigates to login.

```tsx
// src/hooks/useLogout.ts
import { useCallback } from 'react';
import { Alert } from 'react-native';
import { router } from 'expo-router';
import * as SecureStore from 'expo-secure-store';
import { useAuthStore } from '@/store/useAuthStore';
import { useQueryClient } from '@tanstack/react-query';

export function useLogout() {
  const logout = useAuthStore(s => s.logout);
  const queryClient = useQueryClient();

  return useCallback(async (confirm = true) => {
    if (confirm) {
      await new Promise<void>((resolve, reject) =>
        Alert.alert('Sign Out', 'Are you sure you want to sign out?', [
          { text: 'Cancel', style: 'cancel', onPress: reject },
          { text: 'Sign Out', style: 'destructive', onPress: resolve },
        ])
      ).catch(() => { return; }); // user cancelled
    }

    // 1. Notify server (best-effort — don't block on failure)
    await api.auth.logout().catch(() => {});

    // 2. Clear auth store + SecureStore tokens
    await logout();

    // 3. Clear all server-side cached data
    queryClient.clear();

    // 4. Navigate to login — replace so back button doesn't return to app
    router.replace('/(auth)/login');
  }, [logout, queryClient]);
}

// Usage
function ProfileScreen() {
  const handleLogout = useLogout();
  return (
    <Pressable onPress={() => handleLogout()}>
      <Text>Sign Out</Text>
    </Pressable>
  );
}
```

---

## 2. Token-Based Auth

### 2.1 JWT Basics

A JWT (JSON Web Token) is a self-contained token with three Base64-encoded parts:

```
eyJhbGciOiJIUzI1NiJ9   ← Header  (algorithm)
.eyJ1c2VySWQiOiIxMjMifQ  ← Payload (claims: userId, exp, iat, role)
.SflKxwRJSMeKKF2QT4fw     ← Signature (server verifies this)
```

```tsx
// Decode a JWT payload (without verification — for client-side reading only)
function decodeJwtPayload(token: string): Record<string, any> {
  const payload = token.split('.')[1];
  const decoded = atob(payload.replace(/-/g, '+').replace(/_/g, '/'));
  return JSON.parse(decoded);
}

const payload = decodeJwtPayload(accessToken);
// { userId: '123', email: 'nawaz@example.com', role: 'admin', exp: 1714000000, iat: 1713996400 }

// Check expiry
function isTokenExpired(token: string): boolean {
  const { exp } = decodeJwtPayload(token);
  return Date.now() >= exp * 1000; // exp is in seconds
}

// Check if token expires soon (within 5 minutes)
function isTokenExpiringSoon(token: string, bufferMs = 5 * 60 * 1000): boolean {
  const { exp } = decodeJwtPayload(token);
  return Date.now() >= exp * 1000 - bufferMs;
}
```

> **Never trust the client-side decoded payload for authorization decisions.** Always verify the JWT on the server. The client decodes it only to read metadata (like expiry time).

---

### 2.2 Access and Refresh Tokens

| | Access Token | Refresh Token |
|---|---|---|
| Lifetime | Short (15min – 1hr) | Long (7 – 30 days) |
| Used for | Authorizing API requests | Getting new access tokens |
| Storage | Memory or SecureStore | SecureStore only |
| Sent to | Every API request | Only `/auth/refresh` |
| Rotate | No | Yes (each refresh issues new one) |

```tsx
// Token lifecycle
const TOKEN_KEYS = {
  access: 'auth.accessToken',
  refresh: 'auth.refreshToken',
} as const;

export const tokenStore = {
  getAccess: () => SecureStore.getItemAsync(TOKEN_KEYS.access),
  getRefresh: () => SecureStore.getItemAsync(TOKEN_KEYS.refresh),

  setTokens: async (access: string, refresh: string) => {
    await Promise.all([
      SecureStore.setItemAsync(TOKEN_KEYS.access, access),
      SecureStore.setItemAsync(TOKEN_KEYS.refresh, refresh),
    ]);
  },

  clearTokens: async () => {
    await Promise.all([
      SecureStore.deleteItemAsync(TOKEN_KEYS.access),
      SecureStore.deleteItemAsync(TOKEN_KEYS.refresh),
    ]);
  },
};
```

---

### 2.3 Token Storage (SecureStore / Keychain)

**Never store tokens in AsyncStorage** — it is unencrypted plain text on disk.

```bash
npx expo install expo-secure-store
```

`expo-secure-store` uses **iOS Keychain** and **Android Keystore** — hardware-backed encrypted storage.

```tsx
import * as SecureStore from 'expo-secure-store';

// Store a value
await SecureStore.setItemAsync('accessToken', token, {
  // iOS Keychain accessibility options
  keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
  // WHEN_UNLOCKED_THIS_DEVICE_ONLY → not backed up to iCloud, accessible only when unlocked
  // AFTER_FIRST_UNLOCK              → accessible after first unlock (for background tasks)
  // ALWAYS                          → accessible always (less secure)
});

// Read
const token = await SecureStore.getItemAsync('accessToken');

// Delete
await SecureStore.deleteItemAsync('accessToken');

// Check availability (some Android devices may not support it)
const available = await SecureStore.isAvailableAsync();
```

**Accessibility options:**

| Option | iOS Equivalent | When accessible |
|---|---|---|
| `WHEN_UNLOCKED` (default) | `kSecAttrAccessibleWhenUnlocked` | Only when device is unlocked |
| `WHEN_UNLOCKED_THIS_DEVICE_ONLY` | `...WhenUnlockedThisDeviceOnly` | Unlocked + not in iCloud backup |
| `AFTER_FIRST_UNLOCK` | `...AfterFirstUnlock` | After first unlock since boot |
| `ALWAYS` | `...Always` | Always (even on locked device) |

> Use `WHEN_UNLOCKED_THIS_DEVICE_ONLY` for tokens — they won't roam to other devices via iCloud backup and can only be read when the screen is unlocked.

---

### 2.4 Token Refresh Interceptor

See Module 6 Section 5.4 for the full implementation. Here's the summary pattern:

```tsx
// src/lib/apiClient.ts
import axios from 'axios';
import { tokenStore } from '@/lib/tokenStore';
import { useAuthStore } from '@/store/useAuthStore';
import { router } from 'expo-router';

const apiClient = axios.create({ baseURL: process.env.EXPO_PUBLIC_API_URL });

let isRefreshing = false;
let queue: Array<{ resolve: (t: string) => void; reject: (e: unknown) => void }> = [];

function processQueue(error: unknown, token: string | null) {
  queue.forEach(p => token ? p.resolve(token) : p.reject(error));
  queue = [];
}

// Attach access token to every request
apiClient.interceptors.request.use(async config => {
  const token = await tokenStore.getAccess();
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Handle 401 — refresh and replay
apiClient.interceptors.response.use(
  r => r,
  async error => {
    const original = error.config;
    if (error.response?.status !== 401 || original._retry) return Promise.reject(error);

    if (isRefreshing) {
      return new Promise((resolve, reject) => queue.push({ resolve, reject }))
        .then(token => { original.headers.Authorization = `Bearer ${token}`; return apiClient(original); });
    }

    original._retry = true;
    isRefreshing = true;

    try {
      const refresh = await tokenStore.getRefresh();
      const { data } = await axios.post(`${process.env.EXPO_PUBLIC_API_URL}/auth/refresh`, { refresh });
      await tokenStore.setTokens(data.accessToken, data.refreshToken);
      processQueue(null, data.accessToken);
      original.headers.Authorization = `Bearer ${data.accessToken}`;
      return apiClient(original);
    } catch (e) {
      processQueue(e, null);
      await tokenStore.clearTokens();
      useAuthStore.getState().logout();
      router.replace('/(auth)/login');
      return Promise.reject(e);
    } finally {
      isRefreshing = false;
    }
  }
);

export default apiClient;
```

---

### 2.5 Session Expiry Handling

Handle expired sessions gracefully — show a modal instead of a jarring redirect.

```tsx
// src/components/SessionExpiredModal.tsx
import { Modal, View, Text, Pressable, StyleSheet } from 'react-native';
import { useAuthStore } from '@/store/useAuthStore';
import { router } from 'expo-router';

export function SessionExpiredModal() {
  const { sessionExpired, dismissSessionExpired } = useAuthStore();

  const handleLogin = () => {
    dismissSessionExpired();
    router.replace('/(auth)/login');
  };

  return (
    <Modal visible={sessionExpired} transparent animationType="fade">
      <View style={styles.overlay}>
        <View style={styles.card}>
          <Text style={styles.title}>Session Expired</Text>
          <Text style={styles.body}>
            Your session has expired. Please sign in again to continue.
          </Text>
          <Pressable onPress={handleLogin} style={styles.button}>
            <Text style={styles.buttonText}>Sign In Again</Text>
          </Pressable>
        </View>
      </View>
    </Modal>
  );
}

// In the interceptor — set sessionExpired instead of navigating directly
// This prevents navigation errors when the 401 happens deep inside a stack
useAuthStore.getState().setSessionExpired(true);
```

---

## 3. Firebase Authentication

### 3.1 Firebase Project Setup

1. Go to [Firebase Console](https://console.firebase.google.com) → Create project
2. Add iOS app (use bundle ID from `app.json`) → Download `GoogleService-Info.plist`
3. Add Android app (use package from `app.json`) → Download `google-services.json`
4. In Expo: add both files to project root

```json
// app.json
{
  "expo": {
    "ios": {
      "googleServicesFile": "./GoogleService-Info.plist"
    },
    "android": {
      "googleServicesFile": "./google-services.json"
    },
    "plugins": [
      "@react-native-firebase/app",
      "@react-native-firebase/auth"
    ]
  }
}
```

```bash
npx expo install @react-native-firebase/app @react-native-firebase/auth
npx expo run:ios   # must build locally — Firebase requires native code
```

> Firebase requires a **Dev Client build** — it will not work in Expo Go.

---

### 3.2 @react-native-firebase/auth

```tsx
import auth from '@react-native-firebase/auth';
import { useEffect, useState } from 'react';
import { FirebaseAuthTypes } from '@react-native-firebase/auth';

// Listen to auth state changes — the Firebase-native auth hook
export function useFirebaseAuth() {
  const [user, setUser] = useState<FirebaseAuthTypes.User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    // onAuthStateChanged fires immediately with current user (null if not logged in)
    const unsubscribe = auth().onAuthStateChanged(firebaseUser => {
      setUser(firebaseUser);
      setIsLoading(false);
    });
    return unsubscribe; // cleanup on unmount
  }, []);

  return { user, isLoading, isLoggedIn: !!user };
}
```

---

### 3.3 Email/Password Auth

```tsx
import auth from '@react-native-firebase/auth';

// Sign up
async function signUp(email: string, password: string) {
  const credential = await auth().createUserWithEmailAndPassword(email, password);
  // Update display name
  await credential.user.updateProfile({ displayName: 'Nawaz' });
  // Send email verification
  await credential.user.sendEmailVerification();
  return credential.user;
}

// Sign in
async function signIn(email: string, password: string) {
  const credential = await auth().signInWithEmailAndPassword(email, password);
  return credential.user;
}

// Password reset
async function resetPassword(email: string) {
  await auth().sendPasswordResetEmail(email);
}

// Change password
async function changePassword(currentPassword: string, newPassword: string) {
  const user = auth().currentUser;
  if (!user || !user.email) return;

  // Re-authenticate before sensitive operation
  const credential = auth.EmailAuthProvider.credential(user.email, currentPassword);
  await user.reauthenticateWithCredential(credential);
  await user.updatePassword(newPassword);
}

// Sign out
async function signOut() {
  await auth().signOut();
}

// Error handling — Firebase throws typed error codes
function handleFirebaseError(error: any): string {
  switch (error.code) {
    case 'auth/email-already-in-use': return 'This email is already registered.';
    case 'auth/invalid-email': return 'Invalid email address.';
    case 'auth/weak-password': return 'Password must be at least 6 characters.';
    case 'auth/user-not-found':
    case 'auth/wrong-password': return 'Incorrect email or password.';
    case 'auth/too-many-requests': return 'Too many attempts. Try again later.';
    case 'auth/network-request-failed': return 'Network error. Check your connection.';
    default: return 'Authentication failed. Please try again.';
  }
}
```

---

### 3.4 Google Sign-In

```bash
npx expo install @react-native-google-signin/google-signin
```

```tsx
import auth from '@react-native-firebase/auth';
import { GoogleSignin, statusCodes } from '@react-native-google-signin/google-signin';

// Configure once (in app _layout or a service file)
GoogleSignin.configure({
  webClientId: 'YOUR_WEB_CLIENT_ID', // from Firebase Console → Project Settings → Web API Key
});

async function signInWithGoogle() {
  try {
    // Check Play Services availability (Android)
    await GoogleSignin.hasPlayServices({ showPlayServicesUpdateDialog: true });

    // Open Google account picker
    const { idToken } = await GoogleSignin.signIn();

    // Create Firebase credential from Google token
    const googleCredential = auth.GoogleAuthProvider.credential(idToken);

    // Sign into Firebase
    const userCredential = await auth().signInWithCredential(googleCredential);
    return userCredential.user;
  } catch (error: any) {
    if (error.code === statusCodes.SIGN_IN_CANCELLED) return null; // user cancelled
    if (error.code === statusCodes.IN_PROGRESS) return null;       // already signing in
    throw error;
  }
}
```

---

### 3.5 Apple Sign-In (required for iOS)

Apple requires Sign in with Apple for any app that offers third-party social login on iOS.

```bash
npx expo install @invertase/react-native-apple-authentication
```

```tsx
import auth from '@react-native-firebase/auth';
import appleAuth from '@invertase/react-native-apple-authentication';

async function signInWithApple() {
  // Request authorization from Apple
  const appleAuthResponse = await appleAuth.performRequest({
    requestedOperation: appleAuth.Operation.LOGIN,
    requestedScopes: [appleAuth.Scope.EMAIL, appleAuth.Scope.FULL_NAME],
  });

  // Verify the nonce (security requirement)
  if (!appleAuthResponse.identityToken) {
    throw new Error('Apple sign-in failed — no identity token returned');
  }

  // Create Firebase credential
  const { identityToken, nonce } = appleAuthResponse;
  const appleCredential = auth.AppleAuthProvider.credential(identityToken, nonce);

  // Sign into Firebase
  const userCredential = await auth().signInWithCredential(appleCredential);
  return userCredential.user;
}

// Apple Sign-In button — must use official styling per Apple guidelines
import { AppleButton } from '@invertase/react-native-apple-authentication';

<AppleButton
  buttonStyle={AppleButton.Style.BLACK}
  buttonType={AppleButton.Type.SIGN_IN}
  style={{ width: '100%', height: 50 }}
  onPress={signInWithApple}
/>
```

**`app.json` requirement:**
```json
{
  "expo": {
    "ios": {
      "usesAppleSignIn": true
    }
  }
}
```

---

### 3.6 Phone Authentication

```tsx
import auth from '@react-native-firebase/auth';
import { useState } from 'react';

export function usePhoneAuth() {
  const [confirmation, setConfirmation] = useState<any>(null);

  // Step 1: Send OTP
  const sendOtp = async (phoneNumber: string) => {
    // phoneNumber must be in E.164 format: +919876543210
    const confirmation = await auth().signInWithPhoneNumber(phoneNumber);
    setConfirmation(confirmation);
  };

  // Step 2: Verify OTP
  const verifyOtp = async (otp: string) => {
    if (!confirmation) throw new Error('No confirmation pending');
    const credential = await confirmation.confirm(otp);
    return credential.user;
  };

  return { sendOtp, verifyOtp, isWaitingForOtp: !!confirmation };
}

// Usage
function PhoneLoginScreen() {
  const [phone, setPhone] = useState('');
  const { sendOtp, verifyOtp, isWaitingForOtp } = usePhoneAuth();

  return isWaitingForOtp
    ? <OTPInput length={6} onComplete={verifyOtp} />
    : (
      <View>
        <TextInput value={phone} onChangeText={setPhone} keyboardType="phone-pad" placeholder="+91 98765 43210" />
        <Pressable onPress={() => sendOtp(phone)}><Text>Send Code</Text></Pressable>
      </View>
    );
}
```

---

### 3.7 Anonymous Auth

Sign in users without credentials — useful for letting users explore the app before signing up.

```tsx
import auth from '@react-native-firebase/auth';

// Sign in anonymously
const credential = await auth().signInAnonymously();
const { uid, isAnonymous } = credential.user;
// uid is stable within the app install — clears on reinstall

// Convert anonymous account to permanent (link credentials)
async function upgradeAnonymousAccount(email: string, password: string) {
  const user = auth().currentUser;
  if (!user?.isAnonymous) return;

  const emailCredential = auth.EmailAuthProvider.credential(email, password);
  const linked = await user.linkWithCredential(emailCredential);
  // The user's uid stays the same — all their data is preserved
  return linked.user;
}

// Delete anonymous account on logout (clean up Firestore data too)
async function cleanupAnonymous() {
  const user = auth().currentUser;
  if (user?.isAnonymous) {
    await user.delete();
  }
}
```

---

## 4. OAuth / Social Login

### 4.1 expo-auth-session

`expo-auth-session` handles the OAuth browser redirect flow with PKCE. It works in Expo Go for development and supports deep link callbacks.

```bash
npx expo install expo-auth-session expo-crypto expo-web-browser
```

```tsx
import * as AuthSession from 'expo-auth-session';
import * as WebBrowser from 'expo-web-browser';

// Required: complete the OAuth redirect in the browser
WebBrowser.maybeCompleteAuthSession();
```

---

### 4.2 Google OAuth

```tsx
import * as Google from 'expo-auth-session/providers/google';
import * as WebBrowser from 'expo-web-browser';
import { useEffect } from 'react';

WebBrowser.maybeCompleteAuthSession();

export default function GoogleLoginButton() {
  const [request, response, promptAsync] = Google.useAuthRequest({
    clientId: Platform.select({
      ios: 'IOS_CLIENT_ID',       // from Google Cloud Console
      android: 'ANDROID_CLIENT_ID',
      default: 'WEB_CLIENT_ID',
    }),
    scopes: ['openid', 'profile', 'email'],
  });

  useEffect(() => {
    if (response?.type === 'success') {
      const { authentication } = response;
      // Send authentication.accessToken to your backend to verify
      handleGoogleAuth(authentication?.accessToken);
    }
  }, [response]);

  return (
    <Pressable
      onPress={() => promptAsync()}
      disabled={!request}
      style={styles.socialButton}
    >
      <GoogleIcon />
      <Text>Continue with Google</Text>
    </Pressable>
  );
}

async function handleGoogleAuth(accessToken?: string) {
  if (!accessToken) return;
  // Exchange with your backend for your app's JWT
  const { user, tokens } = await api.auth.googleLogin(accessToken);
  await tokenStore.setTokens(tokens.access, tokens.refresh);
  useAuthStore.getState().setUser(user);
  router.replace('/(app)/(tabs)/');
}
```

---

### 4.3 Facebook Login

```bash
npx expo install expo-auth-session
```

```tsx
import * as Facebook from 'expo-auth-session/providers/facebook';
import * as WebBrowser from 'expo-web-browser';

WebBrowser.maybeCompleteAuthSession();

export function FacebookLoginButton() {
  const [request, response, promptAsync] = Facebook.useAuthRequest({
    clientId: 'YOUR_FACEBOOK_APP_ID', // from developers.facebook.com
    scopes: ['public_profile', 'email'],
  });

  useEffect(() => {
    if (response?.type === 'success') {
      const { code } = response.params;
      handleFacebookAuth(code);
    }
  }, [response]);

  return (
    <Pressable onPress={() => promptAsync()} style={styles.fbButton}>
      <Text style={{ color: '#fff' }}>Continue with Facebook</Text>
    </Pressable>
  );
}
```

---

### 4.4 Apple Sign-In

For apps NOT using Firebase, use `expo-apple-authentication` directly:

```bash
npx expo install expo-apple-authentication
```

```tsx
import * as AppleAuthentication from 'expo-apple-authentication';

export function AppleLoginButton() {
  const handleAppleLogin = async () => {
    try {
      const credential = await AppleAuthentication.signInAsync({
        requestedScopes: [
          AppleAuthentication.AppleAuthenticationScope.FULL_NAME,
          AppleAuthentication.AppleAuthenticationScope.EMAIL,
        ],
      });

      // credential.identityToken — JWT from Apple
      // credential.authorizationCode — single-use code
      // credential.fullName — only provided on FIRST sign-in
      // credential.email — only provided on FIRST sign-in

      // Send to backend for verification
      const { user, tokens } = await api.auth.appleLogin({
        identityToken: credential.identityToken,
        authorizationCode: credential.authorizationCode,
        fullName: credential.fullName,
        email: credential.email,
      });

      await tokenStore.setTokens(tokens.access, tokens.refresh);
      router.replace('/(app)/(tabs)/');
    } catch (error: any) {
      if (error.code !== 'ERR_REQUEST_CANCELED') throw error;
      // User cancelled — do nothing
    }
  };

  return (
    <AppleAuthentication.AppleAuthenticationButton
      buttonType={AppleAuthentication.AppleAuthenticationButtonType.SIGN_IN}
      buttonStyle={AppleAuthentication.AppleAuthenticationButtonStyle.BLACK}
      cornerRadius={12}
      style={{ width: '100%', height: 50 }}
      onPress={handleAppleLogin}
    />
  );
}
```

> Apple only sends `email` and `fullName` on the **first** sign-in. Store them on your backend immediately — subsequent logins won't include them.

---

### 4.5 Microsoft / Azure AD

```tsx
import * as AuthSession from 'expo-auth-session';
import * as WebBrowser from 'expo-web-browser';
import * as Crypto from 'expo-crypto';

WebBrowser.maybeCompleteAuthSession();

const TENANT_ID = 'YOUR_TENANT_ID';  // or 'common' for multi-tenant
const CLIENT_ID = 'YOUR_CLIENT_ID';  // from Azure App Registration

const discovery = {
  authorizationEndpoint: `https://login.microsoftonline.com/${TENANT_ID}/oauth2/v2.0/authorize`,
  tokenEndpoint: `https://login.microsoftonline.com/${TENANT_ID}/oauth2/v2.0/token`,
};

export function MicrosoftLoginButton() {
  const redirectUri = AuthSession.makeRedirectUri({ scheme: 'myapp' });

  const [request, response, promptAsync] = AuthSession.useAuthRequest(
    {
      clientId: CLIENT_ID,
      scopes: ['openid', 'profile', 'email', 'User.Read'],
      redirectUri,
      responseType: AuthSession.ResponseType.Code,
      usePKCE: true,
    },
    discovery
  );

  useEffect(() => {
    if (response?.type === 'success') {
      handleMicrosoftAuth(response.params.code, request?.codeVerifier);
    }
  }, [response]);

  return (
    <Pressable onPress={() => promptAsync()} style={styles.msButton}>
      <Text>Continue with Microsoft</Text>
    </Pressable>
  );
}

async function handleMicrosoftAuth(code: string, codeVerifier?: string) {
  const { user, tokens } = await api.auth.microsoftLogin({ code, codeVerifier });
  await tokenStore.setTokens(tokens.access, tokens.refresh);
  router.replace('/(app)/(tabs)/');
}
```

---

### 4.6 PKCE Flow

PKCE (Proof Key for Code Exchange) prevents authorization code interception attacks — mandatory for mobile OAuth.

```
1. App generates:
   codeVerifier  = 43–128 random chars
   codeChallenge = base64url(SHA256(codeVerifier))

2. Authorization request includes:
   code_challenge=<hash>
   code_challenge_method=S256

3. Auth server stores the challenge

4. Token exchange includes:
   code_verifier=<original>

5. Auth server verifies: SHA256(verifier) === stored_challenge
   → Only the original app can exchange the code
```

`expo-auth-session` handles PKCE automatically:

```tsx
const [request, response, promptAsync] = AuthSession.useAuthRequest(
  {
    clientId: CLIENT_ID,
    redirectUri,
    scopes: ['openid', 'profile'],
    usePKCE: true,  // ← generates verifier + challenge automatically
  },
  discovery
);

// On success — request.codeVerifier is the verifier to send with token exchange
if (response?.type === 'success') {
  await exchangeCodeForToken(response.params.code, request.codeVerifier!);
}
```

---

## 5. Enterprise Auth

### 5.1 Azure AD / Entra ID

Azure AD (now Microsoft Entra ID) is the enterprise identity platform. Key concepts:

| Concept | Meaning |
|---|---|
| **Tenant** | Your organization's Azure AD directory |
| **App Registration** | OAuth client registered in Azure |
| **Client ID** | Public identifier for your app |
| **Scopes** | Permissions requested (e.g., `User.Read`) |
| **ID Token** | JWT with user identity (OpenID Connect) |
| **Access Token** | JWT for calling Microsoft Graph or your API |
| **Refresh Token** | Long-lived token for getting new access tokens |

**App Registration setup:**
1. Azure Portal → Microsoft Entra ID → App registrations → New registration
2. Set Redirect URI: `myapp://auth` (Mobile and desktop applications)
3. API Permissions → Add `openid`, `profile`, `email`, `User.Read`
4. Note the **Application (client) ID** and **Directory (tenant) ID**

---

### 5.2 SSO Integration

Single Sign-On — one login works across multiple apps in the same organization.

```tsx
// With MSAL, SSO works automatically within the same app family.
// Users signed into the Microsoft Authenticator app are auto-signed in.

import { PublicClientApplication } from '@azure/msal-browser'; // web
// For RN use react-native-msal instead

// Silent token acquisition (SSO) — tries to get token without user interaction
try {
  const silentRequest = { scopes: ['User.Read'], account: existingAccount };
  const result = await msalInstance.acquireTokenSilent(silentRequest);
  // result.accessToken — no prompt shown
} catch (error) {
  if (error instanceof InteractionRequiredAuthError) {
    // Silent failed — need interactive login
    const result = await msalInstance.acquireTokenInteractive({ scopes: ['User.Read'] });
  }
}
```

---

### 5.3 SAML Basics

SAML (Security Assertion Markup Language) is an XML-based enterprise SSO protocol. Mobile apps cannot implement SAML directly — use a hybrid approach:

```
Mobile App → WebView / Browser
  → Enterprise IdP (SAML) → SP-Initiated SSO
  → IdP returns SAML Assertion → Your backend
  → Backend issues JWT to mobile app
  → App stores JWT in SecureStore
```

**Practical approach for mobile + SAML:**
```tsx
// 1. App opens a WebView/browser to your backend's SAML initiation URL
await WebBrowser.openAuthSessionAsync(
  'https://api.yourapp.com/auth/saml/initiate',
  'myapp://auth/saml/callback'
);

// 2. Backend handles the SAML flow, issues JWT
// 3. Backend redirects to: myapp://auth/saml/callback?token=<jwt>
// 4. App intercepts the deep link and extracts the JWT

const subscription = Linking.addEventListener('url', async ({ url }) => {
  const { queryParams } = ExpoLinking.parse(url);
  if (queryParams?.token) {
    await tokenStore.setTokens(queryParams.token as string, queryParams.refresh as string);
    router.replace('/(app)/(tabs)/');
  }
});
```

---

### 5.4 MSAL Library

Microsoft Authentication Library (MSAL) for React Native:

```bash
npx expo install react-native-msal
```

```tsx
import MSALClient from 'react-native-msal';

const msalConfig = {
  auth: {
    clientId: 'YOUR_CLIENT_ID',
    authority: 'https://login.microsoftonline.com/YOUR_TENANT_ID',
    redirectUri: 'msauth.com.yourname.myapp://auth',
  },
};

const msalInstance = new MSALClient(msalConfig);

// Interactive login
async function loginWithMsal() {
  const result = await msalInstance.acquireToken({
    scopes: ['openid', 'profile', 'email', 'User.Read'],
  });

  // result.accessToken — for Microsoft Graph
  // result.idToken    — for your backend (identity)
  // result.account    — user info

  // Send to your backend
  const { user, tokens } = await api.auth.microsoftLogin(result.idToken);
  await tokenStore.setTokens(tokens.access, tokens.refresh);
}

// Silent token refresh (no UI)
async function refreshMsalToken(account: any) {
  const result = await msalInstance.acquireTokenSilent({
    scopes: ['User.Read'],
    account,
  });
  return result.accessToken;
}

// Sign out
async function logoutMsal(account: any) {
  await msalInstance.removeAccount(account);
}
```

---

## 6. Security Best Practices

### 6.1 Secure Token Storage

Summary of the complete token security model:

```tsx
// src/lib/tokenStore.ts — the single source of truth for tokens
import * as SecureStore from 'expo-secure-store';

const SECURE_OPTIONS: SecureStore.SecureStoreOptions = {
  keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
};

export const tokenStore = {
  // Access token — short-lived, stored in SecureStore
  getAccess: () => SecureStore.getItemAsync('auth.at', SECURE_OPTIONS),
  setAccess: (t: string) => SecureStore.setItemAsync('auth.at', t, SECURE_OPTIONS),
  clearAccess: () => SecureStore.deleteItemAsync('auth.at'),

  // Refresh token — long-lived, stored separately
  getRefresh: () => SecureStore.getItemAsync('auth.rt', SECURE_OPTIONS),
  setRefresh: (t: string) => SecureStore.setItemAsync('auth.rt', t, SECURE_OPTIONS),
  clearRefresh: () => SecureStore.deleteItemAsync('auth.rt'),

  setTokens: async (access: string, refresh: string) => {
    await Promise.all([
      SecureStore.setItemAsync('auth.at', access, SECURE_OPTIONS),
      SecureStore.setItemAsync('auth.rt', refresh, SECURE_OPTIONS),
    ]);
  },

  clearAll: async () => {
    await Promise.all([
      SecureStore.deleteItemAsync('auth.at'),
      SecureStore.deleteItemAsync('auth.rt'),
    ]);
  },
};

// Rules:
// ✅ Tokens in expo-secure-store (Keychain / Keystore)
// ✅ WHEN_UNLOCKED_THIS_DEVICE_ONLY for highest security
// ❌ Never in AsyncStorage (unencrypted)
// ❌ Never in MMKV without encryption key
// ❌ Never logged to console or Sentry
// ❌ Never in URL query params
// ❌ Never in Redux store (may be serialized to AsyncStorage via persist)
```

---

### 6.2 Certificate Pinning

Prevents man-in-the-middle attacks by validating the server's certificate against a known public key.

```bash
npx expo install @cossacklabs/react-native-ssl-pinning
# or
npx expo install react-native-ssl-pinning
```

```tsx
import { fetch as pinnedFetch } from 'react-native-ssl-pinning';

// Replace your regular fetch with pinned fetch
const response = await pinnedFetch('https://api.example.com/data', {
  method: 'GET',
  timeoutInterval: 10000,
  sslPinning: {
    certs: ['cert1', 'cert2'], // filenames of .cer files in assets (without extension)
  },
  headers: {
    Authorization: `Bearer ${token}`,
  },
});
```

**Generate and add certificate:**
```bash
# Extract certificate from server
openssl s_client -connect api.example.com:443 -servername api.example.com < /dev/null \
  | openssl x509 -outform DER -out api_example_com.cer

# Add to:
# iOS: ios/YourApp/api_example_com.cer (add to Xcode bundle)
# Android: android/app/src/main/assets/api_example_com.cer
```

> Pin to the **public key** (SPKI), not the full certificate. Certificates rotate every 1–2 years; public keys can stay stable longer. Implement a backup pin so you can rotate without breaking old app versions.

---

### 6.3 Jailbreak / Root Detection

Detect compromised devices where the app's sandbox may be bypassed.

```bash
npx expo install expo-device
# For advanced detection:
npx expo install @rnhooks/jailbreak-detector
```

```tsx
import * as Device from 'expo-device';

async function checkDeviceIntegrity(): Promise<boolean> {
  // Basic check — simulators and emulators are not production devices
  if (!Device.isDevice) {
    console.warn('Running on simulator/emulator');
    return true; // allow in development
  }

  // expo-device doesn't expose jailbreak detection
  // Use react-native-jailmonkey for production
  return true;
}

// With react-native-jailmonkey (requires Dev Client)
import JailMonkey from 'jail-monkey';

function checkJailbreak(): void {
  if (JailMonkey.isJailBroken()) {
    Alert.alert(
      'Security Warning',
      'This app cannot run on a jailbroken or rooted device for security reasons.',
      [{ text: 'OK', onPress: () => BackHandler.exitApp() }],
      { cancelable: false }
    );
  }
  if (JailMonkey.canMockLocation()) {
    Alert.alert('Security Warning', 'Mock location apps are not allowed.');
  }
}
```

---

### 6.4 Biometric Gating

Require biometric authentication to access sensitive screens, not just at login.

```tsx
import * as LocalAuthentication from 'expo-local-authentication';

// src/hooks/useBiometricGate.ts
export function useBiometricGate() {
  const authenticate = async (reason: string): Promise<boolean> => {
    const isAvailable = await LocalAuthentication.hasHardwareAsync();
    const isEnrolled = await LocalAuthentication.isEnrolledAsync();

    if (!isAvailable || !isEnrolled) return true; // fall through — no biometric available

    const result = await LocalAuthentication.authenticateAsync({
      promptMessage: reason,
      fallbackLabel: 'Use Passcode',
      disableDeviceFallback: false,
    });

    return result.success;
  };

  return { authenticate };
}

// Gate sensitive screens — payment, account deletion, change password
export default function PaymentScreen() {
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const { authenticate } = useBiometricGate();

  useFocusEffect(
    useCallback(() => {
      authenticate('Authenticate to access payments').then(ok => {
        if (!ok) router.back(); // not authenticated — go back
        else setIsAuthenticated(true);
      });
    }, [])
  );

  if (!isAuthenticated) return null;
  return (/* payment UI */);
}
```

**Require biometric on app foreground (after N minutes background):**
```tsx
import { AppState } from 'react-native';
import { useRef } from 'react';

const LOCK_AFTER_MS = 5 * 60 * 1000; // 5 minutes

export function useAppLock() {
  const backgroundedAt = useRef<number | null>(null);
  const { authenticate } = useBiometricGate();

  useEffect(() => {
    const subscription = AppState.addEventListener('change', async (state) => {
      if (state === 'background') {
        backgroundedAt.current = Date.now();
      }

      if (state === 'active' && backgroundedAt.current) {
        const elapsed = Date.now() - backgroundedAt.current;
        if (elapsed > LOCK_AFTER_MS) {
          const ok = await authenticate('Unlock the app to continue');
          if (!ok) {
            await useAuthStore.getState().logout();
            router.replace('/(auth)/login');
          }
        }
        backgroundedAt.current = null;
      }
    });

    return () => subscription.remove();
  }, []);
}
```

---

### 6.5 Session Timeout

Automatically expire inactive sessions on the client side.

```tsx
// src/hooks/useSessionTimeout.ts
import { useEffect, useRef, useCallback } from 'react';
import { AppState, PanResponder } from 'react-native';
import { useAuthStore } from '@/store/useAuthStore';

const TIMEOUT_MS = 30 * 60 * 1000; // 30 minutes of inactivity

export function useSessionTimeout() {
  const logout = useAuthStore(s => s.logout);
  const timerRef = useRef<ReturnType<typeof setTimeout>>();
  const lastActivityRef = useRef(Date.now());

  const resetTimer = useCallback(() => {
    lastActivityRef.current = Date.now();
    clearTimeout(timerRef.current);
    timerRef.current = setTimeout(async () => {
      await logout();
      router.replace('/(auth)/login');
    }, TIMEOUT_MS);
  }, [logout]);

  // Detect user activity via touch events
  const panResponder = useRef(
    PanResponder.create({
      onStartShouldSetPanResponderCapture: () => {
        resetTimer();
        return false; // don't capture — just observe
      },
    })
  ).current;

  // Also reset timer when app comes to foreground
  useEffect(() => {
    resetTimer(); // start timer on mount

    const subscription = AppState.addEventListener('change', state => {
      if (state === 'active') {
        const elapsed = Date.now() - lastActivityRef.current;
        if (elapsed > TIMEOUT_MS) {
          logout();
          router.replace('/(auth)/login');
        }
      }
    });

    return () => {
      clearTimeout(timerRef.current);
      subscription.remove();
    };
  }, [resetTimer, logout]);

  return panResponder.panHandlers; // spread onto root view
}

// Usage — wrap root app view
export default function AppLayout() {
  const sessionHandlers = useSessionTimeout();

  return (
    <View style={{ flex: 1 }} {...sessionHandlers}>
      <Stack />
    </View>
  );
}
```

**Persist last-active timestamp across app restarts:**
```tsx
// On app background — save last active time
AppState.addEventListener('change', state => {
  if (state === 'background') {
    AsyncStorage.setItem('lastActive', Date.now().toString());
  }
  if (state === 'active') {
    AsyncStorage.getItem('lastActive').then(ts => {
      if (ts && Date.now() - Number(ts) > TIMEOUT_MS) {
        logout();
      }
    });
  }
});
```

---

*End of Module 9*
