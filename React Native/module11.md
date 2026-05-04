# Module 11 — Push Notifications

> Push notifications are one of the most powerful re-engagement tools in mobile. This module covers the full stack — APNs/FCM setup, Expo's notification layer, deep link handling on tap, and local scheduling.

---

## Table of Contents

1. [Concepts](#1-concepts)
   - 1.1 [Push vs Local Notifications](#11-push-vs-local-notifications)
   - 1.2 [Notification Lifecycle](#12-notification-lifecycle)
   - 1.3 [Foreground vs Background Handling](#13-foreground-vs-background-handling)
   - 1.4 [Notification Channels (Android)](#14-notification-channels-android)
   - 1.5 [Notification Categories (iOS)](#15-notification-categories-ios)

2. [Setup — iOS (APNs)](#2-setup--ios-apns)
   - 2.1 [Apple Developer Certificates](#21-apple-developer-certificates)
   - 2.2 [Push Notification Capability](#22-push-notification-capability)
   - 2.3 [APNs Keys](#23-apns-keys)
   - 2.4 [Provisioning Profiles](#24-provisioning-profiles)

3. [Setup — Android (FCM)](#3-setup--android-fcm)
   - 3.1 [Firebase Project Configuration](#31-firebase-project-configuration)
   - 3.2 [google-services.json](#32-google-servicesjson)
   - 3.3 [FCM Server Key](#33-fcm-server-key)
   - 3.4 [Android 13+ POST_NOTIFICATIONS Permission](#34-android-13-post_notifications-permission)

4. [Implementation](#4-implementation)
   - 4.1 [expo-notifications](#41-expo-notifications)
   - 4.2 [@react-native-firebase/messaging](#42-react-native-firebasemessaging)
   - 4.3 [Device Token Registration](#43-device-token-registration)
   - 4.4 [Token Refresh Handling](#44-token-refresh-handling)
   - 4.5 [Sending to Backend](#45-sending-to-backend)

5. [Notification Handling](#5-notification-handling)
   - 5.1 [Receiving in Foreground](#51-receiving-in-foreground)
   - 5.2 [Receiving in Background](#52-receiving-in-background)
   - 5.3 [Tap-to-Open Deep Linking](#53-tap-to-open-deep-linking)
   - 5.4 [Notification Payload Parsing](#54-notification-payload-parsing)
   - 5.5 [Badge Count Management](#55-badge-count-management)
   - 5.6 [Rich Notifications (Images, Actions)](#56-rich-notifications-images-actions)
   - 5.7 [Silent / Data-Only Notifications](#57-silent--data-only-notifications)

6. [Local Notifications](#6-local-notifications)
   - 6.1 [Scheduling](#61-scheduling)
   - 6.2 [Recurring Notifications](#62-recurring-notifications)
   - 6.3 [Cancelling Scheduled Notifications](#63-cancelling-scheduled-notifications)

---

## 1. Concepts

### 1.1 Push vs Local Notifications

| | Push Notifications | Local Notifications |
|---|---|---|
| Origin | Remote server via APNs/FCM | Scheduled on the device itself |
| Requires internet | Yes (to receive) | No |
| Use cases | New message, order update, breaking news | Reminders, alarms, recurring alerts |
| Delivery guarantee | Best-effort (OS may throttle) | Guaranteed (if app has permission) |
| Payload limit | ~4KB (APNs) / ~4KB (FCM) | No limit |
| Can wake app | Yes (silent push) | Yes |
| Backend required | Yes | No |

**Both share the same permission model** — the user must grant notification permission regardless of the source.

---

### 1.2 Notification Lifecycle

```
Server sends push
        ↓
  APNs / FCM delivers to device
        ↓
  OS receives notification
        ↓
  App state check:
    ├── KILLED     → notification shown in system tray
    │                 user taps → app cold starts → handle tap
    │
    ├── BACKGROUND → notification shown in system tray
    │                 user taps → app foregrounds → handle tap
    │
    └── FOREGROUND → OS would NOT show banner by default
                      app must show in-app UI
                      OR configure to show banner anyway
```

**Key states to handle:**

| App State | Notification displayed? | Who handles it? |
|---|---|---|
| Killed | Yes — system tray | Background message handler |
| Background | Yes — system tray | Background message handler |
| Foreground | No by default | Foreground listener (`setNotificationHandler`) |

---

### 1.3 Foreground vs Background Handling

```tsx
// Foreground — app is active and visible
// You control whether to show the banner, sound, and badge
Notifications.setNotificationHandler({
  handleNotification: async (notification) => ({
    shouldShowAlert: true,   // show banner while app is open
    shouldPlaySound: true,
    shouldSetBadge: true,
  }),
});

// Background — app is in background or killed
// Handler must be registered at module level (before component mounts)
// Used for data-only (silent) notifications, badge resets, etc.
Notifications.addNotificationReceivedListener(notification => {
  // This only fires in FOREGROUND
});

// For background — use TaskManager
TaskManager.defineTask(BACKGROUND_NOTIFICATION_TASK, ({ data, error }) => {
  // This fires in BACKGROUND and KILLED state (data-only only on iOS)
});
```

---

### 1.4 Notification Channels (Android)

Android 8+ (API 26+) requires notifications to be sent to a **channel**. Users can mute individual channels without revoking all notification permission.

```tsx
import * as Notifications from 'expo-notifications';

// Create channels on app launch
async function setupNotificationChannels() {
  await Notifications.setNotificationChannelAsync('orders', {
    name: 'Order Updates',
    description: 'Delivery status, order confirmations',
    importance: Notifications.AndroidImportance.HIGH,
    sound: 'default',
    vibrationPattern: [0, 250, 250, 250],
    lightColor: '#007AFF',
    enableLights: true,
    enableVibrate: true,
    showBadge: true,
    lockscreenVisibility: Notifications.AndroidNotificationVisibility.PUBLIC,
  });

  await Notifications.setNotificationChannelAsync('messages', {
    name: 'Messages',
    description: 'Direct messages and chat',
    importance: Notifications.AndroidImportance.HIGH,
    sound: 'default',
  });

  await Notifications.setNotificationChannelAsync('promotions', {
    name: 'Promotions',
    description: 'Deals and offers',
    importance: Notifications.AndroidImportance.LOW, // silent, no banner
    sound: undefined,
  });

  await Notifications.setNotificationChannelAsync('reminders', {
    name: 'Reminders',
    description: 'Scheduled reminders',
    importance: Notifications.AndroidImportance.DEFAULT,
  });
}

// Delete a channel
await Notifications.deleteNotificationChannelAsync('old-channel');

// List existing channels
const channels = await Notifications.getNotificationChannelsAsync();
```

**Importance levels:**

| Level | Banner | Sound | Peek | Status bar |
|---|---|---|---|---|
| `NONE` | No | No | No | No |
| `MIN` | No | No | No | Yes |
| `LOW` | No | No | No | Yes |
| `DEFAULT` | Yes | No | No | Yes |
| `HIGH` | Yes | Yes | Yes | Yes |
| `MAX` | Yes | Yes | Yes | Yes |

---

### 1.5 Notification Categories (iOS)

iOS notification categories define **action buttons** that appear on long-press or swipe. Set them up once on app launch.

```tsx
import * as Notifications from 'expo-notifications';

async function setupNotificationCategories() {
  await Notifications.setNotificationCategoryAsync('order', [
    {
      identifier: 'track',
      buttonTitle: 'Track Order',
      options: { opensAppToForeground: true },
    },
    {
      identifier: 'cancel',
      buttonTitle: 'Cancel Order',
      options: { opensAppToForeground: false, isDestructive: true },
    },
  ]);

  await Notifications.setNotificationCategoryAsync('message', [
    {
      identifier: 'reply',
      buttonTitle: 'Reply',
      textInput: {            // inline text input (iOS 9+)
        submitButtonTitle: 'Send',
        placeholder: 'Type a reply...',
      },
    },
    {
      identifier: 'mark_read',
      buttonTitle: 'Mark as Read',
      options: { opensAppToForeground: false },
    },
  ]);
}

// The server sends the categoryIdentifier in the notification payload
// iOS shows the buttons automatically
// { aps: { category: 'order', ... } }
```

---

## 2. Setup — iOS (APNs)

### 2.1 Apple Developer Certificates

APNs requires your app to be signed with a certificate that includes push notification entitlements.

**Two ways to authenticate with APNs:**

| Method | Format | Expires | Recommended |
|---|---|---|---|
| APNs Key (`.p8`) | JWT auth token | Never | ✅ Yes |
| APNs Certificate (`.p12`) | TLS certificate | 1 year | ❌ Legacy |

**Steps for APNs Key:**
1. Apple Developer Console → **Certificates, Identifiers & Profiles**
2. **Keys** → **+** → Check "Apple Push Notifications service (APNs)"
3. Download the `.p8` file — **download once, keep it safe**
4. Note the **Key ID** and your **Team ID**

---

### 2.2 Push Notification Capability

Enable push notifications for your App ID:

1. Developer Console → **Identifiers** → Select your App ID
2. Scroll to **Push Notifications** → Enable → Save

**In Expo — `app.json` handles this automatically with EAS Build:**
```json
{
  "expo": {
    "ios": {
      "bundleIdentifier": "com.yourname.myapp",
      "entitlements": {
        "aps-environment": "production"
      }
    }
  }
}
```

> For development builds: `"aps-environment": "development"`. EAS Build sets this automatically based on the build profile.

---

### 2.3 APNs Keys

Your `.p8` key file is used by your **backend** (or Expo's push service) to send notifications. Never put it in your app bundle.

**Key information you need on the backend:**
```
Key ID:   ABC123DEFG         (from Apple Developer Console)
Team ID:  XYZ987ABCD         (your Apple Developer Team ID)
Key file: AuthKey_ABC123DEFG.p8
Bundle:   com.yourname.myapp
```

**Sending directly from backend (Node.js example):**
```js
// backend — never in the React Native app
import apn from '@parse/node-apn';

const provider = new apn.Provider({
  token: {
    key: './AuthKey_ABC123DEFG.p8',
    keyId: 'ABC123DEFG',
    teamId: 'XYZ987ABCD',
  },
  production: true, // false for sandbox/development
});

const notification = new apn.Notification();
notification.topic = 'com.yourname.myapp';
notification.alert = { title: 'Order Shipped', body: 'Your order is on the way!' };
notification.badge = 1;
notification.payload = { orderId: '12345', screen: '/orders/12345' };

const result = await provider.send(notification, deviceToken);
```

---

### 2.4 Provisioning Profiles

A provisioning profile ties together: your App ID + certificates + device UDIDs (for dev) and authorizes the app to run and use APNs.

**With EAS Build — automatic provisioning:**
```bash
eas build --platform ios --profile development
# EAS automatically creates/updates provisioning profiles
# No manual certificate management needed
```

**Manual flow (if managing profiles yourself):**
1. Developer Console → **Profiles** → **+**
2. Type: **iOS App Development** (dev) or **App Store** (production)
3. Select App ID → Select certificate → Select devices (dev only) → Generate
4. Download `.mobileprovision` → Double-click to install in Xcode

---

## 3. Setup — Android (FCM)

### 3.1 Firebase Project Configuration

1. [Firebase Console](https://console.firebase.google.com) → Select/create project
2. **Project Settings** → **Cloud Messaging** tab
3. Add Android app → Enter package name from `app.json`
4. Download `google-services.json`
5. Enable Cloud Messaging API (Legacy) if using server key, or use FCM v1 HTTP API with OAuth 2.0

---

### 3.2 google-services.json

Place `google-services.json` in your project root and reference it in `app.json`:

```json
{
  "expo": {
    "android": {
      "package": "com.yourname.myapp",
      "googleServicesFile": "./google-services.json"
    },
    "plugins": [
      ["expo-notifications", {
        "icon": "./assets/notification-icon.png",
        "color": "#007AFF",
        "defaultChannel": "default",
        "sounds": ["./assets/notification-sound.wav"]
      }]
    ]
  }
}
```

> `google-services.json` does not contain secrets — it is safe to commit to version control. The FCM server key (for sending) must stay on your backend only.

---

### 3.3 FCM Server Key

The FCM server key is used by your **backend** to send notifications. Two auth methods:

**Legacy HTTP API (simpler but being deprecated):**
```
Project Settings → Cloud Messaging → Server key
Use in Authorization: key=<SERVER_KEY> header
```

**FCM v1 HTTP API (recommended, OAuth 2.0):**
```
Project Settings → Service Accounts → Generate new private key
Download service-account.json → use on your backend
```

```js
// Backend — FCM v1 API (Node.js)
import { GoogleAuth } from 'google-auth-library';
import axios from 'axios';

const auth = new GoogleAuth({
  keyFile: './service-account.json',
  scopes: ['https://www.googleapis.com/auth/firebase.messaging'],
});

async function sendFCMNotification(deviceToken, notification) {
  const client = await auth.getClient();
  const accessToken = await client.getAccessToken();

  await axios.post(
    `https://fcm.googleapis.com/v1/projects/YOUR_PROJECT_ID/messages:send`,
    {
      message: {
        token: deviceToken,
        notification: {
          title: notification.title,
          body: notification.body,
        },
        android: {
          channelId: 'orders',
          priority: 'high',
          notification: { sound: 'default', icon: 'notification_icon', color: '#007AFF' },
        },
        data: notification.data, // string key-value pairs only
      },
    },
    { headers: { Authorization: `Bearer ${accessToken.token}` } }
  );
}
```

---

### 3.4 Android 13+ POST_NOTIFICATIONS Permission

Android 13 (API 33+) requires runtime permission to show notifications — just like iOS.

```json
// app.json
{
  "expo": {
    "android": {
      "permissions": ["POST_NOTIFICATIONS"]
    }
  }
}
```

```tsx
import * as Notifications from 'expo-notifications';
import { Platform } from 'react-native';

async function requestNotificationPermission(): Promise<boolean> {
  if (Platform.OS === 'android' && Platform.Version < 33) {
    // Android < 13 — permission auto-granted, no request needed
    return true;
  }

  const { status: existing } = await Notifications.getPermissionsAsync();
  if (existing === 'granted') return true;

  const { status } = await Notifications.requestPermissionsAsync({
    ios: {
      allowAlert: true,
      allowBadge: true,
      allowSound: true,
      allowProvisional: true,      // iOS: provisional = quiet delivery to Notification Center
      allowCriticalAlerts: false,  // medical/safety apps only
    },
  });

  return status === 'granted';
}
```

---

## 4. Implementation

### 4.1 expo-notifications

The recommended approach for most Expo apps — wraps APNs and FCM behind a unified API.

```bash
npx expo install expo-notifications expo-device expo-constants
```

```tsx
// src/notifications/setup.ts
import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';
import Constants from 'expo-constants';
import { Platform } from 'react-native';

// Step 1 — configure foreground behavior
Notifications.setNotificationHandler({
  handleNotification: async (notification) => {
    const data = notification.request.content.data;

    return {
      shouldShowAlert: true,
      shouldPlaySound: data?.silent !== true,
      shouldSetBadge: true,
      priority: Notifications.AndroidNotificationPriority.HIGH,
    };
  },
});

// Step 2 — setup channels (Android)
async function setupChannels() {
  if (Platform.OS !== 'android') return;

  await Notifications.setNotificationChannelAsync('default', {
    name: 'Default',
    importance: Notifications.AndroidImportance.HIGH,
    sound: 'default',
    vibrationPattern: [0, 250, 250, 250],
    lightColor: '#007AFF',
  });

  await Notifications.setNotificationChannelAsync('orders', {
    name: 'Order Updates',
    importance: Notifications.AndroidImportance.HIGH,
    sound: 'default',
  });

  await Notifications.setNotificationChannelAsync('promotions', {
    name: 'Promotions',
    importance: Notifications.AndroidImportance.LOW,
  });
}

// Step 3 — request permission + get Expo push token
export async function registerForPushNotifications(): Promise<string | null> {
  if (!Device.isDevice) {
    console.warn('Push notifications require a physical device');
    return null;
  }

  await setupChannels();
  await setupNotificationCategories(); // see Section 1.5

  const granted = await requestNotificationPermission();
  if (!granted) return null;

  // Get Expo push token (works with Expo's push service)
  const projectId = Constants.expoConfig?.extra?.eas?.projectId
    ?? Constants.easConfig?.projectId;

  const { data: expoPushToken } = await Notifications.getExpoPushTokenAsync({
    projectId,
  });

  return expoPushToken;
  // Format: ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]
}

// Get native device token (for direct APNs/FCM, bypassing Expo)
export async function getNativeDeviceToken(): Promise<string | null> {
  const { data } = await Notifications.getDevicePushTokenAsync();
  return data; // raw APNs token (iOS) or FCM registration token (Android)
}
```

---

### 4.2 @react-native-firebase/messaging

Use Firebase Messaging directly for more control, FCM-specific features, and when already using Firebase.

```bash
npx expo install @react-native-firebase/app @react-native-firebase/messaging
```

```tsx
import messaging from '@react-native-firebase/messaging';
import { Platform } from 'react-native';

// Request permission (iOS)
async function requestFirebasePermission(): Promise<boolean> {
  const authStatus = await messaging().requestPermission({
    alert: true,
    badge: true,
    sound: true,
    provisional: true,
  });

  return (
    authStatus === messaging.AuthorizationStatus.AUTHORIZED ||
    authStatus === messaging.AuthorizationStatus.PROVISIONAL
  );
}

// Get FCM token
async function getFCMToken(): Promise<string | null> {
  const granted = await requestFirebasePermission();
  if (!granted) return null;

  const token = await messaging().getToken();
  return token;
}

// Check APNs token (iOS) — Firebase needs this internally
async function checkAPNsToken() {
  if (Platform.OS !== 'ios') return;
  const apnsToken = await messaging().getAPNSToken();
  if (!apnsToken) {
    console.warn('APNs token not available — push may not work on iOS');
  }
}
```

---

### 4.3 Device Token Registration

Register the device token with your backend after obtaining it.

```tsx
// src/notifications/tokenManager.ts
import * as Notifications from 'expo-notifications';
import * as SecureStore from 'expo-secure-store';
import { registerForPushNotifications } from './setup';
import apiClient from '@/lib/apiClient';

const TOKEN_KEY = 'push-token';

export async function registerDeviceToken(userId: string): Promise<void> {
  const token = await registerForPushNotifications();
  if (!token) return;

  // Check if token has changed since last registration
  const savedToken = await SecureStore.getItemAsync(TOKEN_KEY);
  if (savedToken === token) return; // no change — skip API call

  try {
    await apiClient.post('/devices/register', {
      userId,
      token,
      platform: Platform.OS,          // 'ios' | 'android'
      appVersion: Application.nativeApplicationVersion,
      deviceModel: Device.modelName,
      osVersion: Device.osVersion,
    });

    // Save locally to detect future changes
    await SecureStore.setItemAsync(TOKEN_KEY, token);
  } catch (error) {
    console.error('Failed to register device token:', error);
    // Don't throw — non-critical, will retry on next launch
  }
}

// Deregister on logout — stop receiving notifications
export async function deregisterDeviceToken(userId: string): Promise<void> {
  const token = await SecureStore.getItemAsync(TOKEN_KEY);
  if (!token) return;

  try {
    await apiClient.delete('/devices/register', { data: { userId, token } });
    await SecureStore.deleteItemAsync(TOKEN_KEY);
  } catch {
    // Best effort
  }
}
```

**Call after login:**
```tsx
// app/(auth)/login.tsx — after successful login
const onSubmit = async (data: LoginForm) => {
  await login(data.email, data.password);
  await registerDeviceToken(user.id); // register token for this user
  router.replace('/(app)/(tabs)/');
};
```

---

### 4.4 Token Refresh Handling

Tokens can change — when the user reinstalls the app, restores from backup, or FCM rotates the token.

```tsx
// src/notifications/tokenManager.ts
import * as Notifications from 'expo-notifications';

export function setupTokenRefreshListener(userId: string): () => void {
  // expo-notifications: Expo push token change
  const subscription = Notifications.addPushTokenListener(async ({ data: newToken }) => {
    console.log('Push token refreshed:', newToken);

    try {
      const oldToken = await SecureStore.getItemAsync(TOKEN_KEY);

      await apiClient.put('/devices/token', {
        userId,
        oldToken,
        newToken,
        platform: Platform.OS,
      });

      await SecureStore.setItemAsync(TOKEN_KEY, newToken);
    } catch (error) {
      console.error('Failed to update refreshed token:', error);
    }
  });

  return () => subscription.remove();
}

// Firebase Messaging: FCM token refresh
import messaging from '@react-native-firebase/messaging';

messaging().onTokenRefresh(async (newToken) => {
  console.log('FCM token refreshed:', newToken);
  const userId = useAuthStore.getState().user?.id;
  if (userId) await updateTokenOnServer(userId, newToken);
});
```

**Setup in root layout:**
```tsx
// app/(app)/_layout.tsx — runs only when logged in
export default function AppLayout() {
  const user = useAuthStore(s => s.user);

  useEffect(() => {
    if (!user) return;

    registerDeviceToken(user.id);
    const cleanup = setupTokenRefreshListener(user.id);
    return cleanup;
  }, [user?.id]);

  return <Stack />;
}
```

---

### 4.5 Sending to Backend

Your backend stores tokens and uses them to send notifications. Here's the pattern:

```
User logs in → app sends token to backend
    ↓
Backend stores: { userId, token, platform, createdAt }
    ↓
Event occurs (new message, order update)
    ↓
Backend looks up tokens for targetUserId
    ↓
Backend sends via Expo Push API or directly to APNs/FCM
    ↓
Device receives notification
```

**Using Expo's Push API (simplest — handles APNs + FCM routing):**
```js
// backend (Node.js)
import { Expo } from 'expo-server-sdk';

const expo = new Expo({ accessToken: process.env.EXPO_ACCESS_TOKEN });

async function sendPushNotification(expoPushToken, { title, body, data, channelId }) {
  if (!Expo.isExpoPushToken(expoPushToken)) {
    console.error('Invalid Expo push token:', expoPushToken);
    return;
  }

  const messages = [{
    to: expoPushToken,
    title,
    body,
    data,
    sound: 'default',
    badge: 1,
    channelId: channelId ?? 'default', // Android channel
    categoryIdentifier: data?.category, // iOS action buttons
  }];

  const chunks = expo.chunkPushNotifications(messages);
  const tickets = [];

  for (const chunk of chunks) {
    const ticketChunk = await expo.sendPushNotificationsAsync(chunk);
    tickets.push(...ticketChunk);
  }

  // Check receipts after ~15 min to detect invalid tokens
  const receiptIds = tickets.filter(t => t.id).map(t => t.id);
  // Store receiptIds for later receipt checking
  return tickets;
}

// Send to multiple users
async function sendToUsers(userIds, notification) {
  const tokens = await db.getTokensForUsers(userIds);
  await Promise.all(tokens.map(t => sendPushNotification(t, notification)));
}
```

---

## 5. Notification Handling

### 5.1 Receiving in Foreground

When the app is active, `setNotificationHandler` decides display, and `addNotificationReceivedListener` lets you react to it.

```tsx
// src/notifications/handlers.ts
import * as Notifications from 'expo-notifications';
import { useEffect, useRef } from 'react';

// Global foreground display config — called before any component mounts
Notifications.setNotificationHandler({
  handleNotification: async (notification) => {
    const { data } = notification.request.content;

    // Don't show banner for silent/data-only notifications
    if (data?.type === 'silent') {
      return { shouldShowAlert: false, shouldPlaySound: false, shouldSetBadge: false };
    }

    // Suppress if user is already on the target screen
    const currentRoute = getCurrentRoute(); // your navigation helper
    if (data?.screen && currentRoute === data.screen) {
      return { shouldShowAlert: false, shouldPlaySound: false, shouldSetBadge: false };
    }

    return {
      shouldShowAlert: true,
      shouldPlaySound: true,
      shouldSetBadge: true,
      priority: Notifications.AndroidNotificationPriority.HIGH,
    };
  },
});

// Hook to handle foreground notification events
export function useForegroundNotifications() {
  const notificationListener = useRef<Notifications.EventSubscription>();

  useEffect(() => {
    notificationListener.current = Notifications.addNotificationReceivedListener(
      (notification) => {
        const { title, body, data } = notification.request.content;
        console.log('[Foreground notification]', { title, body, data });

        // Update badge, refresh data, show in-app toast, etc.
        if (data?.type === 'new_message') {
          useMessageStore.getState().incrementUnread();
        }
        if (data?.type === 'order_update') {
          queryClient.invalidateQueries({ queryKey: ['orders', data.orderId] });
        }
      }
    );

    return () => notificationListener.current?.remove();
  }, []);
}
```

---

### 5.2 Receiving in Background

Background handlers must be registered at the **top level of your entry file**, not inside a component.

**With expo-notifications:**
```tsx
// app/_layout.tsx — module level (outside the component)
import * as TaskManager from 'expo-task-manager';

const BACKGROUND_NOTIFICATION_TASK = 'background-notification-task';

TaskManager.defineTask(BACKGROUND_NOTIFICATION_TASK, ({ data, error, executionInfo }) => {
  if (error) { console.error(error); return; }

  const notification = data?.notification as Notifications.Notification;
  const { type, orderId } = notification.request.content.data ?? {};

  console.log('[Background notification]', type, orderId);

  // Limited what you can do here:
  // ✅ Update local DB / MMKV
  // ✅ Set badge count
  // ❌ Navigate (no active UI)
  // ❌ Show alerts
});

// Register the task
Notifications.registerTaskAsync(BACKGROUND_NOTIFICATION_TASK);
```

**With Firebase Messaging (more reliable for background):**
```tsx
// Register at module level — outside components
import messaging from '@react-native-firebase/messaging';

// Background message handler — fires when app is background OR killed
messaging().setBackgroundMessageHandler(async (remoteMessage) => {
  console.log('[FCM Background]', remoteMessage.data);

  // Update local storage
  const { type, orderId } = remoteMessage.data ?? {};
  if (type === 'order_update') {
    await updateOrderInLocalDB(orderId);
  }
});
```

---

### 5.3 Tap-to-Open Deep Linking

When user taps a notification, navigate to the relevant screen.

```tsx
// src/notifications/deepLink.ts
import * as Notifications from 'expo-notifications';
import { router } from 'expo-router';

type NotificationData = {
  screen?: string;
  orderId?: string;
  userId?: string;
  productId?: string;
};

function navigateFromNotification(data: NotificationData) {
  if (!data?.screen) return;

  switch (data.screen) {
    case 'order':
      router.push(`/orders/${data.orderId}`);
      break;
    case 'message':
      router.push(`/messages/${data.userId}`);
      break;
    case 'product':
      router.push(`/product/${data.productId}`);
      break;
    default:
      router.push(data.screen);
  }
}

// Hook to set up tap listeners
export function useNotificationNavigation() {
  const responseListener = useRef<Notifications.EventSubscription>();
  const navigationReady = useRef(false);

  // Wait for navigation to be ready before handling taps
  const onNavigationReady = useCallback(() => {
    navigationReady.current = true;
  }, []);

  useEffect(() => {
    // Handle tap when app is FOREGROUND or BACKGROUND
    responseListener.current = Notifications.addNotificationResponseReceivedListener(
      (response) => {
        const data = response.notification.request.content.data as NotificationData;
        const actionIdentifier = response.actionIdentifier;

        if (actionIdentifier === Notifications.DEFAULT_ACTION_IDENTIFIER) {
          // User tapped the notification itself
          navigateFromNotification(data);
        } else {
          // User tapped an action button (iOS category actions)
          handleNotificationAction(actionIdentifier, data);
        }
      }
    );

    // Handle tap when app was KILLED (cold start)
    Notifications.getLastNotificationResponseAsync().then((response) => {
      if (response) {
        const data = response.notification.request.content.data as NotificationData;
        // Small delay to ensure navigation is ready
        setTimeout(() => navigateFromNotification(data), 100);
      }
    });

    return () => responseListener.current?.remove();
  }, []);
}

function handleNotificationAction(actionId: string, data: NotificationData) {
  switch (actionId) {
    case 'track':
      router.push(`/orders/${data.orderId}`);
      break;
    case 'cancel':
      cancelOrder(data.orderId!);
      break;
    case 'reply':
      // Text input reply — response.userText contains the typed reply
      break;
    case 'mark_read':
      markMessageRead(data.userId!);
      break;
  }
}
```

**Wire up in root layout:**
```tsx
// app/_layout.tsx
export default function RootLayout() {
  useNotificationNavigation(); // handles all tap scenarios

  return <Stack />;
}
```

---

### 5.4 Notification Payload Parsing

The payload structure differs slightly between Expo Push and raw APNs/FCM. Normalize it.

```tsx
// src/notifications/payloadParser.ts

// Expo push notification content shape
type ExpoNotificationContent = {
  title?: string;
  body?: string;
  data?: Record<string, unknown>;
  badge?: number;
  sound?: string;
};

// Firebase RemoteMessage shape
type FCMPayload = {
  notification?: { title?: string; body?: string };
  data?: Record<string, string>; // FCM data is always string values
};

// Normalized internal shape
type ParsedNotification = {
  title: string;
  body: string;
  type: string;
  screen?: string;
  payload: Record<string, unknown>;
};

export function parseExpoNotification(
  content: ExpoNotificationContent
): ParsedNotification {
  return {
    title: content.title ?? '',
    body: content.body ?? '',
    type: (content.data?.type as string) ?? 'generic',
    screen: content.data?.screen as string | undefined,
    payload: content.data ?? {},
  };
}

export function parseFCMMessage(message: FCMPayload): ParsedNotification {
  // FCM data values are strings — parse JSON if needed
  const data: Record<string, unknown> = {};
  for (const [key, value] of Object.entries(message.data ?? {})) {
    try { data[key] = JSON.parse(value); }
    catch { data[key] = value; }
  }

  return {
    title: message.notification?.title ?? '',
    body: message.notification?.body ?? '',
    type: (data.type as string) ?? 'generic',
    screen: data.screen as string | undefined,
    payload: data,
  };
}
```

---

### 5.5 Badge Count Management

```tsx
import * as Notifications from 'expo-notifications';

// Set badge count
await Notifications.setBadgeCountAsync(5);

// Get current badge count
const count = await Notifications.getBadgeCountAsync();

// Clear badge (set to 0)
await Notifications.setBadgeCountAsync(0);

// Common badge management patterns
export const badgeManager = {
  set: (count: number) => Notifications.setBadgeCountAsync(Math.max(0, count)),
  clear: () => Notifications.setBadgeCountAsync(0),
  get: () => Notifications.getBadgeCountAsync(),
  increment: async () => {
    const current = await Notifications.getBadgeCountAsync();
    await Notifications.setBadgeCountAsync(current + 1);
  },
  decrement: async () => {
    const current = await Notifications.getBadgeCountAsync();
    await Notifications.setBadgeCountAsync(Math.max(0, current - 1));
  },
};

// Clear badge when user opens the app
useFocusEffect(
  useCallback(() => {
    badgeManager.clear();
  }, [])
);

// Sync badge with unread count from server
useEffect(() => {
  if (unreadCount !== undefined) {
    badgeManager.set(unreadCount);
  }
}, [unreadCount]);
```

---

### 5.6 Rich Notifications (Images, Actions)

**Image attachments:**
```js
// Backend — Expo push with image
{
  to: 'ExponentPushToken[xxx]',
  title: 'New Product',
  body: 'Nike Air Max just dropped!',
  data: { productId: '123', screen: '/product/123' },
  richContent: {
    image: 'https://example.com/product-image.jpg', // Expo handles download
  },
}

// APNs raw — image via mutable-content + service extension
{
  aps: {
    alert: { title: 'New Product', body: 'Nike Air Max just dropped!' },
    'mutable-content': 1, // tells iOS to run Notification Service Extension
  },
  imageUrl: 'https://example.com/product-image.jpg',
}
```

**Action buttons via categories (iOS):**
```tsx
// Categories defined in Section 1.5
// Backend sends categoryIdentifier in payload

// Expo push with category
{
  to: 'ExponentPushToken[xxx]',
  title: 'Order Shipped',
  body: 'Your order is on the way!',
  data: { orderId: '456', screen: 'order' },
  categoryIdentifier: 'order', // triggers Track Order / Cancel Order buttons
}
```

**Android action buttons via notification channels and BigPicture style:**
```tsx
// Via expo-notifications local notification with actions
await Notifications.scheduleNotificationAsync({
  content: {
    title: 'New Message',
    body: 'Hey, are you free tonight?',
    categoryIdentifier: 'message',
    data: { userId: '789' },
  },
  trigger: null, // show immediately
});

// Android expanded BigPicture style
// Requires a custom FCM payload processed by a native module
// For complex Android rich notifications, use @notifee/react-native
```

**@notifee/react-native for advanced Android notifications:**
```bash
npx expo install @notifee/react-native
```

```tsx
import notifee, { AndroidStyle, AndroidImportance } from '@notifee/react-native';

await notifee.displayNotification({
  title: 'New Product',
  body: 'Nike Air Max just dropped!',
  android: {
    channelId: 'orders',
    style: {
      type: AndroidStyle.BIGPICTURE,
      picture: 'https://example.com/product-image.jpg',
    },
    actions: [
      { title: 'Buy Now', pressAction: { id: 'buy' } },
      { title: 'View', pressAction: { id: 'view', launchActivity: 'default' } },
    ],
    importance: AndroidImportance.HIGH,
  },
});
```

---

### 5.7 Silent / Data-Only Notifications

Silent notifications carry data without showing any UI — used to trigger background syncs, badge updates, or cache invalidation.

**APNs silent push (content-available):**
```js
// Backend — APNs payload
{
  aps: {
    'content-available': 1,   // triggers background fetch, NO banner
    // Note: do NOT include alert — that would make it visible
  },
  type: 'sync',
  timestamp: '1714000000',
}
```

**FCM data-only message:**
```js
// Backend — FCM, no notification key = data-only = silent
{
  to: deviceToken,
  data: {                        // all strings
    type: 'sync',
    entity: 'products',
    timestamp: '1714000000',
  },
  // No 'notification' key = silent on Android
  android: { priority: 'high' }, // needed to wake device
}
```

**Handling silent notifications in background:**
```tsx
// Firebase — fires for data-only messages in background and killed state
messaging().setBackgroundMessageHandler(async (remoteMessage) => {
  const { type, entity } = remoteMessage.data ?? {};

  if (type === 'sync') {
    await syncManager.sync(entity ?? 'all');
  }
  if (type === 'badge_reset') {
    await Notifications.setBadgeCountAsync(0);
  }
});

// Foreground — addNotificationReceivedListener fires for all
Notifications.addNotificationReceivedListener((notification) => {
  if (notification.request.content.data?.type === 'sync') {
    syncManager.sync('products');
  }
});
```

> On iOS, `content-available` background wakes are heavily throttled by the OS. Apple limits how often the app is woken — don't rely on them for time-critical operations.

---

## 6. Local Notifications

### 6.1 Scheduling

Schedule a notification to appear at a future time — no server required.

```tsx
import * as Notifications from 'expo-notifications';

// Show immediately
const notifId = await Notifications.scheduleNotificationAsync({
  content: {
    title: 'Order Delivered! 🎉',
    body: 'Your order #12345 has been delivered.',
    data: { screen: 'order', orderId: '12345' },
    sound: 'default',
    badge: 1,
  },
  trigger: null, // null = show immediately
});

// Show after a delay (seconds)
await Notifications.scheduleNotificationAsync({
  content: {
    title: 'Don\'t forget!',
    body: 'Your cart has items waiting.',
    data: { screen: '/(app)/(tabs)/' },
  },
  trigger: {
    type: Notifications.SchedulableTriggerInputTypes.TIME_INTERVAL,
    seconds: 3600, // 1 hour from now
    repeats: false,
  },
});

// Show at a specific date/time
await Notifications.scheduleNotificationAsync({
  content: {
    title: 'Appointment Reminder',
    body: 'Your appointment is in 30 minutes.',
    sound: 'default',
  },
  trigger: {
    type: Notifications.SchedulableTriggerInputTypes.DATE,
    date: new Date('2026-04-25T09:30:00'), // exact date and time
  },
});

// Show at a specific calendar time (daily at 8 AM)
await Notifications.scheduleNotificationAsync({
  content: {
    title: 'Good morning!',
    body: 'Check today\'s deals.',
  },
  trigger: {
    type: Notifications.SchedulableTriggerInputTypes.CALENDAR,
    hour: 8,
    minute: 0,
    repeats: true, // fires every day at 8:00 AM
  },
});
```

---

### 6.2 Recurring Notifications

```tsx
// Daily reminder at a specific time
async function scheduleDailyReminder(hour: number, minute: number, message: string) {
  // Cancel existing before re-scheduling to avoid duplicates
  await cancelNotificationByTag('daily-reminder');

  return Notifications.scheduleNotificationAsync({
    content: {
      title: 'Daily Reminder',
      body: message,
      data: { tag: 'daily-reminder' },
    },
    trigger: {
      type: Notifications.SchedulableTriggerInputTypes.CALENDAR,
      hour,
      minute,
      repeats: true,
    },
  });
}

// Weekly on a specific day
await Notifications.scheduleNotificationAsync({
  content: { title: 'Weekly Review', body: 'Time for your weekly check-in!' },
  trigger: {
    type: Notifications.SchedulableTriggerInputTypes.CALENDAR,
    weekday: 2,    // 1 = Sunday, 2 = Monday ... 7 = Saturday
    hour: 9,
    minute: 0,
    repeats: true,
  },
});

// Every N seconds (useful for testing, not for production timers)
await Notifications.scheduleNotificationAsync({
  content: { title: 'Interval', body: 'Every 5 minutes' },
  trigger: {
    type: Notifications.SchedulableTriggerInputTypes.TIME_INTERVAL,
    seconds: 300,  // 5 minutes
    repeats: true,
  },
});

// Monthly on a specific day
await Notifications.scheduleNotificationAsync({
  content: { title: 'Monthly Report', body: 'Your monthly summary is ready.' },
  trigger: {
    type: Notifications.SchedulableTriggerInputTypes.CALENDAR,
    day: 1,       // 1st of every month
    hour: 10,
    minute: 0,
    repeats: true,
  },
});
```

---

### 6.3 Cancelling Scheduled Notifications

```tsx
import * as Notifications from 'expo-notifications';

// Cancel a specific notification by ID (returned from scheduleNotificationAsync)
await Notifications.cancelScheduledNotificationAsync(notifId);

// Cancel all scheduled notifications
await Notifications.cancelAllScheduledNotificationsAsync();

// List all scheduled notifications
const scheduled = await Notifications.getAllScheduledNotificationsAsync();
console.log(`${scheduled.length} notifications scheduled`);
scheduled.forEach(n => {
  console.log(n.identifier, n.content.title, n.trigger);
});

// Cancel by tag (store ID when scheduling, use data field as identifier)
const cancelNotificationByTag = async (tag: string) => {
  const all = await Notifications.getAllScheduledNotificationsAsync();
  const matching = all.filter(n => n.content.data?.tag === tag);
  await Promise.all(
    matching.map(n => Notifications.cancelScheduledNotificationAsync(n.identifier))
  );
};

// Practical pattern — schedule/cancel reminders for specific events
export const reminderManager = {
  scheduleForOrder: async (orderId: string, deliveryDate: Date) => {
    // Cancel any existing reminder for this order
    await cancelNotificationByTag(`order-${orderId}`);

    const reminderTime = new Date(deliveryDate.getTime() - 30 * 60 * 1000); // 30 min before
    if (reminderTime <= new Date()) return; // past — don't schedule

    return Notifications.scheduleNotificationAsync({
      content: {
        title: 'Delivery Soon!',
        body: 'Your order arrives in about 30 minutes.',
        data: {
          tag: `order-${orderId}`,
          screen: 'order',
          orderId,
        },
        sound: 'default',
      },
      trigger: {
        type: Notifications.SchedulableTriggerInputTypes.DATE,
        date: reminderTime,
      },
    });
  },

  cancelForOrder: async (orderId: string) => {
    await cancelNotificationByTag(`order-${orderId}`);
  },

  cancelAll: () => Notifications.cancelAllScheduledNotificationsAsync(),

  getAll: () => Notifications.getAllScheduledNotificationsAsync(),
};
```

---

*End of Module 11*
