# Quick Reference: SMCH Mobile Ticket Management API

## 🚀 Quick Start

### 1. Make an API Request (ALWAYS use this)
```typescript
import axiosInstance from '../utils/axiosConfig';

// GET request
const response = await axiosInstance.get('/tickets');

// POST request with data
const response = await axiosInstance.post('/tickets', {
  title: 'Device Issue',
  description: 'Device not working'
});

// Error handling
try {
  const response = await axiosInstance.get('/tickets');
  console.log(response.data);
} catch (error) {
  console.error(error.response?.data?.message || error.message);
}
```

### 2. Create a Ticket
```typescript
import { useTicketManagement } from '../hooks/useTicketManagement';

function CreateTicketForm() {
  const { createTicket, loading, error } = useTicketManagement();
  
  const handleSubmit = async (description) => {
    try {
      const ticket = await createTicket({
        title: 'Device Issue Report',
        description,
        device_id: 123,
        status: 'pending'
      });
      console.log('Ticket created:', ticket.id);
    } catch (err) {
      console.error('Error:', err.message);
    }
  };
  
  return (
    <Button 
      onPress={() => handleSubmit('Device not working')}
      disabled={loading}
      title={loading ? 'Creating...' : 'Create Ticket'}
    />
  );
}
```

### 3. Upload an Image
```typescript
import axiosInstance from '../utils/axiosConfig';

async function uploadImage(imageUri) {
  const formData = new FormData();
  formData.append('image', {
    uri: imageUri,
    type: 'image/jpeg',
    name: 'report.jpg'
  });
  formData.append('folder', 'report_images');
  
  const response = await axiosInstance.post('/images/upload', formData, {
    headers: { 'Content-Type': 'multipart/form-data' }
  });
  
  return response.data.path;
}
```

### 4. Show Loading State
```typescript
import { SkeletonLoader, SkeletonParagraph } from '../components/SkeletonLoader';

function TicketList({ tickets, loading }) {
  if (loading) {
    return (
      <View>
        <SkeletonLoader width="100%" height={60} />
        <SkeletonLoader width="100%" height={60} />
        <SkeletonParagraph lines={2} />
      </View>
    );
  }
  
  return <FlatList data={tickets} renderItem={renderTicket} />;
}
```

### 5. Display Status Badge
```typescript
import { StatusBadge } from '../components/StatusBadge';

function TicketCard({ ticket }) {
  return (
    <View>
      <Text>{ticket.title}</Text>
      <StatusBadge 
        status={ticket.status}  // 'open', 'in-progress', 'resolved', etc.
        size="small"
      />
    </View>
  );
}
```

---

## ⚠️ Common Mistakes (DON'T DO THESE)

### ❌ WRONG: Using raw axios
```typescript
import axios from 'axios';

// BAD - Missing headers!
axios.post('/api/tickets', data);
```

### ✅ CORRECT: Use axiosInstance
```typescript
import axiosInstance from '../utils/axiosConfig';

// GOOD - Headers added automatically
axiosInstance.post('/tickets', data);
```

---

### ❌ WRONG: Parallel API calls
```typescript
// BAD - Causes ERR_CONNECTION_CLOSED on ngrok
Promise.all([
  axiosInstance.post('/images/upload', img),
  axiosInstance.post('/tickets', data)
]);
```

### ✅ CORRECT: Sequential API calls
```typescript
// GOOD - One at a time
const imagePath = await axiosInstance.post('/images/upload', img);
await axiosInstance.post('/tickets', { ...data, image: imagePath });
```

---

### ❌ WRONG: Manual token headers
```typescript
// BAD - Error prone, violates DRY
const token = await AsyncStorage.getItem('token');
axios.post('/tickets', data, {
  headers: { Authorization: `Bearer ${token}` }
});
```

### ✅ CORRECT: Automatic token injection
```typescript
// GOOD - Token added automatically
axiosInstance.post('/tickets', data);
```

---

### ❌ WRONG: Forgetting ngrok header
```typescript
// BAD - App crashes with JSON parse error
const response = await axios.post('/api/tickets', data);
```

### ✅ CORRECT: axiosInstance handles it
```typescript
// GOOD - ngrok header added automatically
const response = await axiosInstance.post('/tickets', data);
```

---

## 🔄 Sequential Flow Pattern

Always follow this pattern for operations with multiple steps:

```typescript
const handleComplexOperation = async () => {
  try {
    // STEP 1: Validate
    if (!isValid) throw new Error('Validation failed');
    
    // STEP 2: Upload/prepare data
    const result1 = await axiosInstance.post('/step1', data1);
    
    // STEP 3: Wait, then proceed
    const result2 = await axiosInstance.post('/step2', {
      ...data2,
      dependency: result1.data.id
    });
    
    // STEP 4: Final step
    const result3 = await axiosInstance.post('/step3', {
      ...data3,
      dependencies: [result1.data.id, result2.data.id]
    });
    
    // STEP 5: Success
    showSuccess('Operation completed!');
  } catch (err) {
    // Handle all errors at once
    showError(err.message);
  }
};
```

