# Module 7 — Forms & Input Handling

> Forms in React Native require more manual wiring than on the web — no native `<form>` element, no built-in submit, no automatic keyboard management. This module covers everything from raw `TextInput` to full RHF + Zod form architecture.

---

## Table of Contents

1. [Form Basics](#1-form-basics)
   - 1.1 [TextInput Component](#11-textinput-component)
   - 1.2 [Controlled Inputs](#12-controlled-inputs)
   - 1.3 [Keyboard Types](#13-keyboard-types-keyboardtype-prop)
   - 1.4 [returnKeyType](#14-returnkeytype)
   - 1.5 [autoCapitalize and autoCorrect](#15-autocapitalize-and-autocorrect)
   - 1.6 [secureTextEntry](#16-securetextentry)

2. [React Hook Form](#2-react-hook-form)
   - 2.1 [useForm Hook](#21-useform-hook)
   - 2.2 [Controller Component](#22-controller-component)
   - 2.3 [register and setValue](#23-register-and-setvalue)
   - 2.4 [handleSubmit](#24-handlesubmit)
   - 2.5 [formState](#25-formstate-errors-isdirty-isvalid)
   - 2.6 [watch and useWatch](#26-watch-and-usewatch)
   - 2.7 [Reset and Default Values](#27-reset-and-default-values)

3. [Validation with Zod](#3-validation-with-zod)
   - 3.1 [Schema Definition](#31-schema-definition)
   - 3.2 [zodResolver](#32-zodresolver)
   - 3.3 [Custom Validation Rules](#33-custom-validation-rules)
   - 3.4 [Async Validation](#34-async-validation)
   - 3.5 [Cross-Field Validation](#35-cross-field-validation)

4. [Form UX](#4-form-ux)
   - 4.1 [KeyboardAvoidingView](#41-keyboardavoidingview)
   - 4.2 [react-native-keyboard-controller](#42-react-native-keyboard-controller)
   - 4.3 [Input Focus Management](#43-input-focus-management)
   - 4.4 [Error Message Display](#44-error-message-display)
   - 4.5 [Submit Button States](#45-submit-button-states)
   - 4.6 [Form Persistence Across Navigation](#46-form-persistence-across-navigation)

5. [Input Types](#5-input-types)
   - 5.1 [Text Inputs](#51-text-inputs)
   - 5.2 [Picker / Dropdown](#52-picker--dropdown)
   - 5.3 [Date and Time Pickers](#53-date-and-time-pickers)
   - 5.4 [Checkbox and Switch](#54-checkbox-and-switch)
   - 5.5 [Radio Buttons](#55-radio-buttons)
   - 5.6 [Multi-Select](#56-multi-select)
   - 5.7 [Search Inputs](#57-search-inputs)
   - 5.8 [OTP Inputs](#58-otp-inputs)

---

## 1. Form Basics

### 1.1 TextInput Component

`TextInput` is the only text input primitive in React Native. All other input types are built on top of it.

```tsx
import { TextInput, StyleSheet } from 'react-native';

<TextInput
  style={styles.input}
  value={value}
  onChangeText={setValue}
  placeholder="Enter your name"
  placeholderTextColor="#999"
/>

const styles = StyleSheet.create({
  input: {
    height: 48,
    borderWidth: 1,
    borderColor: '#ccc',
    borderRadius: 8,
    paddingHorizontal: 12,
    fontSize: 16,
    color: '#111',
    backgroundColor: '#fff',
  },
});
```

**Key props reference:**

| Prop | Type | Description |
|---|---|---|
| `value` | string | Controlled value |
| `onChangeText` | `(text: string) => void` | Called on every keystroke |
| `onSubmitEditing` | function | Called when return key is pressed |
| `onFocus` / `onBlur` | function | Focus/blur events |
| `placeholder` | string | Placeholder text |
| `placeholderTextColor` | string | Placeholder text color |
| `editable` | boolean | Enable/disable the input |
| `maxLength` | number | Character limit |
| `multiline` | boolean | Allow multiple lines |
| `numberOfLines` | number | Initial visible lines (multiline only) |
| `ref` | RefObject | Ref for programmatic focus |

---

### 1.2 Controlled Inputs

React Native `TextInput` works best as a **controlled component** — React owns the state.

```tsx
import { useState } from 'react';
import { View, TextInput, Text, StyleSheet } from 'react-native';

export default function EmailInput() {
  const [email, setEmail] = useState('');
  const [isFocused, setIsFocused] = useState(false);

  return (
    <View>
      <Text style={styles.label}>Email</Text>
      <TextInput
        style={[
          styles.input,
          isFocused && styles.inputFocused,
        ]}
        value={email}
        onChangeText={setEmail}
        onFocus={() => setIsFocused(true)}
        onBlur={() => setIsFocused(false)}
        placeholder="you@example.com"
        placeholderTextColor="#999"
        keyboardType="email-address"
        autoCapitalize="none"
        autoCorrect={false}
      />
      <Text style={styles.hint}>{email.length}/100</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  label: { fontSize: 14, fontWeight: '600', marginBottom: 6, color: '#333' },
  input: {
    height: 48,
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    paddingHorizontal: 12,
    fontSize: 16,
    backgroundColor: '#fff',
  },
  inputFocused: { borderColor: '#007AFF', borderWidth: 2 },
  hint: { fontSize: 12, color: '#999', marginTop: 4, textAlign: 'right' },
});
```

---

### 1.3 Keyboard Types (keyboardType prop)

Controls which keyboard the OS shows for the input.

```tsx
// Numeric keypad (PIN, quantity)
<TextInput keyboardType="numeric" />

// Phone number pad
<TextInput keyboardType="phone-pad" />

// Email keyboard (@ and . on main screen)
<TextInput keyboardType="email-address" />

// URL keyboard (/ and . on main screen)
<TextInput keyboardType="url" />

// Decimal number (with decimal point)
<TextInput keyboardType="decimal-pad" />

// Default (full keyboard)
<TextInput keyboardType="default" />
```

**All values and platform support:**

| `keyboardType` | iOS | Android | Use for |
|---|---|---|---|
| `default` | ✅ | ✅ | General text |
| `email-address` | ✅ | ✅ | Email fields |
| `numeric` | ✅ | ✅ | Numbers only |
| `phone-pad` | ✅ | ✅ | Phone numbers |
| `decimal-pad` | ✅ | ✅ | Prices, measurements |
| `url` | ✅ | ✅ | URL fields |
| `number-pad` | ✅ | ✅ | PIN, quantity |
| `ascii-capable` | ✅ | ❌ | Latin characters only |
| `visible-password` | ❌ | ✅ | Password with visibility toggle |

---

### 1.4 returnKeyType

Controls the label on the return/done key of the keyboard — signals to the user what happens when they press it.

```tsx
// "Next" — move to next field
<TextInput returnKeyType="next" onSubmitEditing={() => emailRef.current?.focus()} />

// "Done" — close keyboard
<TextInput returnKeyType="done" onSubmitEditing={() => Keyboard.dismiss()} />

// "Search" — trigger search
<TextInput returnKeyType="search" onSubmitEditing={handleSearch} />

// "Send" — send message
<TextInput returnKeyType="send" onSubmitEditing={handleSend} />

// "Go" — submit form
<TextInput returnKeyType="go" onSubmitEditing={handleSubmit} />
```

| Value | Label shown |
|---|---|
| `done` | Done |
| `go` | Go |
| `next` | Next |
| `search` | Search |
| `send` | Send |
| `default` | Return |

---

### 1.5 autoCapitalize and autoCorrect

```tsx
// autoCapitalize options
<TextInput autoCapitalize="none" />        // no auto-cap (email, password, username)
<TextInput autoCapitalize="sentences" />   // default — capitalize first word of sentences
<TextInput autoCapitalize="words" />       // capitalize each word (name fields)
<TextInput autoCapitalize="characters" />  // ALL CAPS

// autoCorrect — disable for fields where autocorrect hurts more than helps
<TextInput autoCorrect={false} />  // username, email, code fields
<TextInput autoCorrect={true} />   // default — message/text fields

// spellCheck (iOS only) — underline misspelled words
<TextInput spellCheck={false} />

// autoComplete — hints to the OS for autofill
<TextInput autoComplete="email" />
<TextInput autoComplete="password" />
<TextInput autoComplete="username" />
<TextInput autoComplete="name" />
<TextInput autoComplete="tel" />
<TextInput autoComplete="postal-code" />
<TextInput autoComplete="street-address" />
```

---

### 1.6 secureTextEntry

Masks input for passwords. Always combine with `autoCapitalize="none"` and `autoCorrect={false}`.

```tsx
import { useState } from 'react';
import { View, TextInput, Pressable, Text, StyleSheet } from 'react-native';
import { Ionicons } from '@expo/vector-icons';

export default function PasswordInput({ value, onChange }: { value: string; onChange: (v: string) => void }) {
  const [isVisible, setIsVisible] = useState(false);

  return (
    <View style={styles.container}>
      <TextInput
        style={styles.input}
        value={value}
        onChangeText={onChange}
        secureTextEntry={!isVisible}   // toggle masking
        autoCapitalize="none"
        autoCorrect={false}
        textContentType="password"     // iOS Keychain autofill
        autoComplete="current-password" // Android autofill
        placeholder="Enter password"
        placeholderTextColor="#999"
      />
      <Pressable
        onPress={() => setIsVisible(v => !v)}
        style={styles.toggle}
        hitSlop={8}
      >
        <Ionicons
          name={isVisible ? 'eye-off-outline' : 'eye-outline'}
          size={20}
          color="#666"
        />
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    alignItems: 'center',
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    backgroundColor: '#fff',
  },
  input: { flex: 1, height: 48, paddingHorizontal: 12, fontSize: 16 },
  toggle: { paddingHorizontal: 12 },
});
```

> `textContentType` (iOS) and `autoComplete` (Android) enable OS-level autofill from Keychain/Google Password Manager.

---

## 2. React Hook Form

React Hook Form (RHF) minimizes re-renders by using uncontrolled inputs under the hood, while still giving you full validation control.

```bash
npx expo install react-hook-form
```

### 2.1 useForm Hook

```tsx
import { useForm } from 'react-hook-form';

type LoginForm = {
  email: string;
  password: string;
  rememberMe: boolean;
};

const {
  control,          // pass to Controller components
  handleSubmit,     // wrap your submit function
  formState,        // { errors, isDirty, isValid, isSubmitting }
  watch,            // observe field values
  setValue,         // programmatically set a field value
  getValues,        // read current values without subscribing
  reset,            // reset form to default values
  setError,         // manually set an error on a field
  clearErrors,      // clear errors
  trigger,          // manually trigger validation
} = useForm<LoginForm>({
  defaultValues: {
    email: '',
    password: '',
    rememberMe: false,
  },
  mode: 'onBlur',   // when to validate: 'onChange' | 'onBlur' | 'onSubmit' | 'all'
});
```

**`mode` options:**

| Mode | Validates on |
|---|---|
| `onSubmit` (default) | Submit only |
| `onBlur` | When user leaves the field |
| `onChange` | Every keystroke (expensive) |
| `onTouched` | First blur, then every change |
| `all` | Both onChange and onBlur |

---

### 2.2 Controller Component

`Controller` bridges RHF with React Native's controlled inputs (since RN inputs use `onChangeText`, not `onChange`).

```tsx
import { Controller, useForm } from 'react-hook-form';
import { TextInput, View, Text, StyleSheet } from 'react-native';

type LoginForm = { email: string; password: string };

export default function LoginScreen() {
  const { control, handleSubmit, formState: { errors } } = useForm<LoginForm>({
    defaultValues: { email: '', password: '' },
  });

  return (
    <View style={styles.form}>
      {/* Email field */}
      <Controller
        control={control}
        name="email"
        rules={{
          required: 'Email is required',
          pattern: { value: /\S+@\S+\.\S+/, message: 'Invalid email address' },
        }}
        render={({ field: { value, onChange, onBlur }, fieldState: { error } }) => (
          <View>
            <Text style={styles.label}>Email</Text>
            <TextInput
              style={[styles.input, error && styles.inputError]}
              value={value}
              onChangeText={onChange}
              onBlur={onBlur}
              keyboardType="email-address"
              autoCapitalize="none"
              autoCorrect={false}
              placeholder="you@example.com"
            />
            {error && <Text style={styles.errorText}>{error.message}</Text>}
          </View>
        )}
      />

      {/* Password field */}
      <Controller
        control={control}
        name="password"
        rules={{ required: 'Password is required', minLength: { value: 8, message: 'Min 8 characters' } }}
        render={({ field: { value, onChange, onBlur }, fieldState: { error } }) => (
          <View>
            <Text style={styles.label}>Password</Text>
            <TextInput
              style={[styles.input, error && styles.inputError]}
              value={value}
              onChangeText={onChange}
              onBlur={onBlur}
              secureTextEntry
              autoCapitalize="none"
              placeholder="Password"
            />
            {error && <Text style={styles.errorText}>{error.message}</Text>}
          </View>
        )}
      />
    </View>
  );
}
```

**Reusable controlled input component:**
```tsx
// src/components/FormInput.tsx
import { Controller, Control, FieldValues, Path, RegisterOptions } from 'react-hook-form';
import { View, Text, TextInput, TextInputProps, StyleSheet } from 'react-native';

type FormInputProps<T extends FieldValues> = TextInputProps & {
  control: Control<T>;
  name: Path<T>;
  label: string;
  rules?: RegisterOptions;
};

export function FormInput<T extends FieldValues>({
  control,
  name,
  label,
  rules,
  ...inputProps
}: FormInputProps<T>) {
  return (
    <Controller
      control={control}
      name={name}
      rules={rules}
      render={({ field: { value, onChange, onBlur }, fieldState: { error } }) => (
        <View style={styles.container}>
          <Text style={styles.label}>{label}</Text>
          <TextInput
            style={[styles.input, error && styles.inputError]}
            value={value as string}
            onChangeText={onChange}
            onBlur={onBlur}
            placeholderTextColor="#999"
            {...inputProps}
          />
          {error && <Text style={styles.error}>{error.message}</Text>}
        </View>
      )}
    />
  );
}

const styles = StyleSheet.create({
  container: { marginBottom: 16 },
  label: { fontSize: 14, fontWeight: '600', marginBottom: 6, color: '#333' },
  input: {
    height: 48,
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    paddingHorizontal: 12,
    fontSize: 16,
    backgroundColor: '#fff',
  },
  inputError: { borderColor: '#ef4444' },
  error: { fontSize: 12, color: '#ef4444', marginTop: 4 },
});

// Usage — clean, no repeated Controller boilerplate
<FormInput control={control} name="email" label="Email" keyboardType="email-address" />
<FormInput control={control} name="password" label="Password" secureTextEntry />
```

---

### 2.3 register and setValue

`register` is the web version of `Controller` — not typically used in React Native since inputs need `onChangeText`. Use `Controller` instead.

`setValue` is useful for setting values programmatically (e.g., after selecting from a picker, or pre-filling from an API).

```tsx
const { setValue, getValues, control } = useForm<ProfileForm>();

// Set a single field
setValue('country', 'US');

// Set with validation trigger
setValue('country', 'US', { shouldValidate: true, shouldDirty: true });

// Set multiple fields at once
const profileData = await api.getProfile();
Object.entries(profileData).forEach(([key, value]) => {
  setValue(key as keyof ProfileForm, value);
});

// Read values without subscribing to re-renders
const currentEmail = getValues('email');
const allValues = getValues(); // all fields
```

---

### 2.4 handleSubmit

Wraps your submit handler — runs validation first, calls your function only if valid.

```tsx
import { useMutation } from '@tanstack/react-query';
import { router } from 'expo-router';

export default function LoginScreen() {
  const { control, handleSubmit, setError, formState: { isSubmitting } } = useForm<LoginForm>();

  const loginMutation = useMutation({ mutationFn: authApi.login });

  const onSubmit = async (data: LoginForm) => {
    try {
      await loginMutation.mutateAsync(data);
      router.replace('/(app)/(tabs)/');
    } catch (error) {
      if (isAxiosError(error) && error.response?.status === 401) {
        // Set a server-side error on a specific field
        setError('password', {
          type: 'server',
          message: 'Incorrect email or password',
        });
      } else {
        // Set a root-level error for non-field errors
        setError('root', { message: getErrorMessage(error) });
      }
    }
  };

  return (
    <View>
      {/* form fields */}
      <Pressable
        onPress={handleSubmit(onSubmit)}  // validates then calls onSubmit
        disabled={isSubmitting}
      >
        <Text>{isSubmitting ? 'Signing in...' : 'Sign In'}</Text>
      </Pressable>
    </View>
  );
}
```

> `handleSubmit` also accepts an `onInvalid` callback as the second argument — fires when validation fails, useful for analytics or scrolling to the first error.

```tsx
handleSubmit(onSubmit, (errors) => {
  console.log('Validation failed:', errors);
  // Scroll to first error, show snackbar, etc.
})
```

---

### 2.5 formState (errors, isDirty, isValid)

```tsx
const {
  formState: {
    errors,         // { email: { message: '...' }, password: { message: '...' } }
    isDirty,        // true if any field differs from defaultValues
    isValid,        // true if all fields pass validation
    isSubmitting,   // true while handleSubmit is executing
    isSubmitted,    // true after first submit attempt
    isSubmitSuccessful, // true if submit completed without errors
    touchedFields,  // { email: true } — fields the user has interacted with
    dirtyFields,    // { email: true } — fields that differ from default
    submitCount,    // number of times submit was attempted
  }
} = useForm<LoginForm>({ mode: 'onBlur' });

// Practical usage
<Pressable
  onPress={handleSubmit(onSubmit)}
  disabled={!isValid || isSubmitting}   // disable if invalid or mid-submit
  style={{ opacity: !isValid ? 0.5 : 1 }}
>
  {isSubmitting ? <ActivityIndicator /> : <Text>Submit</Text>}
</Pressable>

// Show root-level error (non-field error)
{errors.root && <Text style={{ color: 'red' }}>{errors.root.message}</Text>}

// Show unsaved changes warning
{isDirty && <Text>You have unsaved changes</Text>}
```

---

### 2.6 watch and useWatch

Subscribe to field value changes — useful for conditional rendering, dependent fields, and live previews.

```tsx
import { useForm, useWatch } from 'react-hook-form';

type SignupForm = {
  password: string;
  confirmPassword: string;
  role: 'buyer' | 'seller';
  businessName: string;
};

export default function SignupScreen() {
  const { control, watch } = useForm<SignupForm>();

  // watch() — subscribes the whole component (re-renders on any change)
  const role = watch('role');

  // useWatch — more granular subscription (only re-renders on this field)
  const password = useWatch({ control, name: 'password' });
  const passwordStrength = calculateStrength(password);

  return (
    <View>
      {/* Role selector */}
      <Controller control={control} name="role" render={/* ... */} />

      {/* Conditionally show business name for sellers only */}
      {role === 'seller' && (
        <Controller control={control} name="businessName" render={/* ... */} />
      )}

      {/* Live password strength meter */}
      <PasswordStrengthBar strength={passwordStrength} />
    </View>
  );
}
```

**`watch` vs `useWatch`:**

| | `watch('field')` | `useWatch({ control, name })` |
|---|---|---|
| Returns | Current value | Current value |
| Re-renders | Whole component on any field change | Only when this field changes |
| Best for | Conditional form logic | Performance-sensitive subscriptions |

---

### 2.7 Reset and Default Values

```tsx
const { reset, formState: { isDirty } } = useForm<ProfileForm>({
  defaultValues: { name: '', email: '', bio: '' },
});

// Reset to original defaultValues
reset();

// Reset to new values (e.g., after loading data from API)
const profile = await api.getProfile();
reset({
  name: profile.name,
  email: profile.email,
  bio: profile.bio ?? '',
});

// Reset with options
reset(newValues, {
  keepErrors: false,    // clear errors on reset
  keepDirty: false,     // reset isDirty to false
  keepIsSubmitted: false,
  keepTouched: false,
  keepDefaultValues: false,
});

// Pre-fill form from API — common pattern
useEffect(() => {
  if (profileData) {
    reset(profileData);
  }
}, [profileData, reset]);
```

---

## 3. Validation with Zod

```bash
npx expo install zod @hookform/resolvers
```

### 3.1 Schema Definition

```tsx
// src/schemas/authSchema.ts
import { z } from 'zod';

export const loginSchema = z.object({
  email: z
    .string()
    .min(1, 'Email is required')
    .email('Enter a valid email address'),
  password: z
    .string()
    .min(1, 'Password is required')
    .min(8, 'Password must be at least 8 characters'),
});

export const signupSchema = z
  .object({
    name: z
      .string()
      .min(1, 'Name is required')
      .min(2, 'Name must be at least 2 characters')
      .max(50, 'Name must be under 50 characters'),
    email: z.string().min(1, 'Email is required').email('Invalid email'),
    password: z
      .string()
      .min(8, 'At least 8 characters')
      .regex(/[A-Z]/, 'Must contain an uppercase letter')
      .regex(/[0-9]/, 'Must contain a number')
      .regex(/[^A-Za-z0-9]/, 'Must contain a special character'),
    confirmPassword: z.string().min(1, 'Please confirm your password'),
  })
  .refine(data => data.password === data.confirmPassword, {
    message: 'Passwords do not match',
    path: ['confirmPassword'],  // attach error to confirmPassword field
  });

// Infer types from schemas
export type LoginForm = z.infer<typeof loginSchema>;
export type SignupForm = z.infer<typeof signupSchema>;
```

---

### 3.2 zodResolver

Connect your Zod schema to React Hook Form as the validator.

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { loginSchema, type LoginForm } from '@/schemas/authSchema';

export default function LoginScreen() {
  const {
    control,
    handleSubmit,
    formState: { errors, isValid, isSubmitting },
  } = useForm<LoginForm>({
    resolver: zodResolver(loginSchema), // ← Zod handles all validation
    defaultValues: { email: '', password: '' },
    mode: 'onBlur',
  });

  // No more inline `rules` props on Controller — Zod handles everything
  return (
    <View>
      <Controller
        control={control}
        name="email"
        render={({ field: { value, onChange, onBlur }, fieldState: { error } }) => (
          <TextInput
            value={value}
            onChangeText={onChange}
            onBlur={onBlur}
            keyboardType="email-address"
            autoCapitalize="none"
            // error comes from Zod schema, not inline rules
          />
          // error.message is exactly what you wrote in the Zod schema
        )}
      />
    </View>
  );
}
```

---

### 3.3 Custom Validation Rules

Zod's `.refine()` lets you add custom logic that's not expressible with built-in validators.

```tsx
import { z } from 'zod';

const profileSchema = z.object({
  username: z
    .string()
    .min(3)
    .max(20)
    .regex(/^[a-z0-9_]+$/, 'Only lowercase letters, numbers, and underscores')
    .refine(
      val => !['admin', 'root', 'system'].includes(val),
      'This username is reserved'
    ),

  age: z
    .number()
    .int('Age must be a whole number')
    .refine(val => val >= 18, 'You must be at least 18 years old')
    .refine(val => val <= 120, 'Please enter a valid age'),

  website: z
    .string()
    .optional()
    .refine(
      val => !val || val.startsWith('https://'),
      'Website must use HTTPS'
    ),

  phoneNumber: z
    .string()
    .refine(val => {
      const digits = val.replace(/\D/g, '');
      return digits.length === 10 || digits.length === 11;
    }, 'Enter a valid 10 or 11 digit phone number'),
});
```

**`.superRefine()` — multiple errors from one field:**
```tsx
const passwordSchema = z.object({
  password: z.string().superRefine((val, ctx) => {
    if (val.length < 8) {
      ctx.addIssue({ code: z.ZodIssueCode.too_small, minimum: 8, type: 'string', inclusive: true, message: 'At least 8 characters' });
    }
    if (!/[A-Z]/.test(val)) {
      ctx.addIssue({ code: z.ZodIssueCode.custom, message: 'Must contain an uppercase letter' });
    }
    if (!/[0-9]/.test(val)) {
      ctx.addIssue({ code: z.ZodIssueCode.custom, message: 'Must contain a number' });
    }
  }),
});
```

---

### 3.4 Async Validation

Validate against a server (e.g., check if username is taken) without leaving the form.

```tsx
import { z } from 'zod';

const signupSchema = z.object({
  username: z
    .string()
    .min(3)
    .refine(
      async (username) => {
        const { available } = await api.checkUsername(username);
        return available;
      },
      'This username is already taken'
    ),
  email: z.string().email(),
});

// Use zodResolver — it handles async refinements automatically
const { control } = useForm({
  resolver: zodResolver(signupSchema),
  mode: 'onBlur',  // trigger async validation on blur, not every keystroke
});
```

> Set `mode: 'onBlur'` with async validation — triggering a server check on every keystroke is expensive and causes poor UX.

---

### 3.5 Cross-Field Validation

Validate one field relative to another using `.refine()` or `.superRefine()` on the object level.

```tsx
const checkoutSchema = z
  .object({
    startDate: z.string().min(1, 'Start date is required'),
    endDate: z.string().min(1, 'End date is required'),
    promoCode: z.string().optional(),
    totalAmount: z.number().positive(),
    discountAmount: z.number().min(0),
  })
  .refine(
    data => new Date(data.endDate) > new Date(data.startDate),
    { message: 'End date must be after start date', path: ['endDate'] }
  )
  .refine(
    data => data.discountAmount <= data.totalAmount,
    { message: 'Discount cannot exceed total amount', path: ['discountAmount'] }
  );

// Multiple cross-field refinements — superRefine on the object
const transferSchema = z
  .object({
    fromAccount: z.string(),
    toAccount: z.string(),
    amount: z.number().positive(),
  })
  .superRefine((data, ctx) => {
    if (data.fromAccount === data.toAccount) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: 'Cannot transfer to the same account',
        path: ['toAccount'],
      });
    }
  });
```

---

## 4. Form UX

### 4.1 KeyboardAvoidingView

Pushes content up when the keyboard appears so inputs are not covered.

```tsx
import { KeyboardAvoidingView, Platform, ScrollView, StyleSheet } from 'react-native';

export default function LoginScreen() {
  return (
    <KeyboardAvoidingView
      style={{ flex: 1 }}
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
      // iOS: adds padding-bottom equal to keyboard height
      // Android: shrinks the view height
    >
      <ScrollView
        contentContainerStyle={styles.content}
        keyboardShouldPersistTaps="handled" // tapping non-input doesn't dismiss keyboard
      >
        {/* form fields */}
        <FormInput control={control} name="email" label="Email" />
        <FormInput control={control} name="password" label="Password" />
        <SubmitButton />
      </ScrollView>
    </KeyboardAvoidingView>
  );
}

const styles = StyleSheet.create({
  content: {
    flexGrow: 1,
    padding: 24,
    justifyContent: 'center',
  },
});
```

**`keyboardShouldPersistTaps` values:**
| Value | Behavior |
|---|---|
| `'never'` (default) | Tapping outside dismisses keyboard, tap not received by element |
| `'always'` | Keyboard stays open, taps always received |
| `'handled'` | Taps handled by children keep keyboard open; others dismiss it |

> Use `'handled'` in almost all cases — it lets you tap buttons without first dismissing the keyboard.

---

### 4.2 react-native-keyboard-controller

A better alternative to `KeyboardAvoidingView` with smoother animations and more predictable behavior across devices.

```bash
npx expo install react-native-keyboard-controller
```

```tsx
// app/_layout.tsx — add the provider once
import { KeyboardProvider } from 'react-native-keyboard-controller';

export default function RootLayout() {
  return (
    <KeyboardProvider>
      <Stack />
    </KeyboardProvider>
  );
}
```

```tsx
// In any screen with a form
import { KeyboardAwareScrollView } from 'react-native-keyboard-controller';

export default function SignupScreen() {
  return (
    <KeyboardAwareScrollView
      bottomOffset={16}      // gap between keyboard top and focused input
      keyboardShouldPersistTaps="handled"
    >
      <FormInput control={control} name="name" label="Name" />
      <FormInput control={control} name="email" label="Email" />
      <FormInput control={control} name="password" label="Password" />
    </KeyboardAwareScrollView>
  );
}
```

**`useKeyboardHandler` for custom keyboard animations:**
```tsx
import { useKeyboardHandler } from 'react-native-keyboard-controller';
import Animated, { useSharedValue, useAnimatedStyle } from 'react-native-reanimated';

function FloatingSubmitButton() {
  const translateY = useSharedValue(0);

  useKeyboardHandler({
    onMove: (e) => {
      'worklet';
      translateY.value = -e.height; // button floats above keyboard
    },
  });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateY: translateY.value }],
  }));

  return (
    <Animated.View style={[styles.button, animatedStyle]}>
      <Text>Submit</Text>
    </Animated.View>
  );
}
```

---

### 4.3 Input Focus Management

Chain focus through form fields with `returnKeyType="next"` for a smooth tabbing experience.

```tsx
import { useRef } from 'react';
import { TextInput, View } from 'react-native';

export default function SignupForm() {
  const nameRef = useRef<TextInput>(null);
  const emailRef = useRef<TextInput>(null);
  const passwordRef = useRef<TextInput>(null);
  const confirmRef = useRef<TextInput>(null);

  return (
    <View>
      <TextInput
        ref={nameRef}
        placeholder="Full Name"
        returnKeyType="next"
        onSubmitEditing={() => emailRef.current?.focus()}
        blurOnSubmit={false}  // prevent keyboard from closing between fields
      />
      <TextInput
        ref={emailRef}
        placeholder="Email"
        keyboardType="email-address"
        autoCapitalize="none"
        returnKeyType="next"
        onSubmitEditing={() => passwordRef.current?.focus()}
        blurOnSubmit={false}
      />
      <TextInput
        ref={passwordRef}
        placeholder="Password"
        secureTextEntry
        returnKeyType="next"
        onSubmitEditing={() => confirmRef.current?.focus()}
        blurOnSubmit={false}
      />
      <TextInput
        ref={confirmRef}
        placeholder="Confirm Password"
        secureTextEntry
        returnKeyType="done"
        onSubmitEditing={handleSubmit(onSubmit)} // last field submits
      />
    </View>
  );
}
```

**Dismiss keyboard programmatically:**
```tsx
import { Keyboard } from 'react-native';

// Dismiss on tapping outside inputs
<Pressable onPress={Keyboard.dismiss} style={{ flex: 1 }}>
  {/* form content */}
</Pressable>

// Or wrap the whole screen
import { TouchableWithoutFeedback } from 'react-native';
<TouchableWithoutFeedback onPress={Keyboard.dismiss}>
  <View style={{ flex: 1 }}>{/* form */}</View>
</TouchableWithoutFeedback>
```

---

### 4.4 Error Message Display

Consistent, accessible error display below each field.

```tsx
// src/components/FieldError.tsx
import { Text, StyleSheet } from 'react-native';
import { FieldError } from 'react-hook-form';

export function FieldError({ error }: { error?: FieldError }) {
  if (!error?.message) return null;
  return <Text style={styles.error}>{error.message}</Text>;
}

const styles = StyleSheet.create({
  error: {
    fontSize: 12,
    color: '#ef4444',
    marginTop: 4,
    marginLeft: 2,
  },
});

// Inline in a field
<Controller
  control={control}
  name="email"
  render={({ field, fieldState: { error } }) => (
    <View>
      <TextInput style={[styles.input, error && styles.inputError]} {...field} />
      <FieldError error={error} />
    </View>
  )}
/>
```

**Scroll to first error on submit:**
```tsx
import { ScrollView } from 'react-native';
import { useRef } from 'react';

export default function LongForm() {
  const scrollRef = useRef<ScrollView>(null);
  const fieldRefs = useRef<Record<string, { y: number }>>({});

  const scrollToFirstError = (errors: object) => {
    const firstErrorKey = Object.keys(errors)[0];
    const yPosition = fieldRefs.current[firstErrorKey]?.y ?? 0;
    scrollRef.current?.scrollTo({ y: yPosition - 16, animated: true });
  };

  return (
    <ScrollView ref={scrollRef}>
      <View onLayout={e => { fieldRefs.current.email = { y: e.nativeEvent.layout.y }; }}>
        <FormInput control={control} name="email" label="Email" />
      </View>
      {/* more fields */}
      <Pressable onPress={handleSubmit(onSubmit, scrollToFirstError)}>
        <Text>Submit</Text>
      </Pressable>
    </ScrollView>
  );
}
```

---

### 4.5 Submit Button States

A submit button should communicate all four states: idle, loading, success, and error.

```tsx
// src/components/SubmitButton.tsx
import { Pressable, Text, ActivityIndicator, StyleSheet, View } from 'react-native';
import { Ionicons } from '@expo/vector-icons';

type Status = 'idle' | 'loading' | 'success' | 'error';

type Props = {
  onPress: () => void;
  status: Status;
  label: string;
  disabled?: boolean;
};

export function SubmitButton({ onPress, status, label, disabled }: Props) {
  const isDisabled = disabled || status === 'loading';

  return (
    <Pressable
      onPress={onPress}
      disabled={isDisabled}
      style={[
        styles.button,
        status === 'success' && styles.buttonSuccess,
        status === 'error' && styles.buttonError,
        isDisabled && styles.buttonDisabled,
      ]}
    >
      {status === 'loading' && <ActivityIndicator color="#fff" size="small" style={{ marginRight: 8 }} />}
      {status === 'success' && <Ionicons name="checkmark" size={18} color="#fff" style={{ marginRight: 6 }} />}
      {status === 'error' && <Ionicons name="alert-circle" size={18} color="#fff" style={{ marginRight: 6 }} />}

      <Text style={styles.label}>
        {status === 'loading' ? 'Saving...' :
         status === 'success' ? 'Saved!' :
         status === 'error' ? 'Try Again' :
         label}
      </Text>
    </Pressable>
  );
}

const styles = StyleSheet.create({
  button: {
    flexDirection: 'row',
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#007AFF',
    height: 52,
    borderRadius: 12,
  },
  buttonSuccess: { backgroundColor: '#22c55e' },
  buttonError: { backgroundColor: '#ef4444' },
  buttonDisabled: { opacity: 0.5 },
  label: { color: '#fff', fontSize: 16, fontWeight: '600' },
});

// Usage
const [status, setStatus] = useState<Status>('idle');

const onSubmit = async (data: LoginForm) => {
  setStatus('loading');
  try {
    await loginMutation.mutateAsync(data);
    setStatus('success');
  } catch {
    setStatus('error');
    setTimeout(() => setStatus('idle'), 2000); // reset after 2s
  }
};

<SubmitButton
  onPress={handleSubmit(onSubmit)}
  status={status}
  label="Sign In"
  disabled={!isValid}
/>
```

---

### 4.6 Form Persistence Across Navigation

Preserve form state when the user navigates away and returns — common in multi-step flows.

**Option 1 — Zustand for multi-step forms:**
```tsx
// src/store/useCheckoutStore.ts
import { create } from 'zustand';

type CheckoutData = {
  shippingAddress: Partial<AddressForm>;
  paymentMethod: Partial<PaymentForm>;
  setShipping: (data: Partial<AddressForm>) => void;
  setPayment: (data: Partial<PaymentForm>) => void;
  reset: () => void;
};

export const useCheckoutStore = create<CheckoutData>((set) => ({
  shippingAddress: {},
  paymentMethod: {},
  setShipping: (data) => set({ shippingAddress: data }),
  setPayment: (data) => set({ paymentMethod: data }),
  reset: () => set({ shippingAddress: {}, paymentMethod: {} }),
}));
```

```tsx
// Step 1 — Shipping screen
export default function ShippingScreen() {
  const { shippingAddress, setShipping } = useCheckoutStore();
  const { control, handleSubmit } = useForm<AddressForm>({
    defaultValues: shippingAddress, // pre-fill from store
  });

  const onNext = (data: AddressForm) => {
    setShipping(data);         // persist to store
    router.push('/checkout/payment');
  };

  return (/* form */);
}
```

**Option 2 — Save to store on `useFocusEffect` blur:**
```tsx
useFocusEffect(
  useCallback(() => {
    return () => {
      // Runs when screen loses focus — save form draft
      const values = getValues();
      useCheckoutStore.getState().setShipping(values);
    };
  }, [getValues])
);
```

---

## 5. Input Types

### 5.1 Text Inputs

Reusable text input with label, error, and left/right icon slots:

```tsx
// src/components/Input.tsx
import { View, Text, TextInput, TextInputProps, StyleSheet } from 'react-native';

type InputProps = TextInputProps & {
  label?: string;
  error?: string;
  hint?: string;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
};

export function Input({ label, error, hint, leftIcon, rightIcon, style, ...props }: InputProps) {
  return (
    <View style={styles.wrapper}>
      {label && <Text style={styles.label}>{label}</Text>}
      <View style={[styles.inputRow, error && styles.errorBorder]}>
        {leftIcon && <View style={styles.icon}>{leftIcon}</View>}
        <TextInput
          style={[styles.input, style]}
          placeholderTextColor="#999"
          {...props}
        />
        {rightIcon && <View style={styles.icon}>{rightIcon}</View>}
      </View>
      {error && <Text style={styles.error}>{error}</Text>}
      {hint && !error && <Text style={styles.hint}>{hint}</Text>}
    </View>
  );
}

// Multiline textarea variant
export function TextArea(props: InputProps) {
  return (
    <Input
      {...props}
      multiline
      numberOfLines={4}
      style={{ height: 100, textAlignVertical: 'top' }}
    />
  );
}
```

---

### 5.2 Picker / Dropdown

```bash
npx expo install @react-native-picker/picker
```

```tsx
import { Picker } from '@react-native-picker/picker';
import { Controller, Control } from 'react-hook-form';
import { View, Text, StyleSheet } from 'react-native';

type FormPickerProps<T> = {
  control: Control<any>;
  name: string;
  label: string;
  items: { label: string; value: T }[];
};

export function FormPicker<T>({ control, name, label, items }: FormPickerProps<T>) {
  return (
    <Controller
      control={control}
      name={name}
      render={({ field: { value, onChange }, fieldState: { error } }) => (
        <View style={styles.container}>
          <Text style={styles.label}>{label}</Text>
          <View style={[styles.pickerWrapper, error && styles.errorBorder]}>
            <Picker
              selectedValue={value}
              onValueChange={onChange}
              style={styles.picker}
            >
              <Picker.Item label="Select..." value="" />
              {items.map(item => (
                <Picker.Item key={String(item.value)} label={item.label} value={item.value} />
              ))}
            </Picker>
          </View>
          {error && <Text style={styles.error}>{error.message}</Text>}
        </View>
      )}
    />
  );
}

// Usage
<FormPicker
  control={control}
  name="country"
  label="Country"
  items={[
    { label: 'United States', value: 'US' },
    { label: 'India', value: 'IN' },
    { label: 'United Kingdom', value: 'GB' },
  ]}
/>
```

---

### 5.3 Date and Time Pickers

```bash
npx expo install @react-native-community/datetimepicker
```

```tsx
import DateTimePicker, { DateTimePickerEvent } from '@react-native-community/datetimepicker';
import { useState } from 'react';
import { Platform, Pressable, Text, View } from 'react-native';

type DatePickerProps = {
  value: Date;
  onChange: (date: Date) => void;
  label: string;
  mode?: 'date' | 'time' | 'datetime';
};

export function DatePicker({ value, onChange, label, mode = 'date' }: DatePickerProps) {
  const [show, setShow] = useState(false);

  const handleChange = (_: DateTimePickerEvent, selected?: Date) => {
    if (Platform.OS === 'android') setShow(false); // Android auto-closes
    if (selected) onChange(selected);
  };

  return (
    <View>
      <Text style={styles.label}>{label}</Text>
      <Pressable onPress={() => setShow(true)} style={styles.dateButton}>
        <Text style={styles.dateText}>
          {mode === 'time'
            ? value.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
            : value.toLocaleDateString()}
        </Text>
      </Pressable>

      {show && (
        <DateTimePicker
          value={value}
          mode={mode}
          display={Platform.OS === 'ios' ? 'spinner' : 'default'}
          onChange={handleChange}
          minimumDate={new Date()}
        />
      )}
    </View>
  );
}

// With RHF Controller
<Controller
  control={control}
  name="dueDate"
  render={({ field: { value, onChange } }) => (
    <DatePicker
      label="Due Date"
      value={value ?? new Date()}
      onChange={onChange}
    />
  )}
/>
```

---

### 5.4 Checkbox and Switch

```tsx
import { Switch, Pressable, View, Text, StyleSheet } from 'react-native';
import { Controller, Control } from 'react-hook-form';
import { Ionicons } from '@expo/vector-icons';

// Switch (toggle) — great for on/off settings
export function FormSwitch({ control, name, label }: { control: Control<any>; name: string; label: string }) {
  return (
    <Controller
      control={control}
      name={name}
      render={({ field: { value, onChange } }) => (
        <View style={styles.switchRow}>
          <Text style={styles.switchLabel}>{label}</Text>
          <Switch
            value={value}
            onValueChange={onChange}
            trackColor={{ false: '#d1d5db', true: '#007AFF' }}
            thumbColor="#fff"
          />
        </View>
      )}
    />
  );
}

// Custom Checkbox
export function FormCheckbox({ control, name, label }: { control: Control<any>; name: string; label: string }) {
  return (
    <Controller
      control={control}
      name={name}
      render={({ field: { value, onChange }, fieldState: { error } }) => (
        <View>
          <Pressable
            onPress={() => onChange(!value)}
            style={styles.checkboxRow}
            accessibilityRole="checkbox"
            accessibilityState={{ checked: value }}
          >
            <View style={[styles.checkbox, value && styles.checkboxChecked]}>
              {value && <Ionicons name="checkmark" size={14} color="#fff" />}
            </View>
            <Text style={styles.checkboxLabel}>{label}</Text>
          </Pressable>
          {error && <Text style={styles.error}>{error.message}</Text>}
        </View>
      )}
    />
  );
}

const styles = StyleSheet.create({
  switchRow: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center', paddingVertical: 8 },
  switchLabel: { fontSize: 16, color: '#111' },
  checkboxRow: { flexDirection: 'row', alignItems: 'center', gap: 10 },
  checkbox: { width: 20, height: 20, borderRadius: 4, borderWidth: 2, borderColor: '#007AFF', justifyContent: 'center', alignItems: 'center' },
  checkboxChecked: { backgroundColor: '#007AFF' },
  checkboxLabel: { fontSize: 15, color: '#111', flex: 1 },
  error: { fontSize: 12, color: '#ef4444', marginTop: 4, marginLeft: 30 },
});
```

---

### 5.5 Radio Buttons

React Native has no native radio — build it with `Pressable`.

```tsx
import { View, Text, Pressable, StyleSheet } from 'react-native';
import { Controller, Control } from 'react-hook-form';

type Option<T> = { label: string; value: T; description?: string };

type RadioGroupProps<T> = {
  control: Control<any>;
  name: string;
  label: string;
  options: Option<T>[];
};

export function RadioGroup<T extends string | number>({
  control, name, label, options,
}: RadioGroupProps<T>) {
  return (
    <Controller
      control={control}
      name={name}
      render={({ field: { value, onChange }, fieldState: { error } }) => (
        <View>
          <Text style={styles.groupLabel}>{label}</Text>
          {options.map(option => {
            const isSelected = value === option.value;
            return (
              <Pressable
                key={String(option.value)}
                onPress={() => onChange(option.value)}
                style={styles.option}
                accessibilityRole="radio"
                accessibilityState={{ checked: isSelected }}
              >
                <View style={[styles.radio, isSelected && styles.radioSelected]}>
                  {isSelected && <View style={styles.radioInner} />}
                </View>
                <View style={{ flex: 1 }}>
                  <Text style={styles.optionLabel}>{option.label}</Text>
                  {option.description && (
                    <Text style={styles.optionDesc}>{option.description}</Text>
                  )}
                </View>
              </Pressable>
            );
          })}
          {error && <Text style={styles.error}>{error.message}</Text>}
        </View>
      )}
    />
  );
}

const styles = StyleSheet.create({
  groupLabel: { fontSize: 14, fontWeight: '600', marginBottom: 10, color: '#333' },
  option: { flexDirection: 'row', alignItems: 'center', gap: 12, paddingVertical: 10 },
  radio: { width: 20, height: 20, borderRadius: 10, borderWidth: 2, borderColor: '#ccc', justifyContent: 'center', alignItems: 'center' },
  radioSelected: { borderColor: '#007AFF' },
  radioInner: { width: 10, height: 10, borderRadius: 5, backgroundColor: '#007AFF' },
  optionLabel: { fontSize: 15, color: '#111' },
  optionDesc: { fontSize: 12, color: '#666', marginTop: 2 },
  error: { fontSize: 12, color: '#ef4444', marginTop: 4 },
});

// Usage
<RadioGroup
  control={control}
  name="role"
  label="Account Type"
  options={[
    { label: 'Buyer', value: 'buyer', description: 'Browse and purchase products' },
    { label: 'Seller', value: 'seller', description: 'List and sell products' },
  ]}
/>
```

---

### 5.6 Multi-Select

Select multiple values from a list with toggle behavior.

```tsx
import { FlatList, Pressable, Text, View, StyleSheet } from 'react-native';
import { Controller, Control } from 'react-hook-form';
import { Ionicons } from '@expo/vector-icons';

type Option = { label: string; value: string };

type MultiSelectProps = {
  control: Control<any>;
  name: string;
  label: string;
  options: Option[];
};

export function MultiSelect({ control, name, label, options }: MultiSelectProps) {
  return (
    <Controller
      control={control}
      name={name}
      render={({ field: { value = [], onChange }, fieldState: { error } }) => (
        <View>
          <Text style={styles.label}>{label}</Text>
          <View style={styles.grid}>
            {options.map(option => {
              const isSelected = (value as string[]).includes(option.value);
              return (
                <Pressable
                  key={option.value}
                  onPress={() => {
                    const current = value as string[];
                    onChange(
                      isSelected
                        ? current.filter(v => v !== option.value) // deselect
                        : [...current, option.value]              // select
                    );
                  }}
                  style={[styles.chip, isSelected && styles.chipSelected]}
                >
                  {isSelected && (
                    <Ionicons name="checkmark" size={14} color="#fff" style={{ marginRight: 4 }} />
                  )}
                  <Text style={[styles.chipText, isSelected && styles.chipTextSelected]}>
                    {option.label}
                  </Text>
                </Pressable>
              );
            })}
          </View>
          {error && <Text style={styles.error}>{error.message}</Text>}
        </View>
      )}
    />
  );
}

const styles = StyleSheet.create({
  label: { fontSize: 14, fontWeight: '600', marginBottom: 10, color: '#333' },
  grid: { flexDirection: 'row', flexWrap: 'wrap', gap: 8 },
  chip: {
    flexDirection: 'row',
    alignItems: 'center',
    paddingHorizontal: 14,
    paddingVertical: 8,
    borderRadius: 20,
    borderWidth: 1.5,
    borderColor: '#ddd',
    backgroundColor: '#f9fafb',
  },
  chipSelected: { backgroundColor: '#007AFF', borderColor: '#007AFF' },
  chipText: { fontSize: 14, color: '#444' },
  chipTextSelected: { color: '#fff', fontWeight: '600' },
  error: { fontSize: 12, color: '#ef4444', marginTop: 6 },
});
```

---

### 5.7 Search Inputs

A search input with debouncing, clear button, and loading indicator.

```tsx
import { View, TextInput, Pressable, ActivityIndicator, StyleSheet } from 'react-native';
import { Ionicons } from '@expo/vector-icons';
import { useState, useCallback } from 'react';
import { useDebounce } from '@/hooks/useDebounce';

type SearchInputProps = {
  onSearch: (query: string) => void;
  isLoading?: boolean;
  placeholder?: string;
  debounceMs?: number;
};

export function SearchInput({
  onSearch,
  isLoading = false,
  placeholder = 'Search...',
  debounceMs = 400,
}: SearchInputProps) {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, debounceMs);

  // Trigger search when debounced value changes
  useEffect(() => {
    onSearch(debouncedQuery);
  }, [debouncedQuery, onSearch]);

  const handleClear = () => {
    setQuery('');
    onSearch('');
  };

  return (
    <View style={styles.container}>
      <Ionicons name="search-outline" size={18} color="#999" style={styles.searchIcon} />

      <TextInput
        style={styles.input}
        value={query}
        onChangeText={setQuery}
        placeholder={placeholder}
        placeholderTextColor="#999"
        returnKeyType="search"
        autoCorrect={false}
        autoCapitalize="none"
        clearButtonMode="never"  // we use custom clear button
      />

      {isLoading && (
        <ActivityIndicator size="small" color="#999" style={styles.rightIcon} />
      )}

      {!isLoading && query.length > 0 && (
        <Pressable onPress={handleClear} style={styles.rightIcon} hitSlop={8}>
          <Ionicons name="close-circle" size={18} color="#999" />
        </Pressable>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: '#f3f4f6',
    borderRadius: 10,
    paddingHorizontal: 10,
    height: 44,
  },
  searchIcon: { marginRight: 6 },
  input: { flex: 1, fontSize: 16, color: '#111' },
  rightIcon: { marginLeft: 6 },
});
```

---

### 5.8 OTP Inputs

A 6-digit OTP input where each digit has its own box — tapping the first box focuses it, and digits auto-advance.

```tsx
import { useRef, useState } from 'react';
import { View, TextInput, StyleSheet, NativeSyntheticEvent, TextInputKeyPressEventData } from 'react-native';

type OTPInputProps = {
  length?: number;
  onComplete: (otp: string) => void;
};

export function OTPInput({ length = 6, onComplete }: OTPInputProps) {
  const [values, setValues] = useState<string[]>(Array(length).fill(''));
  const refs = useRef<(TextInput | null)[]>(Array(length).fill(null));

  const handleChange = (text: string, index: number) => {
    const digit = text.replace(/[^0-9]/g, '').slice(-1); // only last digit
    const newValues = [...values];
    newValues[index] = digit;
    setValues(newValues);

    if (digit && index < length - 1) {
      refs.current[index + 1]?.focus(); // advance to next box
    }

    const otp = newValues.join('');
    if (otp.length === length && !otp.includes('')) {
      onComplete(otp);
    }
  };

  const handleKeyPress = (
    e: NativeSyntheticEvent<TextInputKeyPressEventData>,
    index: number
  ) => {
    // Backspace on empty field — go back to previous
    if (e.nativeEvent.key === 'Backspace' && !values[index] && index > 0) {
      refs.current[index - 1]?.focus();
    }
  };

  const handlePaste = (text: string) => {
    // Handle paste — fill all boxes at once
    const digits = text.replace(/[^0-9]/g, '').slice(0, length).split('');
    if (digits.length === length) {
      setValues(digits);
      refs.current[length - 1]?.focus();
      onComplete(digits.join(''));
    }
  };

  return (
    <View style={styles.container}>
      {values.map((val, index) => (
        <TextInput
          key={index}
          ref={el => { refs.current[index] = el; }}
          style={[styles.box, val && styles.boxFilled]}
          value={val}
          onChangeText={text => {
            if (text.length > 1) {
              handlePaste(text); // paste detection
            } else {
              handleChange(text, index);
            }
          }}
          onKeyPress={e => handleKeyPress(e, index)}
          keyboardType="number-pad"
          maxLength={1}
          textAlign="center"
          selectTextOnFocus
          autoComplete={index === 0 ? 'one-time-code' : 'off'} // iOS SMS autofill
        />
      ))}
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    gap: 10,
    justifyContent: 'center',
  },
  box: {
    width: 48,
    height: 56,
    borderWidth: 1.5,
    borderColor: '#ddd',
    borderRadius: 8,
    fontSize: 22,
    fontWeight: '700',
    color: '#111',
    backgroundColor: '#fff',
  },
  boxFilled: {
    borderColor: '#007AFF',
    backgroundColor: '#eff6ff',
  },
});

// Usage
<OTPInput length={6} onComplete={(otp) => verifyOtp(otp)} />
```

> `autoComplete="one-time-code"` on the first input enables iOS to suggest the OTP from an incoming SMS automatically — a major UX win.

---

*End of Module 7*
