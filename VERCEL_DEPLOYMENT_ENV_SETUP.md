# Vercel Deployment Checklist & Environment Variables

## Pre-Deployment Quick Checklist

### API Service (smch-api)
- [ ] vercel.json is created and configured
- [ ] .env files are NOT committed to git
- [ ] Database migrations are up to date
- [ ] Storage directory is properly configured

### Web Service (smch-web)
- [ ] vercel.json has correct configuration (no CORS headers)
- [ ] .env.production points to correct API: `VITE_API_BASE_URL=https://smch-api.vercel.app/api`
- [ ] Build command works: `npm run build`
- [ ] dist/ folder is generated

### Mobile App (smch-mobile-app)
- [ ] .env has correct API URL or uses EXPO_PUBLIC_API_URL

---

## Vercel Dashboard Environment Variables

### For API Project (smch-api)

**Go to:** Vercel Dashboard → Select smch-api project → Settings → Environment Variables

Copy and paste these, replacing `<value>` with actual values:

```
APP_ENV=production
APP_DEBUG=false
APP_NAME=SMCH
APP_KEY=<get from local .env, format: base64:xxxxx>
APP_URL=https://smch-api.vercel.app

# CORS & Sanctum
ALLOWED_ORIGINS=https://smch-web.vercel.app,http://localhost:5173
SANCTUM_STATEFUL_DOMAINS=smch-web.vercel.app,localhost:5173,smch-api.vercel.app
SESSION_DRIVER=cookie
SESSION_LIFETIME=120
SESSION_SECURE_COOKIE=true

# Database Connection (change if not using PostgreSQL)
DB_CONNECTION=pgsql
DB_HOST=<database-host>
DB_PORT=5432
DB_DATABASE=<database-name>
DB_USERNAME=<database-username>
DB_PASSWORD=<database-password>

# Mail Configuration
MAIL_MAILER=log
MAIL_FROM_ADDRESS=noreply@example.com
MAIL_FROM_NAME="SMCH"

# Optional: Redis for caching/queues
REDIS_HOST=<redis-host>
REDIS_PASSWORD=<redis-password>
REDIS_PORT=6379

# Optional: JWT Token Secret
JWT_SECRET=<your-jwt-secret>

# Logging
LOG_CHANNEL=stack
LOG_LEVEL=info
```

**⚠️ IMPORTANT for each environment:**
- **Production**: Set `APP_ENV=production`, `APP_DEBUG=false`
- **Preview/Staging**: Can use `APP_DEBUG=true` temporarily for debugging

### For Web Project (smch-web)

**Go to:** Vercel Dashboard → Select smch-web project → Settings → Environment Variables

```
VITE_API_BASE_URL=https://smch-api.vercel.app/api
VITE_APP_ENV=production
```

### For Mobile App (smch-mobile-app)

For EAS build environment, add to `.env` or EAS secrets:

```
EXPO_PUBLIC_API_URL=https://smch-api.vercel.app/api
EXPO_PUBLIC_DEBUG=false
```

---

## Step-by-Step Deployment

### 1. Deploy API First

```bash
# In smch-api directory
git add .
git commit -m "Fix CORS and Vercel configuration"
git push origin master
```

- Go to Vercel Dashboard
- Select smch-api project
- Deployment should auto-trigger
- Set all environment variables listed above
- Monitor deployment logs for errors
- Test health endpoint: `https://smch-api.vercel.app/api/health`

### 2. Deploy Web App

```bash
# In smch-web directory
git add .
git commit -m "Update API URL and fix CORS"
git push origin main
```

- Vercel auto-deploys
- Environment variables should be set (VITE_API_BASE_URL)
- Test that web app loads and can make API requests

### 3. Update Mobile App (if applicable)

- Update .env with new API URL
- Rebuild with `eas build` or local build process

---

## Post-Deployment Testing

### 1. Check API Health

```bash
curl https://smch-api.vercel.app/api/health
```

Expected response:
```json
{
  "status": "ok",
  "database": "connected",
  "timestamp": "2024-05-24T...",
  "env": "production",
  "debug": false
}
```

### 2. Test CORS from Web App

```bash
# In browser console on web app:
fetch('https://smch-api.vercel.app/api/health')
  .then(r => r.json())
  .then(console.log)
```

Should not get CORS errors.

### 3. Check CORS Headers

