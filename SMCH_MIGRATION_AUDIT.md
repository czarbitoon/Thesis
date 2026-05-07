# SMCH System Audit & Migration Plan
## Ready for SMC Demo Tomorrow

---

## 📊 CURRENT STATE ASSESSMENT

### Tech Stack (CORRECTED)
- **API**: Laravel 11 PHP (not Node.js) with Eloquent ORM
- **Database**: Supabase PostgreSQL (already configured in `.env.docker`)
- **Web**: React + Vite frontend bundler  
- **Mobile**: React Native app (to-be-configured)
- **Auth**: Laravel Sanctum (token-based)

### What's Already Good ✅
1. ✅ Supabase credentials exist in `.env.docker` (SUPABASE_URL + SUPABASE_ANON_KEY)
2. ✅ API routes well-structured with 14 controllers
3. ✅ Database migrations ready (15 migrations total)
4. ✅ Authentication middleware in place (auth:sanctum)
5. ✅ Device/Report/Profile controllers exist

### What Needs Fixing 🔧
1. ❌ `.env.docker` still points to MySQL (`DB_CONNECTION=mysql` + `DB_HOST=db`)
2. ❌ Laravel using Eloquent ORM (will work with PostgreSQL, but needs verification)  
3. ❌ No Row Level Security (RLS) configured in Supabase
4. ❌ Mobile app has no configuration for API IP/base URL
5. ❌ JSONB specs for devices table not implemented in migrations
6. ❌ Device image storage needs clarification (local vs. cloud)

---

## 📁 CODEBASE INDEX

### smch-api/ (Laravel Backend)
```
Routes: routes/api.php
├── GET/POST /devices          → DeviceController
├── POST /reports              → ReportController  
├── GET /profile               → AuthController.profile()
├── POST /profile/update       → ProfileController
└── 14 more routes...

Controllers: app/Http/Controllers/
├── DeviceController.php       - READY (Eloquent queries)
├── ReportController.php       - READY (Eloquent + events)
├── ProfileController.php      - READY (user updates)
├── AuthController.php         - READY (Sanctum auth)
└── 10 other controllers

Models: app/Models/
├── Device.php                 - Relationships: office, category, type
├── Report.php                 - Relationships: device, user, office
├── User.php                   - Relationships: office, reports
└── Office.php, etc.

Migrations: database/migrations/ (15 total)
├── users, offices             - ✅ DONE
├── devices, device_types      - ✅ DONE (no JSONB)
├── reports                    - ✅ DONE
└── notifications              - ✅ DONE

Configuration:
├── .env.docker               - ❌ Still MySQL (needs PostgreSQL)
├── .env.supabase.example     - ✅ Reference exists
├── config/database.php       - ✅ Has PostgreSQL driver
```

### smch-web/ (React Frontend)
```
src/
├── axiosInstance.js          - API client (baseURL from env var)
├── App.jsx                   - Main component
├── components/               - React components
├── services/                 - API service calls
├── context/                  - State management
└── hooks/                    - Custom React hooks

Configuration:
├── package.json              - Dependencies + build script
└── .env (missing - needs VITE_API_BASE_URL)
```

### smch-mobile-app/ (React Native)
```
Currently: NOT CONFIGURED for API calls
├── App.tsx                   - Main entry
├── app/                      - App screens
├── components/               - React Native components
├── constants/                - Constants (API_URL missing)
├── hooks/                    - Custom hooks
└── utils/                    - Utilities
```

---

## 🔧 PRIORITY FIX #1: Database Connection (.env.docker)

### Current (❌ WRONG):
```env
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=smch
DB_USERNAME=smch
DB_PASSWORD=secret
```

### Required (✅ CORRECT):
```env
DB_CONNECTION=pgsql
DB_HOST=aetmsaviiieysmekjfct.supabase.co
DB_PORT=5432
DB_DATABASE=postgres
DB_USERNAME=postgres
DB_PASSWORD=your-supabase-password
```

### Action:
Replace the DB section in `.env.docker` with Supabase PostgreSQL credentials.

---

## 🗄️ DATABASE SCHEMA REVIEW

### Devices Table (CURRENT)
```sql
CREATE TABLE devices (
  id BIGSERIAL PRIMARY KEY,
  name VARCHAR(255),
  description TEXT,
  device_category_id BIGINT (FK),
  device_type_id BIGINT (FK),
  office_id BIGINT (FK),
  serial_number VARCHAR(255),
  model_number VARCHAR(255),
  manufacturer VARCHAR(255),
  status VARCHAR(255),
  image VARCHAR(255) NULLABLE,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);
```

### ⚠️ ENHANCEMENT NEEDED: Add JSONB for Specs
Suggested migration (if needed):
```php
Schema::table('devices', function (Blueprint $table) {
    $table->jsonb('specifications')->nullable()->comment('Hardware specs as JSONB');
});
```