---

## 📊 Status Values Reference

```typescript
type TicketStatus = 
  | 'open'           // Blue (#2196F3)
  | 'in-progress'    // Orange (#FF9800)
  | 'resolved'       // Green (#4CAF50)
  | 'pending'        // Yellow (#FBC02D)
  | 'repair'         // Dark Blue (#1976D2)
  | 'decommissioned' // Red (#D32F2F)
```

---

## 🎨 Component Props Reference

### StatusBadge
```typescript
interface StatusBadgeProps {
  status: 'open' | 'in-progress' | 'resolved' | 'pending' | 'repair' | 'decommissioned';
  size?: 'small' | 'medium' | 'large'; // default: 'medium'
  style?: StyleProp<ViewStyle>;
}

// Usage
<StatusBadge status="open" size="small" />
```

### SkeletonLoader
```typescript
interface SkeletonLoaderProps {
  width?: number | string;       // default: '100%'
  height?: number | string;      // default: 16
  borderRadius?: number;          // default: 4
  style?: StyleProp<ViewStyle>;
}

// Usage
<SkeletonLoader width={200} height={24} borderRadius={8} />
```

### SkeletonParagraph
```typescript
interface SkeletonParagraphProps {
  lines?: number;  // default: 3
}

// Usage
<SkeletonParagraph lines={5} />
```

---

## 🔍 Debugging Tips

### Check Token in AsyncStorage
```typescript
import AsyncStorage from '@react-native-async-storage/async-storage';

// In component or debug screen
const token = await AsyncStorage.getItem('token');
console.log('Token:', token ? 'Present' : 'Missing');
```

### Monitor axios Requests
```typescript
import axiosInstance from '../utils/axiosConfig';

// Log all requests
axiosInstance.interceptors.request.use(config => {
  console.log('Request:', config.method, config.url);
  console.log('Headers:', config.headers);
  return config;
});

// Log all responses
axiosInstance.interceptors.response.use(response => {
  console.log('Response:', response.status, response.data);
  return response;
}, error => {
  console.error('Error:', error.response?.status, error.response?.data);
  return Promise.reject(error);
});
```

### Verify ngrok Headers
```typescript
// Check if ngrok header is being sent
// Open dev tools network tab and look for:
// ngrok-skip-browser-warning: true
// Authorization: Bearer <token>
```

---

## 🆘 Troubleshooting

| Error | Cause | Solution |
|-------|-------|----------|
| JSON.parse: unexpected character | Missing ngrok header | Use axiosInstance |
| ERR_CONNECTION_CLOSED | Parallel requests | Use sequential async/await |
| 401 Unauthorized | No/expired token | Check AsyncStorage, re-login |
| 403 Forbidden | Auth issue | Check backend permissions |
| Network Error | Connection issue | Check ngrok tunnel, firewall |
| timeout of 30000ms exceeded | Slow network | Check ngrok latency |

---

## 📚 Files to Study

- [MOBILE_TICKET_REFACTOR.md](./MOBILE_TICKET_REFACTOR.md) - Complete architecture guide
- [IMPLEMENTATION_SUMMARY.md](./IMPLEMENTATION_SUMMARY.md) - What changed
- `utils/axiosConfig.ts` - Axios configuration
- `hooks/useTicketManagement.ts` - Ticket management hook
- `components/SkeletonLoader.tsx` - Loading component
- `components/StatusBadge.tsx` - Status display component
- `app/reportCreate.tsx` - Full example implementation

---

## 💡 Best Practices

1. **Always validate before API calls**
   ```typescript
   if (!email || !email.includes('@')) {
     showError('Invalid email');
     return;
   }
   ```

2. **Use loading states**
   ```typescript
   const { loading } = useTicketManagement();
   <Button disabled={loading} title={loading ? 'Saving...' : 'Save'} />
   ```

3. **Show error messages**
   ```typescript
   const { error } = useTicketManagement();
   {error && <ErrorContainer message={error} />}
   ```

4. **Clean up on unmount**
   ```typescript
   useEffect(() => {
     return () => {
       setIsMounted(false); // Prevent state updates on unmounted component
     };
   }, []);
   ```

5. **Test with slow networks**
   - Use Chrome DevTools network throttling
   - Test with actual ngrok tunnel latency
   - Verify skeleton loaders appear

---

**Last Updated:** May 12, 2026
**Status:** ✅ Complete
**Version:** 1.0
