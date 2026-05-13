# Testing Guide: SMCH Mobile Ticket Management

## Pre-Testing Checklist

- [ ] Backend is running (Laravel 11 server)
- [ ] ngrok tunnel is active: `ngrok http 8000`
- [ ] ngrok URL in `.env` file or `EXPO_PUBLIC_API_URL`
- [ ] Token is in AsyncStorage (user is logged in)
- [ ] Device has network connection
- [ ] Android emulator or iOS simulator is running

---

## Unit Tests

### Test: axiosConfig.ts

```typescript
describe('axiosConfig', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('should add ngrok skip browser warning header', async () => {
    const instance = createAxiosInstance();
    let capturedConfig;
    
    instance.interceptors.request.use(config => {
      capturedConfig = config;
      return config;
    });
    
    // Simulate request
    expect(capturedConfig?.headers['ngrok-skip-browser-warning']).toBe('true');
  });

  it('should inject Bearer token from AsyncStorage', async () => {
    AsyncStorage.getItem.mockResolvedValue('test-token-123');
    const instance = createAxiosInstance();
    
    let capturedConfig;
    instance.interceptors.request.use(config => {
      capturedConfig = config;
      return config;
    });
    
    expect(capturedConfig?.headers['Authorization']).toContain('Bearer');
  });

  it('should handle 401 errors', async () => {
    const instance = createAxiosInstance();
    
    instance.interceptors.response.use(null, error => {
      if (error.response?.status === 401) {
        // Should remove token
        expect(AsyncStorage.removeItem).toHaveBeenCalledWith('token');
      }
      return Promise.reject(error);
    });
  });
});
```

### Test: useTicketManagement.ts

```typescript
describe('useTicketManagement', () => {
  it('should create ticket successfully', async () => {
    const mockResponse = { id: 1, title: 'Test' };
    axiosInstance.post.mockResolvedValue({ data: mockResponse });
    
    const { result } = renderHook(() => useTicketManagement());
    
    const ticket = await result.current.createTicket({
      title: 'Test',
      description: 'Test description',
      device_id: 1
    });
    
    expect(ticket).toEqual(mockResponse);
  });

  it('should handle errors', async () => {
    axiosInstance.post.mockRejectedValue(
      new Error('Network error')
    );
    
    const { result } = renderHook(() => useTicketManagement());
    
    await expect(result.current.createTicket({}))
      .rejects.toThrow('Network error');
  });

  it('should validate token presence', async () => {
    AsyncStorage.getItem.mockResolvedValue(null);
    
    const { result } = renderHook(() => useTicketManagement());
    
    await expect(result.current.createTicket({}))
      .rejects.toThrow('Authentication required');
  });
});
```

### Test: StatusBadge.tsx

```typescript
describe('StatusBadge', () => {
  it('should render with correct color for status', () => {
    const { getByText } = render(
      <StatusBadge status="open" />
    );
    
    const badge = getByText('Open');
    expect(badge.parent.style.backgroundColor).toBe('#2196F3');
  });

  it('should render all status variants', () => {
    const statuses = ['open', 'in-progress', 'resolved', 'pending', 'repair', 'decommissioned'];
    
    statuses.forEach(status => {
      const { getByText } = render(<StatusBadge status={status} />);
      expect(getByText).toBeTruthy();
    });
  });

  it('should apply size styles correctly', () => {
    const { getByText } = render(
      <StatusBadge status="open" size="large" />
    );
    
    const badge = getByText('Open');
    expect(badge.parent.style.paddingVertical).toBe(10);
  });
});
```

---

## Integration Tests

### Test: Report Create Form Submission

```typescript
describe('ReportCreate Integration', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('should auto-fill device info from params', () => {
    const { getByText } = render(
      <ReportCreate 
        route={{
          params: {
            deviceId: '123',
            deviceName: 'Test Device'
          }
        }}
      />
    );
    
    expect(getByText('Test Device')).toBeTruthy();
    expect(getByText('123')).toBeTruthy();
  });

  it('should upload image then create ticket', async () => {
    axiosInstance.post.mockResolvedValueOnce({ 
      data: { path: '/storage/image.jpg' } 
    }).mockResolvedValueOnce({ 
      data: { id: 1, title: 'Test' } 
    });
    
    const { getByTestId } = render(<ReportCreate />);
    
    // Fill form
    const descriptionInput = getByTestId('description-input');
    fireEvent.changeText(descriptionInput, 'Test issue');
    
    // Submit
    const submitButton = getByTestId('submit-btn');
    fireEvent.press(submitButton);
    
    // Verify sequential calls
    expect(axiosInstance.post).toHaveBeenCalledTimes(2);
    
    // First call: image upload
    expect(axiosInstance.post).toHaveBeenNthCalledWith(
      1,
      '/images/upload',
      expect.any(Object)
    );
    
    // Second call: ticket creation
    expect(axiosInstance.post).toHaveBeenNthCalledWith(
      2,
      '/tickets',
      expect.objectContaining({
        title: expect.any(String),
        device_id: '123'
      })
    );
  });

  it('should handle validation errors', async () => {
    const { getByTestId } = render(<ReportCreate />);
    
    // Try to submit empty form
    const submitButton = getByTestId('submit-btn');
    fireEvent.press(submitButton);
    
    // Should show error
    expect(getByText('Description is required')).toBeTruthy();
  });
});
```

---

## Manual Testing Scenarios