```bash
curl -H "Origin: https://smch-web.vercel.app" \
     -H "Access-Control-Request-Method: GET" \
     -X OPTIONS https://smch-api.vercel.app/api/health -v
```

Should see:
```
Access-Control-Allow-Origin: https://smch-web.vercel.app
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, PATCH, OPTIONS
Access-Control-Allow-Credentials: true
```

### 4. Test Web App Functionality

- Log in to web app
- Create/read data
- Upload files (test storage)
- Check browser Network tab for CORS headers

### 5. Test Mobile App (if applicable)

- Run on device or emulator
- Verify API calls work
- Check for CORS errors in console

---

## Troubleshooting Failed Deployment

### Vercel Build Fails

**Check Vercel logs:**
```bash
# View real-time logs
vercel logs <project-name> --tail

# Or in Vercel Dashboard → Deployments → Failed deployment
```

**Common issues:**
- Missing environment variables
- Database connection error
- Build command failed (missing npm packages)

**Fix:**
- Add missing env vars
- Run `npm install` locally and commit lockfile
- Check `composer.json` for PHP dependencies

### CORS Still Broken After Deployment

1. ✅ Verify ALLOWED_ORIGINS in API env vars
2. ✅ Verify VITE_API_BASE_URL in Web env vars
3. ✅ Check that web app is calling `https://smch-api.vercel.app/api`
4. ✅ Clear browser cache (Ctrl+Shift+Del)
5. ✅ Check browser Console for actual error message

### Database Connection Errors

- ✅ Verify DB_HOST, DB_PORT, DB_USERNAME, DB_PASSWORD
- ✅ Ensure database is accessible from Vercel servers
- ✅ Check database firewall rules (whitelist Vercel IPs if needed)
- ✅ Run migrations: Deploy script should handle this

### Mobile App Can't Reach API

- ✅ Verify EXPO_PUBLIC_API_URL is set
- ✅ Check that it includes `/api` suffix
- ✅ Verify app is rebuilt after env change
- ✅ Check mobile device network (might be behind firewall)

---

## Redeployment Scenarios

### Scenario 1: Just update code

```bash
git push origin main/master
# Vercel auto-deploys, env vars already set from before
```

### Scenario 2: Add/change environment variable

1. Update in Vercel Dashboard → Settings → Environment Variables
2. Trigger rebuild: Dashboard → Redeploy
3. Test health endpoint

### Scenario 3: Database migrations needed

Add to `package.json` or post-deploy script:

```json
{
  "scripts": {
    "build": "npm install && npm run build",
    "postinstall": "php artisan migrate --force"
  }
}
```

Or run manually after deploy:
```bash
vercel env pull  # Get env vars locally
php artisan migrate --database=production
```

### Scenario 4: Multiple preview deployments

Vercel creates preview URLs like: `https://smch-api-pr-123.vercel.app`

**No action needed!** The CORS config regex pattern matches all Vercel domains:
```php
'#^https://.*\.vercel\.app$#'
```

So preview deployments automatically have CORS access.

---

## Important Files Reference

| File | Purpose | Location |
|------|---------|----------|
| vercel.json (API) | API deployment config | smch-api/vercel.json |
| vercel.json (Web) | Web deployment config | smch-web/vercel.json |
| .env.production | Production env vars | smch-web/.env.production |
| config/cors.php | CORS configuration | smch-api/config/cors.php |
| Kernel.php | Middleware configuration | smch-api/app/Http/Kernel.php |
| CORS_AND_VERCEL_DEPLOYMENT.md | Detailed guide | Root directory |

---

## Quick Reference: URLs

| Service | Environment | URL |
|---------|-------------|-----|
| API | Development | http://localhost:8000 |
| API | Production | https://smch-api.vercel.app |
| Web App | Development | http://localhost:5173 |
| Web App | Production | https://smch-web.vercel.app |
| Mobile App | Development | Device IP (192.168.x.x) |
| Mobile App | Production | Device (uses EXPO_PUBLIC_API_URL) |

---

## Support

If CORS issues persist:

1. Check CORS_AND_VERCEL_DEPLOYMENT.md for detailed explanation
2. Review Vercel deployment logs
3. Check browser Network tab for actual CORS error details
4. Verify all environment variables are set in Vercel Dashboard
5. Clear browser cache and rebuild mobile app if needed
