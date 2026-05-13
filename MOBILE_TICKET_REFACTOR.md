# SMCH Mobile App - Ticket Management Refactor Guide

## Overview

This document outlines the enterprise-grade refactoring of the Ticket Management interface for the SMCH Hardware Monitoring System mobile app (React Native). The refactor includes:

1. **Axios Configuration with ngrok Support** - Critical headers to prevent browser warning HTML responses
2. **Sequential API Calls** - Prevents `ERR_CONNECTION_CLOSED` on ngrok tunnel
3. **Professional UI Components** - Skeleton loaders, status badges, and enterprise styling
4. **Bearer Token Authentication** - Automatic token injection and 401 error handling

---

## Task 1: Auto-Fill Ticket Flow ✅

### What Changed

**Before:**
```typescript
// Old flow - manual device selection
router.replace('/reportCreate');
```

**After:**
```typescript
// New flow - auto-filled device info
router.push({
  pathname: '/reportCreate',
  params: {
    deviceId: selectedDevice.id,
    deviceName: selectedDevice.name
  }
});
```

### How It Works

1. **Devices Screen** (`app/(tabs)/devices.tsx`)
   - User taps "Report Device" button on any device card
   - Screen captures `deviceId` and `deviceName`
   - Passes as route params via `expo-router`

2. **Report Create Screen** (`app/reportCreate.tsx`)
   - Automatically receives device info via `useLocalSearchParams()`
   - Displays device in pre-filled card
   - User only fills description and optional image

3. **Field Pre-Population**
   ```typescript
   const { deviceId, deviceName } = useLocalSearchParams();
   
   // Auto-fills ticket title
   title: `Issue Report - ${deviceName || 'Device'}`
   ```

**Benefits:**
- Technician standing in front of device needs only 3 taps
- Reduces data entry errors
- Faster ticket creation in field conditions

---

## Task 2: UI/UX Enterprise Overhaul ✅

### Component Library

#### 1. **Skeleton Loader** (`components/SkeletonLoader.tsx`)
```typescript
import { SkeletonLoader, SkeletonParagraph } from '../components/SkeletonLoader';

// Single line skeleton
<SkeletonLoader width="100%" height={16} borderRadius={4} />

// Multi-line paragraph
<SkeletonParagraph lines={3} />
```

**Why It Matters:**
- ngrok tunnel latency can be 2-5 seconds
- Skeleton prevents users from thinking app is frozen
- Improves perceived performance significantly

#### 2. **Status Badge** (`components/StatusBadge.tsx`)
```typescript
import { StatusBadge } from '../components/StatusBadge';

// Color-coded status chips
<StatusBadge status="open" size="medium" />
<StatusBadge status="in-progress" size="small" />
<StatusBadge status="resolved" size="large" />
```