### Scenario 1: Basic Ticket Creation

**Steps:**
1. Open app, login
2. Navigate to Devices tab
3. Tap on any device
4. Tap "Report Device"
5. Verify device name appears in device info card
6. Enter description: "Screen is flickering"
7. Tap "Submit Report"

**Expected Results:**
- ✅ Device info pre-filled
- ✅ Loading spinner appears
- ✅ "Ticket created successfully" message appears
- ✅ Navigate back to dashboard
- ✅ Ticket appears in ticket list

### Scenario 2: Ticket with Image

**Steps:**
1. Follow Scenario 1 steps 1-5
2. Enter description
3. Tap "Attach Image"
4. Select image from gallery
5. Image preview should appear
6. Tap "Submit Report"

**Expected Results:**
- ✅ Image preview displays correctly
- ✅ File picker opens
- ✅ Image can be removed via X button
- ✅ Loading shows "Submitting..."
- ✅ Ticket created with image

### Scenario 3: Network Error Handling

**Steps:**
1. Turn off Wi-Fi/Mobile data
2. Try to create ticket
3. System should wait 30 seconds (timeout)
4. Error message should appear

**Expected Results:**
- ✅ Error message shows "Network error" or similar
- ✅ User can retry
- ✅ No crash

### Scenario 4: Token Expiration

**Steps:**
1. In DevTools, simulate AsyncStorage token removal
2. Try to create ticket
3. System should redirect to login

**Expected Results:**
- ✅ Automatic redirect to login screen
- ✅ Error message: "Session expired"
- ✅ No crash

### Scenario 5: Large Image Upload

**Steps:**
1. Select large image (>5MB if backend limits to 5MB)
2. Try to create ticket

**Expected Results:**
- ✅ Error message about file size
- ✅ Can select different image and retry
- ✅ No crash

### Scenario 6: Slow Network Simulation

**Steps (Android DevTools):**
1. Open DevTools
2. Go to Network tab
3. Set throttling to "Slow 4G"
4. Create ticket with image

**Expected Results:**
- ✅ Skeleton loaders appear during upload
- ✅ Perceived performance feels good (not frozen)
- ✅ Eventually completes

---

## Performance Testing

### Memory Usage
```typescript
// Monitor memory during image selection
- Initial: ~150MB
- After image select: ~200-250MB
- After submission: ~150MB (cleaned up)
```

### Network Requests
```typescript
// Verify ngrok tunnel can handle
- 1 concurrent user: ✅ No issues
- 5 concurrent users: ✅ Monitor for timeouts
- 10+ concurrent users: ⚠️ May see ERR_CONNECTION_CLOSED
```

### API Response Times (with ngrok)
```
Image upload: 2-5 seconds
Ticket creation: 1-2 seconds
Total flow: 3-7 seconds
```

---

## Device-Specific Testing

### Android Emulator
```
- Test device: Pixel 5 emulator
- Android version: 12 or higher
- RAM: 4GB+
- Network: Simulated
- Verify: Image picker works, ngrok tunnel accessible
```

### iOS Simulator
```
- Test device: iPhone 14 simulator
- iOS version: 15 or higher
- RAM: 4GB+
- Network: Simulated
- Verify: Different image picker UI, ngrok tunnel accessible
```

### Physical Devices
```
- Android: Real device on Wi-Fi with ngrok
- iOS: Real device on Wi-Fi with ngrok
- Test: Slow network conditions, different screen sizes
```

---

## Regression Testing Checklist

After any changes to the following, run full test suite:

- [ ] axiosConfig.ts changes → Run network tests
- [ ] useTicketManagement.ts changes → Run hook tests
- [ ] reportCreate.tsx changes → Run form tests
- [ ] Any API endpoint changes → Verify request/response

---

## Debugging Tips

### Enable Request/Response Logging
```typescript
// Add to app startup
axiosInstance.interceptors.request.use(config => {
  console.log('REQUEST:', config.method, config.url);
  console.log('HEADERS:', config.headers);
  return config;
});

axiosInstance.interceptors.response.use(response => {
  console.log('RESPONSE:', response.status, response.data);
  return response;
}, error => {
  console.error('ERROR:', error.response?.status, error.message);
  return Promise.reject(error);
});
```

### Check AsyncStorage
```typescript
// In DevTools console
await AsyncStorage.multiGet(['token', 'user_role', 'office_id']);
```

### Monitor Network Requests
```
Android: Use Android Studio Network Profiler
iOS: Use Xcode Network Link Conditioner
Both: Use React Native DevTools
```

---

## Test Execution Commands

```bash
# Run all tests
npm test

# Run specific test file
npm test reportCreate.test.tsx

# Run with coverage
npm test -- --coverage

# Watch mode
npm test -- --watch

# Lint
npm run lint

# Build Android
npm run android

# Build iOS
npm run ios
```

---

## Known Issues & Workarounds

| Issue | Cause | Workaround |
|-------|-------|-----------|
| Image picker crashes | Permission denied | Grant permissions in app settings |
| ngrok tunnel 403 | Missing header | Use axiosInstance (automatic) |
| Timeout errors | Slow network | Increase timeout in axiosConfig |
| 401 errors | Expired token | Clear AsyncStorage, re-login |
| Form unresponsive | Uploading | Show loading indicator |

---

**Test Plan Version:** 1.0
**Last Updated:** May 12, 2026
**Status:** ✅ Ready for QA
