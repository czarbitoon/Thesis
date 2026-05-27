# CORS and Vercel Deployment Guide

## Overview

This guide ensures CORS is properly configured across all three applications (Web, Mobile, API) and that Vercel deployments work seamlessly without CORS issues.

## Architecture

```
┌─────────────────────┐         ┌─────────────────────┐         ┌─────────────────────┐
│   Web App           │         │   Mobile App        │         │   API Server        │
│  (smch-web)         │         │ (smch-mobile-app)   │         │   (smch-api)        │
│                     │         │                     │         │                     │
│  Vercel URL:        │         │  Expo/Mobile:       │         │  Vercel URL:        │
│  smch-web.vercel... │         │  EXPO_PUBLIC_URL    │         │  smch-api.vercel... │
└──────────┬──────────┘         └──────────┬──────────┘         └──────────┬──────────┘
           │                               │                              │
           └───────────────────────────────┼──────────────────────────────┘
                      All requests to /api via CORS
```

## CORS Configuration Overview

### 1. API CORS Configuration (smch-api)

The API uses **two layers** of CORS configuration:

#### Layer 1: Config File (`config/cors.php`)
- Defines allowed paths, methods, headers
- Uses regex patterns for dynamic domain matching
- Allows Vercel domains: `*.vercel.app`
- Allows ngrok domains: `*.ngrok.dev`
- Allows localhost with any port

#### Layer 2: Middleware (Kernel.php)
- Laravel's built-in `HandleCors` middleware processes requests
- Uses the config/cors.php file to validate origins

### 2. Web App Configuration (smch-web)

**Before (INCORRECT):**
```json
{
  "headers": [
    {
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Credentials": "true"
      // ❌ Invalid: Can't use "*" with credentials=true
    }
  ]
}
```

**After (CORRECT):**
- Removed problematic CORS headers from vercel.json
- Let the backend handle CORS (via Laravel)
- Backend will dynamically set `Access-Control-Allow-Origin` to the requesting domain

### 3. Mobile App Configuration (smch-mobile-app)

- Uses `EXPO_PUBLIC_API_URL` environment variable
- Development: Points to local machine IP (192.168.16.1:8000)
- Production: Should point to Vercel API (smch-api.vercel.app/api)

---

## Vercel Environment Variables

### API Service (smch-api)

Set these in Vercel Dashboard under project settings → Environment Variables:

```
APP_ENV=production
APP_DEBUG=false
APP_NAME=SMCH
APP_KEY=<your-base64-encoded-key>
APP_URL=https://smch-api.vercel.app

# CORS Configuration
ALLOWED_ORIGINS=https://smch-web.vercel.app,http://localhost:5173

# Sanctum Stateful Domains (for session-based auth)
SANCTUM_STATEFUL_DOMAINS=smch-web.vercel.app,localhost:5173,smch-api.vercel.app

# Database (set from Vercel PostgreSQL or external service)
DB_CONNECTION=pgsql
DB_HOST=<db-host>
DB_PORT=5432
DB_DATABASE=<db-name>
DB_USERNAME=<db-user>
DB_PASSWORD=<db-password>

# Session
SESSION_DRIVER=cookie
SESSION_LIFETIME=120
SESSION_SECURE_COOKIE=true

# Other services
MAIL_MAILER=log
REDIS_HOST=<redis-host>
```

### Web Service (smch-web)

Set in Vercel Dashboard:

```
VITE_API_BASE_URL=https://smch-api.vercel.app/api
VITE_APP_ENV=production
```

### Mobile App (smch-mobile-app)

Can be set in `.env` or via EAS build:

```
EXPO_PUBLIC_API_URL=https://smch-api.vercel.app/api
```

---

## How CORS Works on Redeployment

### Scenario: Vercel redeploys API

1. **New Vercel URL generated** (if needed)
2. **API CORS pattern matches**: `#^https://.*\.vercel\.app$#`
3. **No code changes needed**: Pattern already includes all Vercel domains
4. **Existing web/mobile apps continue working**: They can reach the new URL

### Scenario: Vercel redeploys Web app

1. **Web app config still points to** `VITE_API_BASE_URL=https://smch-api.vercel.app/api`
2. **Browser sends preflight OPTIONS request** to API
3. **API responds with CORS headers**: 
   ```
   Access-Control-Allow-Origin: https://smch-web.vercel.app
   Access-Control-Allow-Credentials: true
   ```
4. **Request succeeds**: Browser allows the actual request

---

## Files Modified

### ✅ Fixed Files

1. **smch-api/vercel.json**
   - Was: Empty
   - Now: Proper Laravel configuration for Vercel

