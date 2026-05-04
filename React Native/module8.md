# Module 8 — Native APIs & Device Features

> React Native apps run on real hardware. This module covers how to request permissions, access the camera, file system, and the full suite of device APIs available through Expo's SDK.

---

## Table of Contents

1. [Permissions Handling](#1-permissions-handling)
   - 1.1 [expo-permissions Patterns](#11-expo-permissions-patterns)
   - 1.2 [react-native-permissions](#12-react-native-permissions)
   - 1.3 [iOS Info.plist Entries](#13-ios-infoplist-entries)
   - 1.4 [Android Manifest Permissions](#14-android-manifest-permissions)
   - 1.5 [Runtime Permission Requests](#15-runtime-permission-requests)
   - 1.6 [Permission Rationale UX](#16-permission-rationale-ux)
   - 1.7 [Blocked Permission Recovery](#17-blocked-permission-recovery)

2. [Camera](#2-camera)
   - 2.1 [expo-camera](#21-expo-camera)
   - 2.2 [Photo Capture](#22-photo-capture)
   - 2.3 [Video Recording](#23-video-recording)
   - 2.4 [Camera Permissions](#24-camera-permissions)
   - 2.5 [Front/Back Camera Switching](#25-frontback-camera-switching)
   - 2.6 [Flash Control](#26-flash-control)

3. [Gallery & Image Picker](#3-gallery--image-picker)
   - 3.1 [expo-image-picker](#31-expo-image-picker)
   - 3.2 [Single and Multi-Select](#32-single-and-multi-select)
   - 3.3 [Image Cropping](#33-image-cropping)
   - 3.4 [Video Selection](#34-video-selection)
   - 3.5 [MediaLibrary API](#35-medialibrary-api)

4. [File System](#4-file-system)
   - 4.1 [expo-file-system](#41-expo-file-system)
   - 4.2 [Reading and Writing Files](#42-reading-and-writing-files)
   - 4.3 [File URIs vs Paths](#43-file-uris-vs-paths)
   - 4.4 [Caching Directories](#44-caching-directories)
   - 4.5 [Document Directories](#45-document-directories)

5. [File Upload](#5-file-upload)
   - 5.1 [Multipart Form Data](#51-multipart-form-data)
   - 5.2 [FormData Construction](#52-formdata-construction)
   - 5.3 [Progress Tracking](#53-progress-tracking)
   - 5.4 [Resumable Uploads](#54-resumable-uploads)
   - 5.5 [Background Uploads](#55-background-uploads)

6. [Other Native APIs](#6-other-native-apis)
   - 6.1 [Geolocation (expo-location)](#61-geolocation-expo-location)
   - 6.2 [Contacts](#62-contacts)
   - 6.3 [Calendar](#63-calendar)
   - 6.4 [Clipboard](#64-clipboard)
   - 6.5 [Haptics](#65-haptics)
   - 6.6 [Linking](#66-linking)
   - 6.7 [Share API](#67-share-api)
   - 6.8 [Biometrics](#68-biometrics-face-id--touch-id--fingerprint)
   - 6.9 [Network Info](#69-network-info)
   - 6.10 [Battery and Device Info](#610-battery-and-device-info)

---

## 1. Permissions Handling

### 1.1 expo-permissions Patterns

Modern Expo apps request permissions through individual SDK modules (e.g., `expo-camera` for camera permission, `expo-location` for location). The old standalone `expo-permissions` package is deprecated — each module now exposes its own `requestPermissionsAsync` and `getPermissionsAsync`.

**The universal pattern every permission follows:**

```tsx
import * as Camera from 'expo-camera';

async function requestCameraPermission() {
  // 1. Check current status first
  const { status: existing } = await Camera.getCameraPermissionsAsync();
  if (existing === 'granted') return true;

  // 2. Request only if not yet granted
  const { status } = await Camera.requestCameraPermissionsAsync();
  return status === 'granted';
}
```

**Permission status values:**

| Status | Meaning |
|---|---|
| `'granted'` | User approved — proceed |
| `'denied'` | User denied — show recovery UI |
| `'undetermined'` | Not asked yet — safe to request |
| `'limited'` | iOS only — partial access (e.g., selected photos) |

---

### 1.2 react-native-permissions

For apps needing fine-grained control or permissions not covered by Expo modules (Bluetooth, NFC, health data), use `react-native-permissions`.

```bash
npx expo install react-native-permissions
```

```tsx
import { check, request, PERMISSIONS, RESULTS, openSettings } from 'react-native-permissions';
import { Platform } from 'react-native';

const LOCATION_PERMISSION = Platform.select({
  ios: PERMISSIONS.IOS.LOCATION_WHEN_IN_USE,
  android: PERMISSIONS.ANDROID.ACCESS_FINE_LOCATION,
})!;

async function handleLocationPermission() {
  const result = await check(LOCATION_PERMISSION);

  switch (result) {
    case RESULTS.GRANTED:
      return true;

    case RESULTS.DENIED:
      // Not yet asked — safe to request
      const requestResult = await request(LOCATION_PERMISSION);
      return requestResult === RESULTS.GRANTED;

    case RESULTS.BLOCKED:
      // User denied and checked "Don't ask again"
      // Must redirect to settings
      await openSettings();
      return false;

    case RESULTS.UNAVAILABLE:
      // Feature not available on this device/OS
      Alert.alert('Not supported', 'Location is not available on this device.');
      return false;
  }
}
```

---

### 1.3 iOS Info.plist Entries

iOS requires a **usage description** string for every permission in `Info.plist`. Without it, the app crashes when requesting the permission.

In Expo, these are configured in `app.json` under `expo.ios.infoPlist`:

```json
{
  "expo": {
    "ios": {
      "infoPlist": {
        "NSCameraUsageDescription": "We need camera access to let you take photos for your profile.",
        "NSMicrophoneUsageDescription": "We need microphone access to record video.",
        "NSPhotoLibraryUsageDescription": "We need photo library access to let you upload images.",
        "NSPhotoLibraryAddUsageDescription": "We need permission to save photos to your library.",
        "NSLocationWhenInUseUsageDescription": "We use your location to show nearby stores.",
        "NSLocationAlwaysUsageDescription": "We use your background location to send delivery updates.",
        "NSContactsUsageDescription": "We access your contacts to help you invite friends.",
        "NSFaceIDUsageDescription": "We use Face ID to securely log you in.",
        "NSCalendarsUsageDescription": "We add delivery reminders to your calendar.",
        "NSBluetoothAlwaysUsageDescription": "We use Bluetooth to connect to nearby devices."
      }
    }
  }
}
```

> The description string is shown in the system permission dialog. Make it clear and honest about **why** you need the permission — vague strings get rejected by Apple App Review.

---

### 1.4 Android Manifest Permissions

Android permissions are declared in `AndroidManifest.xml`. In Expo, configure them in `app.json`:

```json
{
  "expo": {
    "android": {
      "permissions": [
        "CAMERA",
        "RECORD_AUDIO",
        "READ_MEDIA_IMAGES",
        "READ_MEDIA_VIDEO",
        "READ_EXTERNAL_STORAGE",
        "WRITE_EXTERNAL_STORAGE",
        "ACCESS_FINE_LOCATION",
        "ACCESS_COARSE_LOCATION",
        "ACCESS_BACKGROUND_LOCATION",
        "READ_CONTACTS",
        "VIBRATE",
        "USE_BIOMETRIC",
        "USE_FINGERPRINT"
      ]
    }
  }
}
```

**Dangerous permissions (require runtime request) vs Normal permissions (auto-granted):**

| Type | Examples | Request needed? |
|---|---|---|
| Normal | VIBRATE, INTERNET, BLUETOOTH | No — auto-granted on install |
| Dangerous | CAMERA, LOCATION, CONTACTS, MICROPHONE | Yes — runtime prompt |

> On Android 13+ (API 33), `READ_EXTERNAL_STORAGE` is replaced by granular media permissions: `READ_MEDIA_IMAGES`, `READ_MEDIA_VIDEO`, `READ_MEDIA_AUDIO`.

---

### 1.5 Runtime Permission Requests

**Generic reusable hook for any Expo permission:**

```tsx
// src/hooks/usePermission.ts
import { useState, useCallback } from 'react';

type PermissionHook = {
  status: string | null;
  request: () => Promise<boolean>;
  isGranted: boolean;
  isBlocked: boolean;
};

type PermissionModule = {
  getPermissionsAsync: () => Promise<{ status: string }>;
  requestPermissionsAsync: () => Promise<{ status: string }>;
};

export function usePermission(module: PermissionModule): PermissionHook {
  const [status, setStatus] = useState<string | null>(null);

  const request = useCallback(async (): Promise<boolean> => {
    const { status: current } = await module.getPermissionsAsync();
    if (current === 'granted') {
      setStatus('granted');
      return true;
    }

    const { status: requested } = await module.requestPermissionsAsync();
    setStatus(requested);
    return requested === 'granted';
  }, [module]);

  return {
    status,
    request,
    isGranted: status === 'granted',
    isBlocked: status === 'denied',
  };
}

// Usage
import * as Camera from 'expo-camera';

const { isGranted, isBlocked, request } = usePermission(Camera);
```

---

### 1.6 Permission Rationale UX

Show a pre-prompt explanation **before** the system dialog — users who understand why grant more often. On Android, use `shouldShowRequestPermissionRationale` to detect if this is needed.

```tsx
import { Platform, Alert } from 'react-native';
import * as Location from 'expo-location';
import { check, PERMISSIONS, RESULTS } from 'react-native-permissions';

async function requestLocationWithRationale(): Promise<boolean> {
  // On Android, check if we should show rationale
  if (Platform.OS === 'android') {
    const permission = PERMISSIONS.ANDROID.ACCESS_FINE_LOCATION;
    const status = await check(permission);

    if (status === RESULTS.DENIED) {
      // Show rationale before the system dialog
      await new Promise<void>(resolve =>
        Alert.alert(
          'Location Access',
          'We use your location to show nearby pickup points and estimate delivery times. Your location is never stored or shared.',
          [
            { text: 'Not Now', style: 'cancel', onPress: resolve },
            { text: 'Continue', onPress: resolve },
          ]
        )
      );
    }
  }

  const { status } = await Location.requestForegroundPermissionsAsync();
  return status === 'granted';
}
```

**Permission gate component — show custom UI before requesting:**
```tsx
import { View, Text, Pressable, StyleSheet } from 'react-native';
import { Ionicons } from '@expo/vector-icons';

type Props = {
  icon: keyof typeof Ionicons.glyphMap;
  title: string;
  description: string;
  onRequest: () => void;
};

export function PermissionGate({ icon, title, description, onRequest }: Props) {
  return (
    <View style={styles.container}>
      <Ionicons name={icon} size={64} color="#007AFF" />
      <Text style={styles.title}>{title}</Text>
      <Text style={styles.description}>{description}</Text>
      <Pressable style={styles.button} onPress={onRequest}>
        <Text style={styles.buttonText}>Allow Access</Text>
      </Pressable>
    </View>
  );
}

// Usage
function CameraScreen() {
  const [hasPermission, setHasPermission] = useState<boolean | null>(null);

  if (hasPermission === null) {
    return (
      <PermissionGate
        icon="camera-outline"
        title="Camera Access"
        description="Allow camera access to scan QR codes and take profile photos."
        onRequest={async () => {
          const granted = await requestCameraPermission();
          setHasPermission(granted);
        }}
      />
    );
  }

  if (!hasPermission) return <BlockedPermissionScreen setting="Camera" />;
  return <CameraView />;
}
```

---

### 1.7 Blocked Permission Recovery

When a permission is permanently denied, the OS won't show the dialog again. You must redirect the user to Settings.

```tsx
import { Linking, Alert, Platform } from 'react-native';
import { openSettings } from 'react-native-permissions';

function BlockedPermissionScreen({ setting }: { setting: string }) {
  const handleOpenSettings = () => {
    Alert.alert(
      `${setting} Permission Required`,
      `Please enable ${setting} access in Settings to use this feature.`,
      [
        { text: 'Cancel', style: 'cancel' },
        {
          text: 'Open Settings',
          onPress: () => {
            // expo-linking or react-native-permissions
            openSettings().catch(() => {
              // Fallback for some Android devices
              Linking.openURL(
                Platform.OS === 'ios'
                  ? 'app-settings:'
                  : 'android.settings.APPLICATION_DETAILS_SETTINGS'
              );
            });
          },
        },
      ]
    );
  };

  return (
    <View style={styles.container}>
      <Ionicons name="lock-closed-outline" size={56} color="#ef4444" />
      <Text style={styles.title}>{setting} Access Blocked</Text>
      <Text style={styles.message}>
        You've previously denied {setting} access. Open Settings to enable it.
      </Text>
      <Pressable style={styles.settingsButton} onPress={handleOpenSettings}>
        <Text style={styles.settingsButtonText}>Open Settings</Text>
      </Pressable>
    </View>
  );
}
```

---

## 2. Camera

### 2.1 expo-camera

```bash
npx expo install expo-camera
```

```tsx
import { CameraView, CameraType, useCameraPermissions } from 'expo-camera';
import { useRef, useState } from 'react';
import { View, Text, Pressable, StyleSheet } from 'react-native';

export default function CameraScreen() {
  const [permission, requestPermission] = useCameraPermissions();
  const [facing, setFacing] = useState<CameraType>('back');
  const cameraRef = useRef<CameraView>(null);

  if (!permission) return <View />; // permissions loading

  if (!permission.granted) {
    return (
      <View style={styles.permissionContainer}>
        <Text>Camera access is required.</Text>
        <Pressable onPress={requestPermission} style={styles.button}>
          <Text style={styles.buttonText}>Grant Permission</Text>
        </Pressable>
      </View>
    );
  }

  return (
    <View style={{ flex: 1 }}>
      <CameraView
        ref={cameraRef}
        style={{ flex: 1 }}
        facing={facing}
      />
      {/* controls rendered on top */}
    </View>
  );
}
```

---

### 2.2 Photo Capture

```tsx
import { CameraView } from 'expo-camera';
import { useRef, useState } from 'react';
import { View, Image, Pressable, Text, ActivityIndicator, StyleSheet } from 'react-native';

export default function PhotoCaptureScreen() {
  const cameraRef = useRef<CameraView>(null);
  const [photo, setPhoto] = useState<string | null>(null);
  const [isCapturing, setIsCapturing] = useState(false);

  const takePhoto = async () => {
    if (!cameraRef.current || isCapturing) return;
    setIsCapturing(true);

    try {
      const result = await cameraRef.current.takePictureAsync({
        quality: 0.8,          // 0–1 compression quality
        base64: false,         // true = include base64 string (heavy)
        exif: false,           // true = include EXIF metadata
        skipProcessing: false, // true = faster but less optimized
        imageType: 'jpg',      // 'jpg' | 'png'
      });

      setPhoto(result.uri);
    } finally {
      setIsCapturing(false);
    }
  };

  if (photo) {
    return (
      <View style={{ flex: 1 }}>
        <Image source={{ uri: photo }} style={{ flex: 1 }} resizeMode="contain" />
        <View style={styles.actions}>
          <Pressable style={styles.retakeButton} onPress={() => setPhoto(null)}>
            <Text>Retake</Text>
          </Pressable>
          <Pressable style={styles.useButton} onPress={() => usePhoto(photo)}>
            <Text style={{ color: '#fff' }}>Use Photo</Text>
          </Pressable>
        </View>
      </View>
    );
  }

  return (
    <View style={{ flex: 1 }}>
      <CameraView ref={cameraRef} style={{ flex: 1 }} />
      <View style={styles.controls}>
        <Pressable
          onPress={takePhoto}
          disabled={isCapturing}
          style={styles.shutter}
        >
          {isCapturing ? <ActivityIndicator color="#000" /> : <View style={styles.shutterInner} />}
        </Pressable>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  controls: { position: 'absolute', bottom: 40, width: '100%', alignItems: 'center' },
  shutter: { width: 72, height: 72, borderRadius: 36, backgroundColor: '#fff', justifyContent: 'center', alignItems: 'center' },
  shutterInner: { width: 60, height: 60, borderRadius: 30, backgroundColor: '#fff', borderWidth: 2, borderColor: '#000' },
  actions: { flexDirection: 'row', padding: 20, gap: 12 },
  retakeButton: { flex: 1, padding: 14, borderWidth: 1, borderRadius: 8, alignItems: 'center' },
  useButton: { flex: 1, padding: 14, backgroundColor: '#007AFF', borderRadius: 8, alignItems: 'center' },
});
```

---

### 2.3 Video Recording

```tsx
import { CameraView } from 'expo-camera';
import { useRef, useState } from 'react';
import { View, Pressable, Text, StyleSheet } from 'react-native';

export default function VideoRecordScreen() {
  const cameraRef = useRef<CameraView>(null);
  const [isRecording, setIsRecording] = useState(false);
  const [videoUri, setVideoUri] = useState<string | null>(null);

  const startRecording = async () => {
    if (!cameraRef.current) return;
    setIsRecording(true);

    try {
      const video = await cameraRef.current.recordAsync({
        maxDuration: 60,         // max seconds (0 = unlimited)
        maxFileSize: 50 * 1024 * 1024, // 50 MB limit
        mute: false,
      });
      setVideoUri(video.uri);
    } finally {
      setIsRecording(false);
    }
  };

  const stopRecording = () => {
    cameraRef.current?.stopRecording();
    // recordAsync promise resolves when stopRecording is called
  };

  return (
    <View style={{ flex: 1 }}>
      <CameraView
        ref={cameraRef}
        style={{ flex: 1 }}
        mode="video"                   // required for video recording
        videoQuality="1080p"           // '2160p' | '1080p' | '720p' | '480p'
      />
      <View style={styles.controls}>
        {isRecording && <View style={styles.recordingDot} />}
        <Pressable
          onPress={isRecording ? stopRecording : startRecording}
          style={[styles.recordButton, isRecording && styles.recordButtonActive]}
        >
          <View style={[styles.recordIcon, isRecording && styles.stopIcon]} />
        </Pressable>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  controls: { position: 'absolute', bottom: 40, width: '100%', alignItems: 'center' },
  recordButton: { width: 72, height: 72, borderRadius: 36, borderWidth: 4, borderColor: '#fff', justifyContent: 'center', alignItems: 'center' },
  recordButtonActive: { borderColor: '#ef4444' },
  recordIcon: { width: 52, height: 52, borderRadius: 26, backgroundColor: '#ef4444' },
  stopIcon: { width: 28, height: 28, borderRadius: 4, backgroundColor: '#ef4444' },
  recordingDot: { width: 12, height: 12, borderRadius: 6, backgroundColor: '#ef4444', marginBottom: 12 },
});
```

---

### 2.4 Camera Permissions

```tsx
import { useCameraPermissions, useMicrophonePermissions } from 'expo-camera';

export function useCameraAndMicPermissions() {
  const [cameraPermission, requestCameraPermission] = useCameraPermissions();
  const [micPermission, requestMicPermission] = useMicrophonePermissions();

  const requestAll = async () => {
    const [cam, mic] = await Promise.all([
      requestCameraPermission(),
      requestMicPermission(),
    ]);
    return cam.granted && mic.granted;
  };

  return {
    hasCameraPermission: cameraPermission?.granted ?? false,
    hasMicPermission: micPermission?.granted ?? false,
    hasAllPermissions: (cameraPermission?.granted && micPermission?.granted) ?? false,
    requestAll,
  };
}
```

---

### 2.5 Front/Back Camera Switching

```tsx
import { CameraType } from 'expo-camera';
import { useState } from 'react';
import { Pressable } from 'react-native';
import { Ionicons } from '@expo/vector-icons';

export default function CameraWithFlip() {
  const [facing, setFacing] = useState<CameraType>('back');

  const toggleFacing = () => {
    setFacing(prev => prev === 'back' ? 'front' : 'back');
  };

  return (
    <View style={{ flex: 1 }}>
      <CameraView style={{ flex: 1 }} facing={facing} />

      <Pressable
        onPress={toggleFacing}
        style={styles.flipButton}
      >
        <Ionicons name="camera-reverse-outline" size={28} color="#fff" />
      </Pressable>
    </View>
  );
}
```

---

### 2.6 Flash Control

```tsx
import { FlashMode } from 'expo-camera';
import { useState } from 'react';
import { Pressable } from 'react-native';
import { Ionicons } from '@expo/vector-icons';

const flashModes: FlashMode[] = ['off', 'on', 'auto'];
const flashIcons: Record<FlashMode, keyof typeof Ionicons.glyphMap> = {
  off: 'flash-off-outline',
  on: 'flash-outline',
  auto: 'flash-outline',   // auto shows a different color
};

export default function CameraWithFlash() {
  const [flashMode, setFlashMode] = useState<FlashMode>('off');

  const cycleFlash = () => {
    setFlashMode(prev => {
      const idx = flashModes.indexOf(prev);
      return flashModes[(idx + 1) % flashModes.length];
    });
  };

  return (
    <View style={{ flex: 1 }}>
      <CameraView
        style={{ flex: 1 }}
        flash={flashMode}
        enableTorch={flashMode === 'on'}  // torch = continuous light
      />

      <Pressable onPress={cycleFlash} style={styles.flashButton}>
        <Ionicons
          name={flashIcons[flashMode]}
          size={24}
          color={flashMode === 'auto' ? '#facc15' : '#fff'}
        />
        <Text style={{ color: '#fff', fontSize: 11 }}>{flashMode.toUpperCase()}</Text>
      </Pressable>
    </View>
  );
}
```

---

## 3. Gallery & Image Picker

### 3.1 expo-image-picker

The standard way to access the device photo library or launch the camera from within the app.

```bash
npx expo install expo-image-picker
```

```tsx
import * as ImagePicker from 'expo-image-picker';
```

---

### 3.2 Single and Multi-Select

```tsx
import * as ImagePicker from 'expo-image-picker';
import { Image, Pressable, Text, View, FlatList } from 'react-native';
import { useState } from 'react';

export default function ImagePickerScreen() {
  const [images, setImages] = useState<ImagePicker.ImagePickerAsset[]>([]);

  const pickSingle = async () => {
    const result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ImagePicker.MediaTypeOptions.Images,
      quality: 0.8,
      allowsEditing: true,      // enable crop/resize after selection
      aspect: [1, 1],           // crop to 1:1 (only when allowsEditing = true)
    });

    if (!result.canceled) {
      const asset = result.assets[0];
      // asset.uri, asset.width, asset.height, asset.fileSize, asset.type
      setImages([asset]);
    }
  };

  const pickMultiple = async () => {
    const result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ImagePicker.MediaTypeOptions.Images,
      allowsMultipleSelection: true,   // iOS 14+, Android
      selectionLimit: 10,              // max selectable
      quality: 0.8,
      orderedSelection: true,          // preserve selection order (iOS)
    });

    if (!result.canceled) {
      setImages(result.assets);
    }
  };

  return (
    <View>
      <Pressable onPress={pickSingle} style={styles.button}>
        <Text>Pick Single Photo</Text>
      </Pressable>
      <Pressable onPress={pickMultiple} style={styles.button}>
        <Text>Pick Multiple Photos</Text>
      </Pressable>

      <FlatList
        data={images}
        numColumns={3}
        keyExtractor={item => item.uri}
        renderItem={({ item }) => (
          <Image source={{ uri: item.uri }} style={styles.thumbnail} />
        )}
      />
    </View>
  );
}
```

**Asset properties:**
```tsx
const asset: ImagePicker.ImagePickerAsset = result.assets[0];

asset.uri        // local file URI
asset.width      // pixel width
asset.height     // pixel height
asset.fileSize   // bytes (may be undefined)
asset.type       // 'image' | 'video'
asset.fileName   // original filename (may be undefined on some Android)
asset.mimeType   // 'image/jpeg' | 'image/png' | etc.
asset.base64     // base64 string (only if base64: true was passed)
asset.duration   // video duration in ms (only for videos)
```

---

### 3.3 Image Cropping

`allowsEditing: true` shows the system crop UI after selection.

```tsx
// Square avatar crop
const result = await ImagePicker.launchImageLibraryAsync({
  mediaTypes: ImagePicker.MediaTypeOptions.Images,
  allowsEditing: true,
  aspect: [1, 1],       // [width, height] ratio
  quality: 1,           // no compression on crop result
});

// 16:9 banner crop
const result = await ImagePicker.launchImageLibraryAsync({
  allowsEditing: true,
  aspect: [16, 9],
  quality: 0.9,
});

// Free-form crop (no aspect constraint)
const result = await ImagePicker.launchImageLibraryAsync({
  allowsEditing: true,
  // omit aspect — user can crop freely
  quality: 0.8,
});
```

> For advanced cropping (rotate, zoom, pan, multiple aspect ratios), use `expo-image-manipulator` to post-process, or `react-native-image-crop-picker` as an alternative library.

---

### 3.4 Video Selection

```tsx
const result = await ImagePicker.launchImageLibraryAsync({
  mediaTypes: ImagePicker.MediaTypeOptions.Videos,  // videos only
  quality: ImagePicker.UIImagePickerControllerQualityType.Medium,
  videoMaxDuration: 60,   // seconds — 0 = no limit
});

// For both photos and videos
const result = await ImagePicker.launchImageLibraryAsync({
  mediaTypes: ImagePicker.MediaTypeOptions.All,
});

if (!result.canceled) {
  const asset = result.assets[0];
  if (asset.type === 'video') {
    console.log('Video URI:', asset.uri);
    console.log('Duration:', asset.duration, 'ms');
  }
}
```

---

### 3.5 MediaLibrary API

Read and write to the device's media library directly — save photos, create albums, query assets.

```bash
npx expo install expo-media-library
```

```tsx
import * as MediaLibrary from 'expo-media-library';

// ─── Save photo to gallery ──────────────────────────────────────────────────
async function saveToGallery(uri: string) {
  const { status } = await MediaLibrary.requestPermissionsAsync();
  if (status !== 'granted') return;

  const asset = await MediaLibrary.createAssetAsync(uri);

  // Optional: save to a specific album
  const album = await MediaLibrary.getAlbumAsync('MyApp');
  if (album) {
    await MediaLibrary.addAssetsToAlbumAsync([asset], album, false);
  } else {
    await MediaLibrary.createAlbumAsync('MyApp', asset, false);
  }
}

// ─── Query assets ──────────────────────────────────────────────────────────
async function getRecentPhotos() {
  const { status } = await MediaLibrary.requestPermissionsAsync();
  if (status !== 'granted') return [];

  const { assets } = await MediaLibrary.getAssetsAsync({
    mediaType: MediaLibrary.MediaType.photo,
    sortBy: MediaLibrary.SortBy.creationTime,
    first: 20,           // page size
  });

  return assets;
  // asset.uri, asset.filename, asset.width, asset.height, asset.creationTime
}

// ─── Pagination ────────────────────────────────────────────────────────────
async function getPaginatedPhotos(after?: string) {
  const { assets, hasNextPage, endCursor } = await MediaLibrary.getAssetsAsync({
    mediaType: 'photo',
    first: 30,
    after,             // cursor from previous page's endCursor
  });

  return { assets, hasNextPage, cursor: endCursor };
}
```

---

## 4. File System

### 4.1 expo-file-system

Provides access to the local file system for reading, writing, downloading, and uploading files.

```bash
npx expo install expo-file-system
```

```tsx
import * as FileSystem from 'expo-file-system';
```

---

### 4.2 Reading and Writing Files

```tsx
import * as FileSystem from 'expo-file-system';

const filePath = FileSystem.documentDirectory + 'user-data.json';

// ─── Write ─────────────────────────────────────────────────────────────────
await FileSystem.writeAsStringAsync(
  filePath,
  JSON.stringify({ name: 'Nawaz', score: 100 }),
  { encoding: FileSystem.EncodingType.UTF8 }
);

// ─── Read ──────────────────────────────────────────────────────────────────
const content = await FileSystem.readAsStringAsync(filePath, {
  encoding: FileSystem.EncodingType.UTF8,
});
const data = JSON.parse(content);

// ─── Binary read (Base64) ──────────────────────────────────────────────────
const base64 = await FileSystem.readAsStringAsync(imagePath, {
  encoding: FileSystem.EncodingType.Base64,
});

// ─── Check if file exists ──────────────────────────────────────────────────
const info = await FileSystem.getInfoAsync(filePath);
if (info.exists) {
  console.log('Size:', info.size, 'bytes');
  console.log('Modified:', new Date(info.modificationTime * 1000));
}

// ─── Delete ────────────────────────────────────────────────────────────────
await FileSystem.deleteAsync(filePath, { idempotent: true }); // idempotent = no error if missing

// ─── Copy and Move ─────────────────────────────────────────────────────────
await FileSystem.copyAsync({ from: sourcePath, to: destPath });
await FileSystem.moveAsync({ from: sourcePath, to: destPath });

// ─── Create directory ──────────────────────────────────────────────────────
await FileSystem.makeDirectoryAsync(FileSystem.documentDirectory + 'uploads/', {
  intermediates: true, // create parent dirs if needed
});

// ─── List directory contents ───────────────────────────────────────────────
const files = await FileSystem.readDirectoryAsync(FileSystem.documentDirectory!);
console.log(files); // ['user-data.json', 'uploads/']
```

---

### 4.3 File URIs vs Paths

React Native uses **file URIs** (prefixed with `file://`), not raw paths.

```tsx
// URI — use with Image, Video, upload functions
const uri = 'file:///var/mobile/Containers/Data/.../Documents/photo.jpg';

// Path — strip the prefix for Node.js-style operations (rare in Expo)
const path = uri.replace('file://', '');

// expo-file-system always returns URIs
console.log(FileSystem.documentDirectory);
// → 'file:///var/mobile/Containers/Data/.../Documents/'

console.log(FileSystem.cacheDirectory);
// → 'file:///var/mobile/Containers/Data/.../Library/Caches/'

// Always join with the directory URI
const fileUri = FileSystem.documentDirectory + 'filename.json';

// Download a file and get its local URI
const { uri: localUri } = await FileSystem.downloadAsync(
  'https://example.com/data.json',
  FileSystem.cacheDirectory + 'data.json'
);
// localUri is a file:// URI ready to use
```

---

### 4.4 Caching Directories

`cacheDirectory` is for temporary files — the OS may delete them when storage is low.

```tsx
import * as FileSystem from 'expo-file-system';

// ─── Download and cache a remote file ─────────────────────────────────────
async function getCachedFile(url: string): Promise<string> {
  const filename = url.split('/').pop()!;
  const cachedUri = FileSystem.cacheDirectory + filename;

  const info = await FileSystem.getInfoAsync(cachedUri);
  if (info.exists) {
    return cachedUri; // serve from cache
  }

  // Not cached yet — download it
  const { uri } = await FileSystem.downloadAsync(url, cachedUri);
  return uri;
}

// ─── Clear cache ───────────────────────────────────────────────────────────
async function clearCache() {
  const cacheDir = FileSystem.cacheDirectory!;
  const files = await FileSystem.readDirectoryAsync(cacheDir);
  await Promise.all(
    files.map(file => FileSystem.deleteAsync(cacheDir + file, { idempotent: true }))
  );
}

// ─── Check cache size ──────────────────────────────────────────────────────
async function getCacheSize(): Promise<number> {
  const cacheDir = FileSystem.cacheDirectory!;
  const files = await FileSystem.readDirectoryAsync(cacheDir);
  const sizes = await Promise.all(
    files.map(async file => {
      const info = await FileSystem.getInfoAsync(cacheDir + file);
      return info.exists && 'size' in info ? info.size : 0;
    })
  );
  return sizes.reduce((sum, size) => sum + size, 0);
}
```

---

### 4.5 Document Directories

`documentDirectory` is for user-generated or persistent data that should survive app restarts. This directory is included in iCloud/Google Drive backups by default.

```tsx
// Recommended structure for document directory
const DIRS = {
  documents: FileSystem.documentDirectory!,
  uploads: FileSystem.documentDirectory + 'uploads/',
  exports: FileSystem.documentDirectory + 'exports/',
  db: FileSystem.documentDirectory + 'SQLite/',
};

// Ensure directories exist on app launch
async function ensureDirectories() {
  await Promise.all(
    Object.values(DIRS).map(dir =>
      FileSystem.makeDirectoryAsync(dir, { intermediates: true })
        .catch(() => {}) // ignore if already exists
    )
  );
}

// Sharing a document — make it accessible to other apps
import * as Sharing from 'expo-sharing';

async function shareFile(fileUri: string) {
  const isAvailable = await Sharing.isAvailableAsync();
  if (!isAvailable) {
    Alert.alert('Sharing not available on this device');
    return;
  }
  await Sharing.shareAsync(fileUri, {
    mimeType: 'application/pdf',
    dialogTitle: 'Share your report',
  });
}
```

---

## 5. File Upload

### 5.1 Multipart Form Data

The HTTP standard for uploading files — each part of the request body contains a file or field.

```
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="userId"

user-123
------WebKitFormBoundary
Content-Disposition: form-data; name="photo"; filename="avatar.jpg"
Content-Type: image/jpeg

[binary data]
------WebKitFormBoundary--
```

---

### 5.2 FormData Construction

```tsx
import apiClient from '@/lib/apiClient';
import * as FileSystem from 'expo-file-system';

async function uploadAvatar(imageUri: string, userId: string) {
  // Get file info for mime type
  const filename = imageUri.split('/').pop() ?? 'photo.jpg';
  const extension = filename.split('.').pop()?.toLowerCase() ?? 'jpg';
  const mimeType = extension === 'png' ? 'image/png' : 'image/jpeg';

  // Build FormData — React Native's FormData accepts uri directly
  const formData = new FormData();

  formData.append('userId', userId);
  formData.append('photo', {
    uri: imageUri,       // file:// URI
    name: filename,      // filename the server receives
    type: mimeType,      // MIME type
  } as any);             // RN's FormData type is not fully typed

  // Extra metadata
  formData.append('timestamp', new Date().toISOString());

  const response = await apiClient.post('/users/avatar', formData, {
    headers: {
      'Content-Type': 'multipart/form-data', // axios sets boundary automatically
    },
  });

  return response.data;
}
```

---

### 5.3 Progress Tracking

Use `expo-file-system`'s `uploadAsync` for progress events:

```tsx
import * as FileSystem from 'expo-file-system';
import { useState } from 'react';
import { View, Text, StyleSheet } from 'react-native';

export function useFileUpload() {
  const [progress, setProgress] = useState(0);
  const [isUploading, setIsUploading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const upload = async (fileUri: string, uploadUrl: string) => {
    setIsUploading(true);
    setProgress(0);
    setError(null);

    try {
      const result = await FileSystem.uploadAsync(uploadUrl, fileUri, {
        httpMethod: 'POST',
        uploadType: FileSystem.FileSystemUploadType.MULTIPART,
        fieldName: 'file',               // form field name
        mimeType: 'image/jpeg',
        parameters: { userId: 'user-123' }, // extra form fields
        headers: { Authorization: `Bearer ${getToken()}` },

        sessionType: FileSystem.FileSystemSessionType.BACKGROUND, // survive app background

        // Progress callback
        uploadProgressCallback: (written, total) => {
          setProgress(Math.round((written / total) * 100));
        },
      });

      if (result.status !== 200) throw new Error('Upload failed');
      return JSON.parse(result.body);
    } catch (err: any) {
      setError(err.message);
      throw err;
    } finally {
      setIsUploading(false);
    }
  };

  return { upload, progress, isUploading, error };
}

// ─── Progress bar component ────────────────────────────────────────────────
function UploadProgressBar({ progress }: { progress: number }) {
  return (
    <View style={styles.track}>
      <View style={[styles.fill, { width: `${progress}%` }]} />
      <Text style={styles.label}>{progress}%</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  track: { height: 8, backgroundColor: '#e5e5e5', borderRadius: 4, overflow: 'hidden', marginVertical: 8 },
  fill: { height: '100%', backgroundColor: '#007AFF', borderRadius: 4 },
  label: { textAlign: 'center', fontSize: 12, color: '#666', marginTop: 4 },
});
```

---

### 5.4 Resumable Uploads

For large files, resumable uploads continue from where they stopped after a network interruption.

```tsx
import * as FileSystem from 'expo-file-system';
import AsyncStorage from '@react-native-async-storage/async-storage';

const UPLOAD_STATE_KEY = 'resumable_upload_state';

async function resumableUpload(fileUri: string, uploadUrl: string) {
  // Check if a previous upload was interrupted
  const savedState = await AsyncStorage.getItem(UPLOAD_STATE_KEY);
  const resumeData = savedState ? JSON.parse(savedState) : null;

  const uploadTask = new FileSystem.UploadTask(
    uploadUrl,
    fileUri,
    {
      httpMethod: 'POST',
      uploadType: FileSystem.FileSystemUploadType.BINARY_CONTENT,
      sessionType: FileSystem.FileSystemSessionType.BACKGROUND,
      headers: { Authorization: `Bearer ${getToken()}` },
    },
    (written, total) => {
      const pct = Math.round((written / total) * 100);
      console.log(`Upload progress: ${pct}%`);
    }
  );

  // Persist upload task ID so we can resume if interrupted
  uploadTask.addProgressListener(() => {
    // Save task reference for resume
  });

  const result = await uploadTask.uploadAsync();
  await AsyncStorage.removeItem(UPLOAD_STATE_KEY); // clear saved state on success
  return result;
}
```

---

### 5.5 Background Uploads

Uploads that continue even when the user puts the app in the background.

```tsx
import * as FileSystem from 'expo-file-system';
import * as BackgroundFetch from 'expo-background-fetch';
import * as TaskManager from 'expo-task-manager';

const UPLOAD_TASK = 'background-upload-task';

// Register the background task
TaskManager.defineTask(UPLOAD_TASK, async ({ data, error }) => {
  if (error) {
    console.error(error);
    return BackgroundFetch.BackgroundFetchResult.Failed;
  }

  // data contains the upload queue from AsyncStorage
  const queue = await getUploadQueue();
  for (const item of queue) {
    await FileSystem.uploadAsync(item.url, item.uri, {
      httpMethod: 'POST',
      uploadType: FileSystem.FileSystemUploadType.MULTIPART,
      sessionType: FileSystem.FileSystemSessionType.BACKGROUND, // ← key setting
    });
    await removeFromQueue(item.id);
  }

  return BackgroundFetch.BackgroundFetchResult.NewData;
});

// Register for background fetch
await BackgroundFetch.registerTaskAsync(UPLOAD_TASK, {
  minimumInterval: 15 * 60, // minimum 15 minutes on iOS
  stopOnTerminate: false,
  startOnBoot: true,
});
```

> `FileSystemSessionType.BACKGROUND` is the critical setting — it uses `NSURLSession` (iOS) and `WorkManager` (Android) to handle the upload outside your JS thread.

---

## 6. Other Native APIs

### 6.1 Geolocation (expo-location)

```bash
npx expo install expo-location
```

```tsx
import * as Location from 'expo-location';

// One-time current location
async function getCurrentPosition() {
  const { status } = await Location.requestForegroundPermissionsAsync();
  if (status !== 'granted') return null;

  const location = await Location.getCurrentPositionAsync({
    accuracy: Location.Accuracy.Balanced,
    // Accuracy levels: Lowest | Low | Balanced | High | Highest | BestForNavigation
  });

  return {
    latitude: location.coords.latitude,
    longitude: location.coords.longitude,
    altitude: location.coords.altitude,
    accuracy: location.coords.accuracy, // meters
    speed: location.coords.speed,       // m/s
  };
}

// Continuous location tracking
async function watchLocation() {
  const { status } = await Location.requestForegroundPermissionsAsync();
  if (status !== 'granted') return;

  const subscription = await Location.watchPositionAsync(
    { accuracy: Location.Accuracy.High, distanceInterval: 10 }, // update every 10m
    (location) => {
      console.log('New position:', location.coords);
    }
  );

  // Stop tracking
  return () => subscription.remove();
}

// Reverse geocoding — lat/lng → address
async function reverseGeocode(latitude: number, longitude: number) {
  const [address] = await Location.reverseGeocodeAsync({ latitude, longitude });
  return `${address.street}, ${address.city}, ${address.country}`;
}

// Forward geocoding — address → lat/lng
async function geocode(address: string) {
  const results = await Location.geocodeAsync(address);
  return results[0]; // { latitude, longitude, accuracy, altitude }
}
```

**Background location** (requires additional permission):
```tsx
await Location.requestBackgroundPermissionsAsync();

await Location.startLocationUpdatesAsync('background-location-task', {
  accuracy: Location.Accuracy.Balanced,
  distanceInterval: 100,
  showsBackgroundLocationIndicator: true, // iOS blue bar
  foregroundService: {                    // Android foreground service
    notificationTitle: 'Tracking your route',
    notificationBody: 'Tap to open the app',
  },
});
```

---

### 6.2 Contacts

```bash
npx expo install expo-contacts
```

```tsx
import * as Contacts from 'expo-contacts';

async function getContacts() {
  const { status } = await Contacts.requestPermissionsAsync();
  if (status !== 'granted') return [];

  const { data } = await Contacts.getContactsAsync({
    fields: [
      Contacts.Fields.Name,
      Contacts.Fields.PhoneNumbers,
      Contacts.Fields.Emails,
      Contacts.Fields.Image,
    ],
    sort: Contacts.SortTypes.FirstName,
  });

  return data;
}

// Search contacts
const { data } = await Contacts.getContactsAsync({
  name: 'Nawaz', // partial match
});

// Get a single contact by ID
const contact = await Contacts.getContactByIdAsync(contactId, [
  Contacts.Fields.PhoneNumbers,
  Contacts.Fields.Emails,
]);

// Add a new contact
await Contacts.addContactAsync({
  name: 'John Doe',
  phoneNumbers: [{ label: 'mobile', number: '+1234567890' }],
  emails: [{ label: 'work', email: 'john@example.com' }],
});
```

---

### 6.3 Calendar

```bash
npx expo install expo-calendar
```

```tsx
import * as Calendar from 'expo-calendar';

async function addEventToCalendar() {
  const { status } = await Calendar.requestCalendarPermissionsAsync();
  if (status !== 'granted') return;

  // Get available calendars
  const calendars = await Calendar.getCalendarsAsync(Calendar.EntityTypes.EVENT);
  const defaultCalendar = calendars.find(c => c.allowsModifications);

  if (!defaultCalendar) return;

  const eventId = await Calendar.createEventAsync(defaultCalendar.id, {
    title: 'Order Delivery',
    startDate: new Date('2026-04-25T10:00:00'),
    endDate: new Date('2026-04-25T11:00:00'),
    location: '123 Main St, Mumbai',
    notes: 'Order #12345 delivery window',
    alarms: [{ relativeOffset: -60 }], // 60 min before
    timeZone: 'Asia/Kolkata',
  });

  Alert.alert('Added to Calendar', `Event ID: ${eventId}`);
}

// Remove event
await Calendar.deleteEventAsync(eventId);
```

---

### 6.4 Clipboard

```bash
npx expo install expo-clipboard
```

```tsx
import * as Clipboard from 'expo-clipboard';
import { Alert } from 'react-native';

// Write to clipboard
async function copyToClipboard(text: string) {
  await Clipboard.setStringAsync(text);
  Alert.alert('Copied!');  // user feedback is important
}

// Read from clipboard
async function pasteFromClipboard(): Promise<string> {
  return Clipboard.getStringAsync();
}

// Listen for clipboard changes (iOS 14+ requires permission)
const subscription = Clipboard.addClipboardListener(({ contentTypes }) => {
  if (contentTypes.includes(Clipboard.ContentType.PLAIN_TEXT)) {
    Clipboard.getStringAsync().then(text => console.log('Clipboard changed:', text));
  }
});

// Copy image URL as image (base64)
await Clipboard.setImageAsync(base64Image);
const hasImage = await Clipboard.hasImageAsync();

// Cleanup
subscription.remove();
```

---

### 6.5 Haptics

Provide tactile feedback. Use sparingly — excessive haptics feel annoying.

```bash
npx expo install expo-haptics
```

```tsx
import * as Haptics from 'expo-haptics';

// Impact feedback — physical tap-like vibration
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Heavy);
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Rigid);
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Soft);

// Notification feedback — communicates success/warning/error
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Warning);
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Error);

// Selection feedback — subtle, for picker/selector changes
await Haptics.selectionAsync();

// Usage patterns
const onAddToCart = async () => {
  await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
  addToCart(product);
};

const onDeleteItem = async () => {
  await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
  deleteItem(id);
};

const onPickerChange = async () => {
  await Haptics.selectionAsync(); // subtle feedback on scroll
};
```

---

### 6.6 Linking

Open URLs, email, phone, SMS, and deep links to other apps.

```tsx
import { Linking } from 'react-native';
import * as ExpoLinking from 'expo-linking';

// Open URL in browser
await Linking.openURL('https://example.com');

// Open with check first (safer)
const canOpen = await Linking.canOpenURL('https://example.com');
if (canOpen) await Linking.openURL('https://example.com');

// Phone call
await Linking.openURL('tel:+919876543210');

// Send SMS
await Linking.openURL('sms:+919876543210?body=Hello%20there');

// Send email
await Linking.openURL('mailto:support@example.com?subject=Help&body=I%20need%20help');

// Open device Settings
await Linking.openSettings();

// Open another app (if installed) — deep link
await Linking.openURL('instagram://user?username=nawaz');
await Linking.openURL('whatsapp://send?phone=+919876543210&text=Hello');

// Open Maps
await Linking.openURL(`maps:?q=${lat},${lng}`);              // iOS
await Linking.openURL(`geo:${lat},${lng}?q=${lat},${lng}`);  // Android

// Cross-platform maps
const scheme = Platform.select({ ios: 'maps:', android: 'geo:' });
await Linking.openURL(`${scheme}?q=${encodeURIComponent(address)}`);

// Listen for incoming deep links (cold start)
const initialUrl = await Linking.getInitialURL();

// Listen for deep links while app is running
const subscription = Linking.addEventListener('url', ({ url }) => {
  const { hostname, path, queryParams } = ExpoLinking.parse(url);
  console.log('Incoming link:', { hostname, path, queryParams });
});
```

---

### 6.7 Share API

Trigger the native share sheet to share text, URLs, or files.

```tsx
import { Share } from 'react-native';
import * as Sharing from 'expo-sharing';

// Share text or URL — uses native share sheet
const shareText = async () => {
  const result = await Share.share({
    title: 'Check this out',
    message: 'I found this awesome product: https://example.com/product/123',
    url: 'https://example.com/product/123', // iOS — separate URL field
  });

  if (result.action === Share.sharedAction) {
    if (result.activityType) {
      console.log('Shared via:', result.activityType);
    }
  } else if (result.action === Share.dismissedAction) {
    console.log('Share dismissed');
  }
};

// Share a file — PDF, image, etc.
const shareFile = async (fileUri: string) => {
  const isAvailable = await Sharing.isAvailableAsync();
  if (!isAvailable) return;

  await Sharing.shareAsync(fileUri, {
    mimeType: 'application/pdf',
    dialogTitle: 'Share your invoice',   // Android only
    UTI: 'com.adobe.pdf',               // iOS Universal Type Identifier
  });
};
```

---

### 6.8 Biometrics (Face ID / Touch ID / Fingerprint)

```bash
npx expo install expo-local-authentication
```

```tsx
import * as LocalAuthentication from 'expo-local-authentication';

async function authenticateWithBiometrics(): Promise<boolean> {
  // Check hardware availability
  const isAvailable = await LocalAuthentication.hasHardwareAsync();
  if (!isAvailable) return false;

  // Check enrolled biometrics
  const isEnrolled = await LocalAuthentication.isEnrolledAsync();
  if (!isEnrolled) {
    Alert.alert('No biometrics enrolled', 'Please set up Face ID or fingerprint in Settings.');
    return false;
  }

  // What biometric types are available?
  const types = await LocalAuthentication.supportedAuthenticationTypesAsync();
  // [1 = Fingerprint, 2 = Face, 3 = Iris]
  const hasFaceId = types.includes(LocalAuthentication.AuthenticationType.FACIAL_RECOGNITION);

  // Authenticate
  const result = await LocalAuthentication.authenticateAsync({
    promptMessage: hasFaceId ? 'Authenticate with Face ID' : 'Authenticate with fingerprint',
    fallbackLabel: 'Use Passcode',          // iOS: label for the fallback button
    disableDeviceFallback: false,           // false = allow PIN/password fallback
    cancelLabel: 'Cancel',
  });

  return result.success;
}

// Usage — secure action gate
const handleDeleteAccount = async () => {
  const authenticated = await authenticateWithBiometrics();
  if (!authenticated) {
    Alert.alert('Authentication required', 'Biometric authentication failed.');
    return;
  }
  await api.deleteAccount();
};
```

**`app.json` for Face ID on iOS:**
```json
{
  "expo": {
    "ios": {
      "infoPlist": {
        "NSFaceIDUsageDescription": "We use Face ID to securely verify your identity."
      }
    }
  }
}
```

---

### 6.9 Network Info

```bash
npx expo install @react-native-community/netinfo
```

```tsx
import NetInfo, { NetInfoState } from '@react-native-community/netinfo';
import { useEffect, useState } from 'react';

// One-time check
const state: NetInfoState = await NetInfo.fetch();
console.log('Connected:', state.isConnected);
console.log('Connection type:', state.type); // 'wifi' | 'cellular' | 'none' | 'bluetooth' | ...
console.log('Is WiFi:', state.type === 'wifi');
console.log('Is metered:', state.details?.isConnectionExpensive); // cellular = true

// Hook for continuous monitoring
export function useNetworkInfo() {
  const [networkState, setNetworkState] = useState<NetInfoState | null>(null);

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener(setNetworkState);
    return unsubscribe;
  }, []);

  return {
    isConnected: networkState?.isConnected ?? true,
    isWifi: networkState?.type === 'wifi',
    isCellular: networkState?.type === 'cellular',
    isExpensive: networkState?.details?.isConnectionExpensive ?? false,
    type: networkState?.type,
  };
}

// Usage — show offline banner
export function OfflineBanner() {
  const { isConnected } = useNetworkInfo();

  if (isConnected) return null;

  return (
    <View style={{ backgroundColor: '#ef4444', padding: 8, alignItems: 'center' }}>
      <Text style={{ color: '#fff', fontWeight: '600' }}>No Internet Connection</Text>
    </View>
  );
}
```

---

### 6.10 Battery and Device Info

```bash
npx expo install expo-battery expo-device expo-application expo-constants
```

```tsx
import * as Battery from 'expo-battery';
import * as Device from 'expo-device';
import * as Application from 'expo-application';
import Constants from 'expo-constants';

// ─── Battery ───────────────────────────────────────────────────────────────
const batteryLevel = await Battery.getBatteryLevelAsync(); // 0.0 – 1.0
const batteryState = await Battery.getBatteryStateAsync();
// BatteryState: UNKNOWN | UNPLUGGED | CHARGING | FULL

const isLowPower = await Battery.isLowPowerModeEnabledAsync(); // iOS Low Power Mode

// Subscribe to battery changes
const subscription = Battery.addBatteryLevelListener(({ batteryLevel }) => {
  if (batteryLevel < 0.2) Alert.alert('Low Battery', 'Please charge your device.');
});

// ─── Device info ───────────────────────────────────────────────────────────
Device.brand;           // 'Apple' | 'Samsung' | 'Google' | ...
Device.manufacturer;    // 'Apple Inc.' | 'Samsung Electronics' | ...
Device.modelName;       // 'iPhone 15 Pro' | 'Pixel 8' | ...
Device.osName;          // 'iOS' | 'Android'
Device.osVersion;       // '17.0' | '14'
Device.totalMemory;     // RAM in bytes
Device.isDevice;        // false on simulators/emulators

Device.deviceType;
// DeviceType: UNKNOWN | PHONE | TABLET | DESKTOP | TV

// ─── App info ──────────────────────────────────────────────────────────────
Application.applicationId;              // 'com.yourname.myapp'
Application.nativeApplicationVersion;  // '1.2.3' (CFBundleShortVersionString)
Application.nativeBuildVersion;        // '45' (CFBundleVersion / versionCode)

// ─── Expo config ───────────────────────────────────────────────────────────
Constants.expoConfig?.name;            // App name from app.json
Constants.expoConfig?.version;         // Version from app.json
Constants.sessionId;                   // Unique ID per app session
Constants.platform?.ios?.buildNumber;  // iOS build number

// ─── Practical use: include device info in crash reports ───────────────────
function getDeviceContext() {
  return {
    model: Device.modelName,
    os: `${Device.osName} ${Device.osVersion}`,
    appVersion: Application.nativeApplicationVersion,
    buildNumber: Application.nativeBuildVersion,
    isDevice: Device.isDevice,
  };
}
```

---

*End of Module 8*