### Reports Table (GOOD)
```sql
CREATE TABLE reports (
  id BIGSERIAL PRIMARY KEY,
  title VARCHAR(255),
  description TEXT,
  device_id BIGINT (FK),
  user_id BIGINT (FK),
  office_id BIGINT (FK),
  status ENUM (pending, in_progress, completed, closed),
  resolution_notes TEXT NULLABLE,
  resolved_at TIMESTAMP NULLABLE,
  resolved_by BIGINT (FK - users) NULLABLE,
  device_image_url VARCHAR(255) NULLABLE,
  report_image VARCHAR(255) NULLABLE,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);
```

### Users Table (GOOD)
```sql
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  name VARCHAR(255),
  email VARCHAR(255) UNIQUE,
  profile_picture VARCHAR(255) NULLABLE,
  email_verified_at TIMESTAMP NULLABLE,
  password VARCHAR(255),
  user_role VARCHAR(255) DEFAULT 'user',
  api_token VARCHAR(255) UNIQUE NULLABLE,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);
```

---

## 🔐 SUPABASE ROW LEVEL SECURITY (RLS) - REQUIRED

### Enable RLS in Supabase:
1. Go to **SQL Editor** in Supabase dashboard
2. Run these policies:

#### Users Table - User can only see own profile
```sql
CREATE POLICY "Users can see own profile"
ON users FOR SELECT
USING (auth.uid()::text = id::text);

CREATE POLICY "Users can update own profile"
ON users FOR UPDATE
USING (auth.uid()::text = id::text);
```

#### Devices Table - Office staff can see devices in their office
```sql
CREATE POLICY "Users can see devices in their office"
ON devices FOR SELECT
USING (office_id = (SELECT office_id FROM users WHERE id = auth.uid()::bigint));
```

#### Reports Table - Users can see reports for their devices
```sql
CREATE POLICY "Users can see reports for their devices"
ON reports FOR SELECT
USING (user_id = auth.uid()::bigint);

CREATE POLICY "Users can create reports"
ON reports FOR INSERT
WITH CHECK (user_id = auth.uid()::bigint);
```

---

## 🚀 API CONTROLLER FIXES NEEDED

### Issue #1: DeviceController - Image URL Handling
**File**: `app/Http/Controllers/DeviceController.php`

**Current Code** (Line ~95):
```php
return response()->json([
    'success' => true,
    'data' => $devices->through(function ($device) {
        return array_merge($device->toArray(), [
            'image_url' => $device->image_url  // ❌ Method doesn't exist
        ]);
    })
]);
```

**Fix**: Update to use actual image model attribute
```php
return response()->json([
    'success' => true,
    'data' => $devices->through(function ($device) {
        return array_merge($device->toArray(), [
            'image_url' => $device->image ? asset('storage/' . $device->image) : null
        ]);
    })
]);
```

### Issue #2: ReportController - Typo in Status Variable
**File**: `app/Http/Controllers/ReportController.php`

**Current Code** (Line ~59):
```php
'status' => strtolower($status),  // ❌ $status not defined
```

**Fix**: Use validated data
```php
'status' => $validatedData['status'] ?? 'pending',
```

### Issue #3: ProfileController - Missing office_id Update
**File**: `app/Http/Controllers/ProfileController.php`

**Current Code**: Doesn't handle office updates

**Add Method**:
```php
public function updateOffice(Request $request)
{
    $request->validate([
        'office_id' => 'required|exists:offices,id',
    ]);
    
    $user = Auth::user();
    if (!$user) {
        return response()->json(['message' => 'User not authenticated'], 401);
    }
    
    $user->update(['office_id' => $request->office_id]);
    
    return response()->json([
        'message' => 'Office updated successfully',
        'office_id' => $user->office_id,
    ]);
}
```

---

## 📱 MOBILE APP CONFIG - REQUIRED FOR TOMORROW

### File: `smch-mobile-app/constants/index.ts` (or similar)

**Add**:
```typescript
// API Configuration
export const API_CONFIG = {
  // Use local IP for physical device testing
  // Development: Replace with your machine's local IP
  BASE_URL: process.env.API_URL || 'http://192.168.x.x:8000', // ⚠️ UPDATE THIS
  TIMEOUT: 30000,
  HEADERS: {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  }
};

// Device & Report Types
export const DEVICE_STATUS = {
  ACTIVE: 'active',
  INACTIVE: 'inactive',
  MAINTENANCE: 'maintenance',
};

export const REPORT_STATUS = {
  PENDING: 'pending',
  IN_PROGRESS: 'in_progress',
  COMPLETED: 'completed',
  CLOSED: 'closed',
};
```

### File: `smch-mobile-app/utils/api.ts` (or create new)

