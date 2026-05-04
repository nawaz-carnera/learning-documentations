# React Native Learning Notes

A structured collection of React Native study notes covering fundamentals to advanced topics.

---

## Table of Contents

| # | Module | Topics Covered |
|---|--------|----------------|
| 1 | [Module 1 — React Native Fundamentals](#module-1--react-native-fundamentals) | Hooks, Component Patterns, RN Specifics |
| 2 | [Module 2 — Expo Setup & Development Tooling](#module-2--expo-setup--development-tooling) | Expo vs CLI, Project Init, Dev Tools |
| 3 | [Module 3 — Styling in React Native](#module-3--styling-in-react-native) | StyleSheet, Flexbox, Responsive Design, Theming |
| 4 | [Module 4 — Navigation with Expo Router](#module-4--navigation-with-expo-router) | File-Based Routing, Navigation Patterns, Headers |
| 5 | [Module 5 — State Management](#module-5--state-management) | Local State, Zustand, Redux Toolkit, TanStack Query |
| 6 | [Module 6 — Networking & API Integration](#module-6--networking--api-integration) | HTTP Clients, Data Fetching, Error Handling, Validation |
| 7 | [Module 7 — Forms & Input Handling](#module-7--forms--input-handling) | React Hook Form, Zod, Keyboard, Input Types |
| 8 | [Module 8 — Native APIs & Device Features](#module-8--native-apis--device-features) | Camera, Gallery, File System, Geolocation, Biometrics |
| 9 | [Module 9 — Authentication & Security](#module-9--authentication--security) | Auth Flows, JWT, Firebase, OAuth, Security |
| 10 | [Module 10 — Storage, Offline & Caching](#module-10--storage-offline--caching) | Key-Value, SQLite, Offline-First, Sync, Cache |
| 11 | [Module 11 — Push Notifications](#module-11--push-notifications) | APNs, FCM, expo-notifications, Local Notifications |
| 12 | [Module 12 — Architecture & Best Practices](#module-12--architecture-code-organization--best-practices) | Folder Structure, Patterns, TypeScript, Code Quality |

---

## Module 1 — React Native Fundamentals

[module1.md](module1.md)

- **Hooks Essentials** — useState, useEffect, useRef, useMemo, useCallback, useContext, useReducer, Custom Hooks
- **Component Patterns** — Functional Components, Props, Children & Composition, Conditional Rendering, Lists & Keys, Lifting State, Controlled vs Uncontrolled
- **React Native Specifics** — Core Components, Differences from DOM, JSX Rules, Platform Module, Dimensions API, Platform-Specific File Extensions

---

## Module 2 — Expo Setup & Development Tooling

[module2.md](module2.md)

- **Expo vs Bare React Native CLI** — When to Choose Which, Expo SDK Versions, Expo Dev Client vs Expo Go
- **Project Initialization** — create-expo-app, TypeScript Setup, ESLint & Prettier, Folder Structure
- **Development Tools** — Metro Bundler, React Native DevTools, Flipper / React DevTools Profiler, Expo CLI Commands, iOS Simulator & Android Emulator Setup

---

## Module 3 — Styling in React Native

[module3.md](module3.md)

- **StyleSheet Fundamentals** — StyleSheet.create, Inline vs StyleSheet, Style Composition, camelCase Properties
- **Flexbox in React Native** — flexDirection, justifyContent, alignItems, alignSelf, flex / flexGrow / flexShrink / flexBasis, gap
- **Responsive Design** — Dimensions API, useWindowDimensions, PixelRatio, Percentage Sizing, Aspect Ratio, Tablets & Foldables, Orientation
- **Safe Areas & Notches** — SafeAreaView, react-native-safe-area-context, useSafeAreaInsets, StatusBar
- **Styling Libraries** — NativeWind v4 (Tailwind for RN), Unistyles 3.0, StyleSheet with Theme Context
- **Theming** — Light & Dark Mode, useColorScheme, Theme Provider Pattern, Dynamic Color Schemes
- **Platform-Specific Styling** — Platform.select, Shadow vs Elevation, Font Handling

---

## Module 4 — Navigation with Expo Router

[module4.md](module4.md)

- **Expo Router** — File-Based Routing, app Directory Structure, Layouts & Nested Routes, Dynamic Routes, Route Groups, Modal Routes
- **Navigation Patterns** — Passing Params, Nested Navigators, Auth Flow Navigation, Deep Linking, Universal Links, useNavigation, useRoute, useFocusEffect
- **Header & UI Customization** — Screen Options, Custom Headers, Tab Bar Customization, Back Button Handling, Hardware Back Button (Android)

---

## Module 5 — State Management

[module5.md](module5.md)

- **Local State** — useState Patterns, useReducer for Complex State, Context API with useContext
- **Global State — Zustand** — Store Creation, Selectors & Shallow Comparison, Slices Pattern, Persist Middleware, DevTools Middleware
- **Global State — Redux Toolkit** — Store Setup, createSlice, createAsyncThunk, RTK Query, Redux Persist, useSelector & useDispatch
- **Server State — TanStack Query** — QueryClient Setup, useQuery, useMutation, Query Keys & Caching, Invalidation, Optimistic Updates, Infinite Queries, Offline-First
- **State Architecture Decisions** — Server vs Client State, Lifting State, Avoiding Prop Drilling, Atomic State with Jotai

---

## Module 6 — Networking & API Integration

[module6.md](module6.md)

- **HTTP Clients** — fetch API, Axios Setup & Instance, Request/Response Interceptors, Base URL Configuration, Timeout Handling
- **Data Fetching Patterns** — TanStack Query Integration, Loading States, Error States, Empty States, Pagination, Infinite Scroll
- **Error Handling** — Try-Catch, HTTP Status Codes, Network Error Detection, Retry Logic, Error Boundaries, Global Error Handlers, User-Friendly Messages
- **Data Validation** — Zod Schemas for API Responses, Runtime Type Checking, TypeScript Types from Zod
- **Advanced Concepts** — Request Cancellation (AbortController), Debouncing & Throttling, Race Condition Handling, Token Refresh Flow, API Versioning

---

## Module 7 — Forms & Input Handling

[module7.md](module7.md)

- **Form Basics** — TextInput Component, Controlled Inputs, Keyboard Types, returnKeyType, autoCapitalize & autoCorrect, secureTextEntry
- **React Hook Form** — useForm, Controller, register & setValue, handleSubmit, formState, watch & useWatch, Reset & Default Values
- **Validation with Zod** — Schema Definition, zodResolver, Custom Validation, Async Validation, Cross-Field Validation
- **Form UX** — KeyboardAvoidingView, react-native-keyboard-controller, Focus Management, Error Display, Submit Button States, Form Persistence
- **Input Types** — Text, Picker/Dropdown, Date & Time Pickers, Checkbox & Switch, Radio Buttons, Multi-Select, Search Inputs, OTP Inputs

---

## Module 8 — Native APIs & Device Features

[module8.md](module8.md)

- **Permissions Handling** — expo-permissions, react-native-permissions, iOS Info.plist, Android Manifest, Runtime Requests, Permission Rationale UX, Blocked Permission Recovery
- **Camera** — expo-camera, Photo Capture, Video Recording, Camera Permissions, Front/Back Switching, Flash Control
- **Gallery & Image Picker** — expo-image-picker, Single & Multi-Select, Image Cropping, Video Selection, MediaLibrary API
- **File System** — expo-file-system, Reading & Writing Files, File URIs vs Paths, Caching Directories, Document Directories
- **File Upload** — Multipart Form Data, FormData Construction, Progress Tracking, Resumable Uploads, Background Uploads
- **Other Native APIs** — Geolocation, Contacts, Calendar, Clipboard, Haptics, Linking, Share API, Biometrics (Face ID / Touch ID), Network Info, Battery & Device Info

---

## Module 9 — Authentication & Security

[module9.md](module9.md)

- **Auth Flows** — Login/Signup UI, Forgot Password, OTP/Email Verification, Protected Routes, Auth State Persistence, Auto-Login, Logout
- **Token-Based Auth** — JWT Basics, Access & Refresh Tokens, Token Storage (SecureStore/Keychain), Token Refresh Interceptor, Session Expiry Handling
- **Firebase Authentication** — Firebase Setup, @react-native-firebase/auth, Email/Password, Google Sign-In, Apple Sign-In, Phone Auth, Anonymous Auth
- **OAuth / Social Login** — expo-auth-session, Google OAuth, Facebook Login, Apple Sign-In, Microsoft/Azure AD, PKCE Flow
- **Enterprise Auth** — Azure AD / Entra ID, SSO Integration, SAML Basics, MSAL Library
- **Security Best Practices** — Secure Token Storage, Certificate Pinning, Jailbreak/Root Detection, Biometric Gating, Session Timeout

---

## Module 10 — Storage, Offline & Caching

[module10.md](module10.md)

- **Key-Value Storage** — AsyncStorage Basics, react-native-mmkv (Recommended), expo-secure-store (Sensitive Data), Storage Encryption
- **Database Solutions** — SQLite (expo-sqlite), WatermelonDB, Realm / MongoDB Realm, op-sqlite (Performance)
- **Offline-First Patterns** — Cache-Then-Network, Optimistic UI Updates, Queue-Based Sync, Conflict Resolution, Network State Detection
- **Data Synchronization** — Last-Write-Wins, Timestamp-Based Sync, Delta Sync, Background Sync, TanStack Query Persistence
- **Cache Management** — Image Caching (expo-image), API Response Caching, Cache Invalidation, Storage Limits & Cleanup

---

## Module 11 — Push Notifications

[module11.md](module11.md)

- **Concepts** — Push vs Local Notifications, Notification Lifecycle, Foreground vs Background Handling, Notification Channels (Android), Notification Categories (iOS)
- **Setup — iOS (APNs)** — Apple Developer Certificates, Push Notification Capability, APNs Keys, Provisioning Profiles
- **Setup — Android (FCM)** — Firebase Project Configuration, google-services.json, FCM Server Key, Android 13+ POST_NOTIFICATIONS Permission
- **Implementation** — expo-notifications, @react-native-firebase/messaging, Device Token Registration, Token Refresh Handling, Sending to Backend
- **Notification Handling** — Foreground & Background Receiving, Tap-to-Open Deep Linking, Payload Parsing, Badge Count, Rich Notifications, Silent/Data-Only Notifications
- **Local Notifications** — Scheduling, Recurring Notifications, Cancelling Scheduled Notifications

---

## Module 12 — Architecture, Code Organization & Best Practices

[module12.md](module12.md)

- **Folder Structure** — Feature-Based, Layer-Based, src/ Organization, Barrel Exports, Path Aliases
- **Reusable Components** — Atomic Design, Component Library Structure, Prop API Design, Compound Components, Render Props, ForwardRef
- **Custom Hooks** — Hook Extraction Patterns, Naming Conventions, Returning Tuples vs Objects, Shared Business Logic Hooks
- **Clean Code Practices** — Separation of Concerns, Service Layer, Repository Pattern, Container vs Presentational, Single Responsibility, DRY vs Premature Abstraction
- **TypeScript Patterns** — Strict Mode, Type vs Interface, Generics in Components, Utility Types, Type Guards, Discriminated Unions, Avoiding any
- **Code Quality Tools** — ESLint, Prettier, Husky Pre-commit Hooks, lint-staged, Commit Conventions