2. **smch-web/vercel.json**
   - Was: Incorrect CORS headers with `*` origin
   - Now: Removed problematic headers, let backend handle CORS

3. **smch-web/.env.production**
   - Was: Pointing to Railway API
   - Now: Pointing to Vercel API (smch-api.vercel.app)

4. **smch-api/.env**
   - Updated SANCTUM_STATEFUL_DOMAINS

5. **smch-mobile-app/.env**
   - Added comment for production setup

### ✅ Already Correct

- **smch-api/config/cors.php**: Already has proper regex patterns
- **smch-api/app/Http/Kernel.php**: Already has HandleCors middleware
- **smch-web/src/axiosInstance.js**: Already uses VITE_API_BASE_URL
- **smch-mobile-app/utils/api.ts**: Already uses EXPO_PUBLIC_API_URL

---

## Testing CORS Before Deployment

### 1. Local Testing

```bash
# Terminal 1: Start API
cd smch-api
php artisan serve

# Terminal 2: Start Web App
cd smch-web
npm run dev

# Test in browser console:
# fetch('http://localhost:8000/api/health')
# Should see CORS headers in response
```

### 2. Check CORS Headers

```bash
# Test with curl
curl -H "Origin: http://localhost:5173" \
     -H "Access-Control-Request-Method: GET" \
     -X OPTIONS http://localhost:8000/api/health -v
```

You should see:
```
Access-Control-Allow-Origin: http://localhost:5173
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, PATCH, OPTIONS
Access-Control-Allow-Credentials: true
```

### 3. Pre-Deployment Verification

Before deploying to Vercel:

1. ✅ Test API locally: `php artisan serve`
2. ✅ Test Web app locally: `npm run dev`
3. ✅ Verify VITE_API_BASE_URL in .env.production
4. ✅ Verify Vercel environment variables are set
5. ✅ Test with mobile app (if applicable)

---

## Troubleshooting

### Issue: "Access-Control-Allow-Origin" header missing

**Cause**: Origin not in ALLOWED_ORIGINS or doesn't match patterns

**Fix**:
```php
// In smch-api/.env, add the domain:
ALLOWED_ORIGINS=https://smch-web.vercel.app,https://your-new-domain.vercel.app

// OR update regex in config/cors.php if needed
'allowed_origins_patterns' => [
    '#^https://.*\.vercel\.app$#',  // This should catch all Vercel domains
    '#^https://.*\.ngrok(?:-free)?\.dev$#',
    '#^https?://localhost.*$#',
]
```

### Issue: Credentials not sent with requests

**Cause**: `supports_credentials` not set or requests don't include credentials

**Fix**:
```php
// In smch-api/config/cors.php
'supports_credentials' => true,  // Should be true

// In browser requests
fetch(url, {
    credentials: 'include',  // Important for cookies
    headers: { 'Authorization': 'Bearer ' + token }
})
```

### Issue: Preflight requests fail

**Cause**: OPTIONS requests not allowed

**Fix**: API should respond to OPTIONS requests automatically via Laravel middleware

```bash
# Test with:
curl -X OPTIONS https://smch-api.vercel.app/api/health -v
# Should return 200 OK with CORS headers
```

### Issue: Mobile app can't reach API

**Cause**: EXPO_PUBLIC_API_URL not set or incorrect

**Fix**:
```bash
# In smch-mobile-app/.env
EXPO_PUBLIC_API_URL=https://smch-api.vercel.app/api

# Verify in app:
console.log(process.env.EXPO_PUBLIC_API_URL)
```

---

## Deployment Checklist

- [ ] API vercel.json is configured
- [ ] Web app vercel.json CORS headers removed
- [ ] VITE_API_BASE_URL set in web app .env.production
- [ ] API ALLOWED_ORIGINS includes web app domain
- [ ] SANCTUM_STATEFUL_DOMAINS configured
- [ ] Vercel environment variables set for all services
- [ ] Local testing passes
- [ ] CORS headers visible in network tab
- [ ] Mobile app can reach API (if applicable)
- [ ] Preview deployments work (domain patterns should match)

---

## Key Takeaway

**CORS is automatically handled via:**
1. Backend API regex patterns for Vercel domains (`*.vercel.app`)
2. Backend API ALLOWED_ORIGINS for specific domains
3. Frontend environment variables pointing to correct API URL
4. Browser sends CORS headers, backend validates and responds

**No changes needed on redeployment** as long as:
- URLs stay the same or follow the pattern
- Environment variables are set in Vercel dashboard
- CORS config uses regex patterns (already done)