```typescript
import { API_CONFIG } from '../constants';

export const apiClient = {
  async getDevices(token: string) {
    const response = await fetch(`${API_CONFIG.BASE_URL}/api/devices`, {
      headers: {
        'Authorization': `Bearer ${token}`,
        ...API_CONFIG.HEADERS,
      },
    });
    return response.json();
  },

  async postTicket(token: string, ticketData: any) {
    const response = await fetch(`${API_CONFIG.BASE_URL}/api/reports`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        ...API_CONFIG.HEADERS,
      },
      body: JSON.stringify(ticketData),
    });
    return response.json();
  },

  async getProfile(token: string) {
    const response = await fetch(`${API_CONFIG.BASE_URL}/api/profile`, {
      headers: {
        'Authorization': `Bearer ${token}`,
        ...API_CONFIG.HEADERS,
      },
    });
    return response.json();
  },
};
```

---

## 🌐 WEB APP ENV VARS

### File: `smch-web/.env` (create if missing)

```env
# API Configuration
VITE_API_BASE_URL=http://localhost:8000

# Feature Flags (optional)
VITE_DEBUG_MODE=false
VITE_LOG_REQUESTS=true
```

### File: `smch-web/.env.production`

```env
VITE_API_BASE_URL=https://your-api-domain.com
VITE_DEBUG_MODE=false
```

---

## ✅ IMPLEMENTATION CHECKLIST - TOMORROW

### Morning (Before Demo):

- [ ] **Step 1**: Update `.env.docker` with PostgreSQL details (3 min)
  
- [ ] **Step 2**: Fix DeviceController image URL (2 min)
  
- [ ] **Step 3**: Fix ReportController status variable (2 min)
  
- [ ] **Step 4**: Add ProfileController.updateOffice() (5 min)
  
- [ ] **Step 5**: Configure RLS policies in Supabase (5 min)
  
- [ ] **Step 6**: Run Laravel migrations on Supabase (2 min)
  ```bash
  php artisan config:clear
  php artisan migrate --force
  ```

- [ ] **Step 7**: Test API endpoints (5 min)
  ```bash
  php artisan tinker
  >>> User::count()
  >>> Device::count()
  >>> Report::count()
  ```

- [ ] **Step 8**: Set up mobile app constants (3 min)
  
- [ ] **Step 9**: Create `.env` file for web (1 min)
  
- [ ] **Step 10**: Test full stack connectivity (5 min)

**Total Time**: ~30 minutes

---

## 🧪 DEMO TEST SCENARIOS

### Test 1: Devices Endpoint
```bash
curl -X GET http://localhost:8000/api/devices \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json"
```

**Expected Response**:
```json
{
  "success": true,
  "data": {
    "data": [
      {
        "id": 1,
        "name": "Device Name",
        "status": "active",
        "office_id": 1,
        "image_url": "https://...",
        "created_at": "2025-..."
      }
    ]
  }
}
```

### Test 2: Create Report
```bash
curl -X POST http://localhost:8000/api/reports \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Device Not Working",
    "description": "Device XYZ has stopped responding",
    "device_id": 1,
    "status": "pending"
  }'
```

### Test 3: Get Profile
```bash
curl -X GET http://localhost:8000/api/profile \
  -H "Authorization: Bearer YOUR_TOKEN"
```

---

## 🐳 DOCKER CLEANUP

**Remove MySQL dependency** from docker-compose.yml:

```yaml
# docker-compose.yml

services:
  api:
    # ... config ...
    # Remove: depends_on: - db
    
  # REMOVE or COMMENT OUT:
  # db:
  #   image: mysql:8.0
  #   ...
  # phpmyadmin:
  #   ...

volumes:
  # REMOVE: db_data:
```

---

## 📋 MIGRATION SUCCESS CRITERIA

✅ ALL must be true for demo:

1. ✅ DeviceController returns devices from Supabase PostgreSQL
2. ✅ ReportController creates reports with correct device_id link
3. ✅ ProfileController returns user data
4. ✅ RLS policies prevent unauthorized access
5. ✅ Mobile app can connect via local IP (192.168.x.x)
6. ✅ Web frontend calls API successfully
7. ✅ Authentication works with Sanctum tokens
8. ✅ Docker runs without MySQL service

---

## 🎯 NEXT IMMEDIATE ACTIONS

1. **Confirm Supabase credentials** (host, password)
2. **Get your local machine IP** for mobile config:
   ```bash
   ipconfig  # On Windows: look for IPv4 address (192.168.x.x)
   ```
3. **Run migrations**:
   ```bash
   cd c:\Users\Admin\Documents\Thesis\smch-api
   php artisan config:clear
   php artisan migrate
   ```
4. **Apply the 3 controller fixes** (image URL, status, office_id)
5. **Test endpoints** before demo

---

**Status**: Ready to execute. Estimated time: 30-45 minutes.
**Demo readiness**: HIGH - All components identified and fixes clear.