**Color Scheme:**
- **Open** (Blue: #2196F3) - New, unassigned
- **In Progress** (Orange: #FF9800) - Currently being worked on
- **Resolved** (Green: #4CAF50) - Completed
- **Pending** (Yellow: #FBC02D) - Waiting for action
- **Under Repair** (Dark Blue: #1976D2) - Device in maintenance
- **Decommissioned** (Red: #D32F2F) - No longer in use

#### 3. **Professional Input Forms**
```typescript
<View style={styles.deviceInfoCard}>
  <Text style={styles.deviceLabel}>Device</Text>
  <Text style={styles.deviceName}>{deviceName}</Text>
  <View style={styles.deviceIdContainer}>
    <Ionicons name="information-circle" size={16} />
    <Text style={styles.deviceId}>ID: {deviceId}</Text>
  </View>
</View>
```

**Features:**
- Left border accent (#1976d2)
- Icon-based visual hierarchy
- Shadow depth for emphasis

#### 4. **Sequential Loading Pattern**
```typescript
// All requests use async/await sequentially
// NEVER fire multiple requests in parallel

// 1. Upload image (sequential)
await axiosInstance.post('/images/upload', imgForm);

// 2. Wait for completion, then create ticket
await createTicket(ticketPayload);

// 3. Show success, navigate away
```

---

## Task 3: Authentication Stability ✅

### Axios Configuration with ngrok Support

**File:** `utils/axiosConfig.ts`

```typescript
import axios from 'axios';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { API_URL } from './api';

export const createAxiosInstance = () => {
  const instance = axios.create({
    baseURL: API_URL,
    timeout: 30000,
  });

  // Request interceptor
  instance.interceptors.request.use(
    async (config) => {
      // CRITICAL: ngrok skip-browser-warning header
      config.headers['ngrok-skip-browser-warning'] = 'true';
      
      // Attach Bearer token
      const token = await AsyncStorage.getItem('token');
      if (token) {
        config.headers['Authorization'] = `Bearer ${token}`;
      }
      
      return config;
    },
    (error) => Promise.reject(error)
  );

  // Response interceptor handles 401
  instance.interceptors.response.use(
    (response) => response,
    (error) => {
      if (error.response?.status === 401) {
        AsyncStorage.removeItem('token');
        // Optionally redirect to login
      }
      return Promise.reject(error);
    }
  );

  return instance;
};

export const axiosInstance = createAxiosInstance();
```

### Critical: The ngrok Header

**Problem:**
```
Without: ngrok-skip-browser-warning header
↓
ngrok returns HTML warning page (403)
↓
App tries to parse HTML as JSON
↓
app crashes with: "JSON.parse: unexpected character"
```

**Solution:**
```typescript
// Add to EVERY axios request
headers: {
  'ngrok-skip-browser-warning': 'true'
}
```

**Applied Automatically:**
Our axios config adds this to all requests via interceptor - no manual header needed!

### Bearer Token Usage

**Automatic Injection:**
```typescript
// No need to manually add headers anymore
await axiosInstance.post('/tickets', payload);

// Automatically includes:
// Authorization: Bearer <token>
// ngrok-skip-browser-warning: true
```

**Old Way (DON'T USE):**
```typescript
// ❌ WRONG - Manual headers (error-prone)
axios.post(url, data, {
  headers: {
    Authorization: `Bearer ${token}`,
    'ngrok-skip-browser-warning': 'true'
  }
});
```

**New Way (USE THIS):**
```typescript
// ✅ CORRECT - Automatic via interceptor
axiosInstance.post('/tickets', data);
```

---

## New Hook: useTicketManagement ✅

**File:** `hooks/useTicketManagement.ts`

Handles sequential ticket creation with proper error handling:

```typescript
import { useTicketManagement } from '../hooks/useTicketManagement';

function MyComponent() {
  const { createTicket, loading, error, success } = useTicketManagement();
  
  const handleCreate = async () => {
    try {
      const response = await createTicket({
        title: 'Issue Report - Device Name',
        description: 'Details here',
        device_id: 123,
        status: 'pending'
      });
      
      console.log('Ticket created:', response.id);
    } catch (err) {
      console.error('Failed:', err.message);
    }
  };
  
  return (
    <TouchableOpacity 
      onPress={handleCreate}
      disabled={loading}
    >
      <Text>{loading ? 'Creating...' : 'Create Ticket'}</Text>
    </TouchableOpacity>
  );
}
```

**Key Features:**
- Sequential request handling
- Automatic token attachment
- Error message extraction
- Loading state management
- Type-safe response handling

---

## File Structure

```
smch-mobile-app/
├── utils/
│   ├── api.ts                    # API URL configuration
│   ├── axiosConfig.ts            # ✅ NEW - Axios with ngrok headers
│   └── apiClient.ts              # Legacy API client (deprecated)
├── components/
│   ├── SkeletonLoader.tsx         # ✅ NEW - Loading placeholders
│   ├── StatusBadge.tsx            # ✅ NEW - Color-coded status chips
│   └── ...
├── hooks/
│   ├── useTicketManagement.ts     # ✅ NEW - Sequential ticket creation
│   └── ...
└── app/
    ├── reportCreate.tsx           # ✅ REFACTORED - Professional UI
    └── (tabs)/
        └── devices.tsx            # ✅ UPDATED - Proper params passing
```

---

## Usage Examples

### Example 1: Creating a Ticket with Image

```typescript
import { useTicketManagement } from '../hooks/useTicketManagement';
import axiosInstance from '../utils/axiosConfig';

export function CreateTicketFlow() {
  const { createTicket, loading } = useTicketManagement();
  const [image, setImage] = useState(null);
  
  const handleSubmit = async () => {
    try {
      let imagePath = null;
      
      // STEP 1: Upload image FIRST (sequential)
      if (image) {
        const formData = new FormData();
        formData.append('image', image);
        formData.append('folder', 'report_images');
        
        const imgRes = await axiosInstance.post(
          '/images/upload',
          formData
        );
        imagePath = imgRes.data.path;
      }
      
      // STEP 2: Wait for image, then create ticket
      await createTicket({
        title: 'Issue Report - Device',
        description: userInput,
        device_id: 123,
        status: 'pending',
        report_image: imagePath
      });
      
      // STEP 3: Success!
      console.log('Ticket created');
    } catch (err) {
      console.error('Error:', err.message);
    }
  };
  
  return <Button onPress={handleSubmit} disabled={loading} />;
}
```

### Example 2: Showing Status Badge

```typescript
import { StatusBadge } from '../components/StatusBadge';

export function TicketListItem({ ticket }) {
  return (
    <View>
      <Text>{ticket.title}</Text>
      <StatusBadge 
        status={ticket.status}  // 'open', 'in-progress', 'resolved'
        size="small"
      />
    </View>
  );
}
```

### Example 3: Auto-Fill Flow

```typescript
// In devices list
<TouchableOpacity onPress={() => {
  router.push({
    pathname: '/reportCreate',
    params: {
      deviceId: device.id,
      deviceName: device.name
    }
  });
}}>
  <Text>Report Issue</Text>
</TouchableOpacity>

// In reportCreate - auto-filled
const { deviceId, deviceName } = useLocalSearchParams();
// deviceName displays in device info card automatically
```

---

## Troubleshooting

### "JSON.parse: unexpected character"
**Cause:** Missing `ngrok-skip-browser-warning` header
**Solution:** Use `axiosInstance` instead of raw `axios`

### "ERR_CONNECTION_CLOSED"
**Cause:** Multiple parallel requests to ngrok tunnel
**Solution:** Use sequential async/await, not Promise.all()

### "Authorization required"
**Cause:** Token not in AsyncStorage
**Solution:** Check user is logged in before making requests

### "Cannot read property 'id' of undefined"
**Cause:** Route params not passed correctly
**Solution:** Verify using `router.push({ pathname, params })`

---

## Migration Checklist

- [x] Created `axiosConfig.ts` with ngrok headers
- [x] Created `SkeletonLoader.tsx` component
- [x] Created `StatusBadge.tsx` component
- [x] Created `useTicketManagement.ts` hook
- [x] Refactored `reportCreate.tsx` with new UI
- [x] Updated `devices.tsx` to pass params correctly
- [ ] Install/verify react-native-paper dependency
- [ ] Test on Android emulator (check ngrok headers)
- [ ] Test on iOS simulator (check ngrok headers)
- [ ] Test with actual ngrok tunnel
- [ ] Update backend routes to match new structure
- [ ] Document API endpoints being used

---

## Best Practices

1. **Always use `axiosInstance`** - Never use raw `axios` for API calls
2. **Sequential over parallel** - One request at a time for ngrok
3. **Validate early** - Check params in useEffect, not during render
4. **Show feedback** - Use skeleton loaders and snackbar notifications
5. **Handle 401s** - Redirect to login on auth errors
6. **Type your responses** - Define interfaces for API responses
7. **Test on both platforms** - iOS and Android may behave differently

---

## Next Steps

1. Install missing dependencies if needed
2. Run tests to verify no regressions
3. Deploy to staging environment
4. Load test with multiple concurrent users
5. Monitor actual ngrok tunnel performance
6. Gather technician feedback in field testing

---

**Document Version:** 1.0
**Last Updated:** May 12, 2026
**Status:** Complete Implementation
