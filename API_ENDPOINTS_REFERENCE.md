# API Endpoints Reference: SMCH Ticket Management

## Base URL Configuration

```typescript
// Development (ngrok tunnel)
API_URL: http://localhost:8000/api

// Production
API_URL: https://your-production-domain.com/api

// Set via environment variable
EXPO_PUBLIC_API_URL=<your_url>
```

---

## Authentication

### Headers (Applied Automatically via axiosConfig)
```
Authorization: Bearer <token>
ngrok-skip-browser-warning: true
Content-Type: application/json (or multipart/form-data for uploads)
```

### Token Storage
```typescript
// Stored in AsyncStorage
await AsyncStorage.getItem('token')
await AsyncStorage.setItem('token', tokenValue)
```

---

## Endpoints Used in Ticket Management

### 1. Create Ticket
**Endpoint:** `POST /tickets`

**Request:**
```json
{
  "title": "Issue Report - Device Name",
  "description": "Device is not powering on",
  "device_id": 123,
  "status": "pending",
  "report_image": "/storage/report_images/image-123.jpg" (optional)
}
```

**Response (201 Created):**
```json
{
  "id": 456,
  "title": "Issue Report - Device Name",
  "description": "Device is not powering on",
  "device_id": 123,
  "status": "pending",
  "report_image": "/storage/report_images/image-123.jpg",
  "created_at": "2026-05-12T10:30:00Z",
  "updated_at": "2026-05-12T10:30:00Z"
}
```

**Error Responses:**
```json
// 400 Bad Request
{
  "message": "Description is required",
  "errors": { "description": ["The description field is required."] }
}

// 401 Unauthorized
{
  "message": "Unauthenticated."
}

// 422 Unprocessable Entity
{
  "message": "The device_id field is required.",
  "errors": { "device_id": ["The device_id field is required."] }
}
```

### 2. Upload Image
**Endpoint:** `POST /images/upload`

**Request (multipart/form-data):**
```
Field: image (File) - JPG, PNG, GIF, WEBP
Field: folder (String) - "report_images" or "device_images"
```

**Response (200 OK):**
```json
{
  "path": "/storage/report_images/12345_report.jpg",
  "url": "http://localhost:8000/storage/report_images/12345_report.jpg",
  "message": "Image uploaded successfully"
}
```

**Error Responses:**
```json
// 422 Validation Error
{
  "message": "The image must be an image.",
  "errors": { "image": ["The image must be an image."] }
}

// 413 File Too Large
{
  "message": "File is too large. Maximum size: 5MB"
}
```

### 3. Get Tickets
**Endpoint:** `GET /tickets`

**Query Parameters:**
```
?per_page=20
?page=1
?status=pending
?device_id=123
?order_by=-created_at
```

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": 456,
      "title": "Issue Report - Device Name",
      "description": "Device is not powering on",
      "device_id": 123,
      "device": {
        "id": 123,
        "name": "Device Name",
        "type": { "id": 1, "name": "Desktop" },
        "office": { "id": 5, "name": "Main Office" }
      },
      "status": "pending",
      "report_image": "/storage/report_images/image-123.jpg",
      "created_at": "2026-05-12T10:30:00Z",
      "updated_at": "2026-05-12T10:30:00Z"
    }
  ],
  "meta": {
    "current_page": 1,
    "from": 1,
    "last_page": 5,
    "per_page": 20,
    "to": 20,
    "total": 95
  }
}
```

### 4. Get Ticket By ID
**Endpoint:** `GET /tickets/:id`

**Response (200 OK):**
```json
{
  "id": 456,
  "title": "Issue Report - Device Name",
  "description": "Device is not powering on",
  "device_id": 123,
  "device": {
    "id": 123,
    "name": "Device Name",
    "type": { "id": 1, "name": "Desktop" },
    "category": { "id": 2, "name": "Computer" },
    "office": { "id": 5, "name": "Main Office" },
    "status": "pending",
    "image": "/storage/devices/device-123.jpg"
  },
  "status": "pending",
  "report_image": "/storage/report_images/image-123.jpg",
  "created_by": { "id": 10, "name": "John Doe", "email": "john@example.com" },
  "assigned_to": null,
  "created_at": "2026-05-12T10:30:00Z",
  "updated_at": "2026-05-12T10:30:00Z"
}
```

### 5. Update Ticket Status
**Endpoint:** `PUT /tickets/:id/status`

**Request:**
```json
{
  "status": "in-progress"
}
```

**Response (200 OK):**
```json
{
  "id": 456,
  "status": "in-progress",
  "updated_at": "2026-05-12T11:45:00Z"
}
```

### 6. Get Device Information
**Endpoint:** `GET /devices/:id`

**Response (200 OK):**
```json
{
  "id": 123,
  "name": "Office Desktop PC",
  "description": "Dell OptiPlex 7090",
  "image": "/storage/devices/device-123.jpg",
  "type": {
    "id": 1,
    "name": "Desktop Computer"
  },
  "category": {
    "id": 2,
    "name": "Computer"
  },
  "office": {
    "id": 5,
    "name": "Main Office",
    "location": "Ground Floor"
  },
  "status": "resolved",
  "created_at": "2026-01-15T08:00:00Z",
  "updated_at": "2026-05-10T14:30:00Z"
}
```

---

## Implementation Example: Complete Ticket Creation

```typescript
import axiosInstance from '../utils/axiosConfig';

