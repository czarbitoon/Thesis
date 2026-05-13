Submit Report Form - Debug & Fix Guide
Issues Identified from Screenshot
1. API Response Format Mismatch ⚠️ CRITICAL
Symptom: "Unexpected API response format: {success: true, data: {}}"
Root Cause: The API is returning an empty data object, but the client may be expecting specific fields or the structure doesn't match what the form is configured to parse.
Fix:
javascript// ❌ WRONG - Server sends empty data
{
  "success": true,
  "data": {}
}

// ✅ CORRECT - Server should return meaningful data
{
  "success": true,
  "data": {
    "report_id": "rpt_abc123",
    "created_at": "2025-05-10T14:30:00Z",
    "device": {
      "id": "dev_001",
      "name": "Desktop - Science Lab"
    }
  }
}
In your backend (e.g., smch-api):
python# FastAPI example
@app.post("/api/reports")
async def submit_report(request: ReportRequest):
    # Process report...
    report_id = save_report(request)
    
    return {
        "success": True,
        "data": {
            "report_id": report_id,
            "status": "submitted",
            "created_at": datetime.now().isoformat()
        }
    }

2. Device Enumeration Returns "No devices found" ⚠️ HIGH PRIORITY
Symptom: The Device dropdown shows "No devices found" even when devices are connected.
Root Causes:

WebUSB not available: Browser doesn't support WebUSB API
No USB permission granted: User hasn't granted access to USB devices
Wrong enumeration method: Code is trying to access devices that aren't exposed

Fixes:
Option A: Request User Permission (Recommended)
javascript// Don't just enumerate — ask the user
const handleRequestDevice = async () => {
  try {
    const device = await navigator.usb.requestDevice({
      filters: [
        { classCode: 0xFF }, // Vendor-specific devices
        { vendorId: 0x1234 }  // Replace with your device vendor ID
      ]
    });
    setDevices(prev => [...prev, device]);
  } catch (error) {
    if (error.name !== 'NotFoundError') {
      console.error('Device request failed:', error);
    }
  }
};
Option B: Use SerialPort API (For USB-serial devices)
javascript// Better for devices like Arduino, Teensy, microcontrollers
const handleRequestSerial = async () => {
  try {
    const port = await navigator.serial.requestPort();
    setSerialPort(port);
  } catch (error) {
    console.error('Serial port request failed:', error);
  }
};
Option C: Use HID API (For keyboards, mice, custom HID devices)
javascriptconst handleRequestHID = async () => {
  try {
    const devices = await navigator.hid.requestDevice({
      filters: [{ vendorId: 0x1234 }]
    });
    setHIDDevices(devices);
  } catch (error) {
    console.error('HID request failed:', error);
  }
};

3. Form Validation & Error States Missing
Current Issue: Form can submit with empty fields; no loading state feedback.
Fixes:
javascript// Validate before submit
const handleSubmit = async (e) => {
  e.preventDefault();
  
  // ✅ Validate
  if (!selectedDevice) {
    setError('Please select a device');
    return;
  }
  
  if (!formData.issue?.trim()) {
    setError('Please describe the issue');
    return;
  }
  
  // ✅ Show loading
  setIsSubmitting(true);
  
  try {
    const response = await submitReport();
    setSubmitSuccess(true);
  } catch (error) {
    setError(error.message);
  } finally {
    setIsSubmitting(false);
  }
};

4. Image Upload Issues
Symptom: Image may not be uploaded correctly with the report.
Fix: Use FormData for multipart/form-data:
javascriptconst handleSubmit = async (e) => {
  e.preventDefault();
  
  // ✅ Create FormData for multipart
  const payload = new FormData();
  payload.append('device_id', selectedDevice.vendorId);
  payload.append('description', formData.issue);
  
  // ✅ Append image file (not just name)
  if (formData.image) {
    payload.append('image', formData.image); // File object
  }
  
  // ✅ Send with proper headers
  const response = await axios.post('/api/reports', payload, {
    headers: {
      'Content-Type': 'multipart/form-data'
    }
  });
};

5. Lazy Image Loading Errors
Console shows: "[LazyImage] Loading image 1 after 0ms delay"
Fix: Only use lazy loading for images that are truly off-screen; for form UI, load immediately:
javascript// ❌ WRONG for UI images
<img src={url} loading="lazy" />

// ✅ RIGHT for form elements
<img src={url} loading="eager" />
Or use the image_search tool with proper error handling instead of LazyImage component.

Backend (API) Checklist
Routes to implement/debug:
python# POST /api/reports
# Expected payload (multipart/form-data):
# - device_id: str
# - device_name: str
# - description: str
# - image: File (optional)

# Response:
{
    "success": true,
    "data": {
        "report_id": "rpt_abc123",
        "created_at": "2025-05-10T14:30:00Z"
    }
}

# Error Response:
{
    "success": false,
    "message": "Device not found or invalid"
}

Environment Variables Checklist
.env (Frontend)
VITE_API_BASE_URL=http://localhost:8000
# or for production:
VITE_API_BASE_URL=https://api.yoursite.com
.env (Backend)
DATABASE_URL=postgresql://user:password@localhost:5432/reports
UPLOAD_DIR=/var/uploads
ALLOWED_ORIGINS=http://localhost:5173,https://yoursite.com

Testing Checklist

 Device enumeration works without errors
 Form validates required fields before submit
 Loading state shows during submission
 Success message appears on successful submit
 Error message shows for invalid submissions
 Image uploads correctly with report
 API returns {success: true, data: {...}}
 Form clears after successful submission
 Error message clears when user retries


Quick Debug: Console Logging
Add this to your frontend to trace the flow:
javascriptconst handleSubmit = async (e) => {
  e.preventDefault();
  
  console.log('[1] Form submitted', { 
    device: selectedDevice, 
    issue: formData.issue,
    hasImage: !!formData.image 
  });
  
  const payload = new FormData();
  payload.append('device_id', selectedDevice.vendorId);
  payload.append('description', formData.issue);
  if (formData.image) {
    payload.append('image', formData.image);
  }
  
  try {
    console.log('[2] Sending to API:', import.meta.env.VITE_API_BASE_URL);
    
    const response = await axios.post(
      `${import.meta.env.VITE_API_BASE_URL}/api/reports`,
      payload,
      { headers: { 'Content-Type': 'multipart/form-data' } }
    );
    
    console.log('[3] Success response:', response.data);
    
    if (response.data?.success) {
      console.log('[4] Report submitted:', response.data.data);
      setSubmitSuccess(true);
    } else {
      throw new Error('Server returned success=false');
    }
  } cat (error) {
    console.error('[ERROR]', error.message);
    console.error('Response:', error.response?.data);
    setError(error.message);
  }
};

Next Steps

Verify API endpoint returns correct response structure
Test device enumeration with actual USB devices
Check CORS if API is on different domain
Validate file upload size limits on backend
Add rate limiting to prevent spam submissions
Implement logging for submitted reports