async function createTicketWithImage(description, imageUri, deviceId, deviceName) {
  try {
    let imagePath = null;
    
    // STEP 1: Upload image (sequential)
    if (imageUri) {
      const imageForm = new FormData();
      imageForm.append('image', {
        uri: imageUri,
        type: 'image/jpeg',
        name: 'report_image.jpg'
      });
      imageForm.append('folder', 'report_images');
      
      const imageResponse = await axiosInstance.post(
        '/images/upload',
        imageForm,
        {
          headers: { 'Content-Type': 'multipart/form-data' }
        }
      );
      
      imagePath = imageResponse.data?.path;
      console.log('Image uploaded:', imagePath);
    }
    
    // STEP 2: Create ticket (sequential - after image completes)
    const ticketPayload = {
      title: `Issue Report - ${deviceName}`,
      description: description,
      device_id: deviceId,
      status: 'pending'
    };
    
    if (imagePath) {
      ticketPayload.report_image = imagePath;
    }
    
    const ticketResponse = await axiosInstance.post('/tickets', ticketPayload);
    
    console.log('Ticket created successfully:', ticketResponse.data);
    return ticketResponse.data;
    
  } catch (error) {
    const errorMessage = 
      error.response?.data?.message ||
      error.response?.data?.errors?.[Object.keys(error.response.data.errors)[0]]?.[0] ||
      error.message ||
      'Unknown error occurred';
    
    console.error('Error creating ticket:', errorMessage);
    throw new Error(errorMessage);
  }
}

// Usage
try {
  const ticket = await createTicketWithImage(
    'Device is not responding to network pings',
    'file:///path/to/image.jpg',
    123,
    'Network Printer - Floor 2'
  );
  
  console.log(`Ticket #${ticket.id} created successfully`);
} catch (error) {
  console.error('Failed to create ticket:', error.message);
}
```

---

## Error Codes Reference

| Code | Meaning | Common Causes |
|------|---------|---------------|
| 400 | Bad Request | Invalid parameters, missing required fields |
| 401 | Unauthorized | No/expired token |
| 403 | Forbidden | User lacks permissions for action |
| 404 | Not Found | Resource doesn't exist |
| 422 | Unprocessable Entity | Validation failed (detailed error messages) |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Backend error |
| 503 | Service Unavailable | Backend maintenance or down |

---

## Response Types

### Success Response (2xx)
```json
{
  "id": 123,
  "title": "...",
  "data": {...}
}
```

### Validation Error (422)
```json
{
  "message": "The given data was invalid.",
  "errors": {
    "fieldName": ["Error message 1", "Error message 2"],
    "anotherField": ["Error message"]
  }
}
```

### Authentication Error (401)
```json
{
  "message": "Unauthenticated."
}
```

---

## Testing Endpoints with Postman

### 1. Set up environment variables
```
base_url: http://localhost:8000/api
token: <your_bearer_token>
```

### 2. Create Ticket Request
```
Method: POST
URL: {{base_url}}/tickets
Headers:
  Authorization: Bearer {{token}}
  Content-Type: application/json
  ngrok-skip-browser-warning: true

Body (JSON):
{
  "title": "Test Ticket",
  "description": "Test description",
  "device_id": 1,
  "status": "pending"
}
```

### 3. Upload Image Request
```
Method: POST
URL: {{base_url}}/images/upload
Headers:
  Authorization: Bearer {{token}}
  ngrok-skip-browser-warning: true

Body (form-data):
  image: <select file>
  folder: report_images
```

---

## Rate Limits

Most API endpoints have rate limits:
- **Authenticated requests:** 300 requests per minute
- **Image uploads:** 10 uploads per minute
- **Ticket creation:** 60 tickets per hour

---

## Pagination

For list endpoints:
```typescript
// Get page 2 with 20 items per page
const response = await axiosInstance.get('/tickets', {
  params: {
    page: 2,
    per_page: 20
  }
});

console.log(response.data.meta); // Contains pagination info
```

---

## Field Validation Rules

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| title | string | No | Max 255 chars |
| description | string | Yes | Min 10 chars, Max 5000 chars |
| device_id | integer | Yes | Must exist in devices table |
| status | string | No | One of: pending, in-progress, resolved, open, repair, decommissioned |
| report_image | string | No | Valid storage path |

---

## Common Response Scenarios

### Successful Ticket Creation
```
Status: 201 Created
{
  "id": 456,
  "title": "Issue Report - Device",
  "status": "pending",
  ...
}
```

### Missing Required Field
```
Status: 422 Unprocessable Entity
{
  "message": "The given data was invalid.",
  "errors": {
    "description": ["The description field is required."]
  }
}
```

### Expired Token
```
Status: 401 Unauthorized
{
  "message": "Unauthenticated."
}
```

### Device Not Found
```
Status: 422 Unprocessable Entity
{
  "message": "The given data was invalid.",
  "errors": {
    "device_id": ["The selected device_id is invalid."]
  }
}
```

---

**Last Updated:** May 12, 2026
**API Version:** 1.0
**Backend:** Laravel 11
**Status:** ✅ Production Ready
