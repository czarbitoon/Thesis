# Supabase Migration Guide for SMCH API

Complete step-by-step guide to migrate from MySQL (Railway) to PostgreSQL (Supabase).

---

## Pre-Migration: Prerequisites

Before starting, ensure you have:
- [x] Current MySQL database credentials (from `.env.example`)
- [x] Supabase account created at https://supabase.com
- [x] New Supabase project created
- [ ] Supabase connection credentials collected
- [ ] MySQL data backed up

---

## Step 1: Backup Your MySQL Data

### Option A: Using PHP (Recommended for Laravel)
```bash
cd c:\Users\Admin\Documents\Thesis\smch-api

# Create backup directory
mkdir backups

# Export current MySQL data
mysqldump -h mysql.railway.internal -u root -p"TetAAOjCSjJSUiOMucyoyqEnAcrtbAUW" railway > backups/mysql_backup_$(date +%Y%m%d_%H%M%S).sql
```

### Option B: Using Laravel
```bash
# Export database using Artisan
php artisan db:seed  # Note current seeded data first
```

---

## Step 2: Update Laravel Configuration

### Create Supabase Connection in `.env`

**Replace** your current `.env` database settings:

```diff
- DB_CONNECTION=mysql
- DB_HOST=mysql.railway.internal
- DB_PORT=3306
- DB_DATABASE=railway
- DB_USERNAME=root
- DB_PASSWORD=TetAAOjCSjJSUiOMucyoyqEnAcrtbAUW

+ DB_CONNECTION=pgsql
+ DB_HOST=your-project.supabase.co
+ DB_PORT=5432
+ DB_DATABASE=postgres
+ DB_USERNAME=postgres
+ DB_PASSWORD=your-supabase-password
```

### Supabase Connection Details to Gather

1. Log in to [Supabase Dashboard](https://app.supabase.com)
2. Select your project
3. Go to **Settings → Database**
4. Copy credentials from **Connection string** section

**Example Supabase .env:**
```env
DB_CONNECTION=pgsql
DB_HOST=abc123def456.supabase.co
DB_PORT=5432
DB_DATABASE=postgres
DB_USERNAME=postgres
DB_PASSWORD=your_secure_password_here
DB_SSLMODE=require
```

---

## Step 3: Migrate Data from MySQL to PostgreSQL

### Option A: Direct Migration Using MySQL & PostgreSQL Tools (Recommended)

**Install pgloader (one-time setup):**
```bash
# Using Chocolatey on Windows
choco install pgloader

# Or download from: https://pgloader.io/
```

**Run migration command:**
```bash
pgloader \
  mysql://root:TetAAOjCSjJSUiOMucyoyqEnAcrtbAUW@mysql.railway.internal/railway \
  postgresql://postgres:your_supabase_password@your-project.supabase.co:5432/postgres
```

### Option B: Using mysqldump + psql (SQL dump method)

**1. Export MySQL schema and data:**
```bash
mysqldump -h mysql.railway.internal -u root -p"TetAAOjCSjJSUiOMucyoyqEnAcrtbAUW" \
  --compatible=postgresql railway > backup.sql
```

**2. Convert MySQL dump to PostgreSQL format:**
```bash
# Using sed for basic conversions
sed -i 's/^\/\*.*\*\/;$//' backup.sql
sed -i 's/ KEY / INDEX /' backup.sql
```

**3. Import into Supabase:**
```bash
psql -h your-project.supabase.co -U postgres -d postgres < backup.sql
```

**When prompted, enter your Supabase password**

### Option C: Using Laravel Migration (Programmatic)

Create a custom command:
```bash
php artisan make:command MigrateToSupabase
```

---

## Step 4: Verify Data Transfer

### Check data in Supabase (via SQL Editor):

```sql
-- In Supabase SQL Editor:
SELECT table_name FROM information_schema.tables WHERE table_schema = 'public';

-- Count records in each table
SELECT COUNT(*) as total_users FROM users;
SELECT COUNT(*) as total_reports FROM reports;
SELECT COUNT(*) as total_devices FROM devices;
```

### Check Laravel connection:

```bash
php artisan tinker
>>> DB::connection('pgsql')->select('SELECT 1')
```

---

## Step 5: Update Application Configuration

### 1. Update `config/database.php` if needed:

PostgreSQL settings are already configured, but verify:

```php
'default' => env('DB_CONNECTION', 'pgsql'),  // Change from 'sqlite'
```

### 2. Install PostgreSQL PHP Extension (if not installed):

```bash
composer require illuminate/database

# Windows: Enable in php.ini
# Uncomment: extension=pdo_pgsql
```

### 3. Update Docker configuration (if using Docker):

**Update `docker-compose.yml`:**

```yaml
services:
  # Remove old MySQL service or comment out
  # mysql:
  #   ...

  # Add Supabase reference (Supabase is managed cloud service)
  app:
    environment:
      DB_CONNECTION: pgsql
      DB_HOST: ${SUPABASE_HOST}
      DB_PORT: 5432
      DB_DATABASE: postgres
      DB_USERNAME: postgres
      DB_PASSWORD: ${SUPABASE_PASSWORD}
```

---

## Step 6: Run Database Migrations

### Reset and re-migrate:

```bash
# Option 1: Fresh migration (recommended for first-time)
php artisan migrate:fresh

# Option 2: If you already have data and just need schema
php artisan migrate

# Option 3: Rollback and re-migrate
php artisan migrate:rollback
php artisan migrate
```

### Seed data if needed:

```bash
php artisan db:seed
```

---

## Step 7: Update Environment-Specific Configurations

### Update `.env.railway` (if using Railway for other services):

```env
DB_CONNECTION=pgsql
DB_HOST=your-project.supabase.co
DB_PORT=5432
DB_DATABASE=postgres
DB_USERNAME=postgres
DB_PASSWORD=your_password
```

### Update `.env.docker`:

```env
DB_CONNECTION=pgsql
DB_HOST=your-project.supabase.co
DB_PORT=5432
DB_DATABASE=postgres
DB_USERNAME=postgres
DB_PASSWORD=your_password
```

---

## Step 8: Test Application

```bash
# Run Laravel tests
php artisan test

# Test database connection
php artisan db:show

# Run tinker to verify data
php artisan tinker
>>> User::count()
>>> Report::count()
>>> Device::count()
```

---

## PostgreSQL-Specific Notes

### MySQL to PostgreSQL Differences to Watch:

| Feature | MySQL | PostgreSQL |
|---------|-------|-----------|
| **AUTO_INCREMENT** | `AUTO_INCREMENT` | `SERIAL` or `BIGSERIAL` |
| **String types** | `VARCHAR(255)` | `VARCHAR(255)` (same) |
| **Boolean** | `TINYINT(1)` | `BOOLEAN` |
| **Datetime** | `TIMESTAMP` | `TIMESTAMP` |
| **JSON** | `JSON` | `JSONB` (better) |
| **Schemas** | Single database | Multiple schemas |

### Supabase PostgreSQL Configuration:

- **Version**: PostgreSQL 14+
- **SSL Mode**: Required (sslmode=require)
- **Connection Limit**: 100 connections (check plan)
- **Backup**: Automatic daily backups included

---

## Troubleshooting

### Connection refused error:

```bash
# Test connection
psql -h your-project.supabase.co -U postgres -d postgres

# If fails, verify:
# 1. Host is correct
# 2. Password is correct  
# 3. IP whitelist in Supabase (allow all if testing)
```

### Data not transferred:

```bash
# Check if data exists in MySQL
mysql -h mysql.railway.internal -u root -p"password" railway -e "SELECT COUNT(*) FROM users;"

# Verify PostgreSQL received data
psql -h your-project.supabase.co -U postgres -d postgres -c "SELECT COUNT(*) FROM users;"
```

### Laravel migration errors:

```bash
# Check for schema differences
php artisan tinker
>>> Schema::hasTable('users')
>>> Schema::getColumns('users')

# Compare with old database
mysql -h mysql.railway.internal -u root -p"password" railway -e "DESCRIBE users;"
```

---

## Rollback Plan

If issues occur, revert to MySQL:

```bash
# Update .env back to MySQL
DB_CONNECTION=mysql
DB_HOST=mysql.railway.internal
DB_PORT=3306
DB_DATABASE=railway
DB_USERNAME=root
DB_PASSWORD=TetAAOjCSjJSUiOMucyoyqEnAcrtbAUW

# Clear Laravel cache
php artisan config:clear
php artisan cache:clear

# Test connection
php artisan tinker
>>> DB::connection('mysql')->select('SELECT 1')
```

---

## Next Steps

1. [ ] Create Supabase PostgreSQL project
2. [ ] Collect Supabase credentials
3. [ ] Backup MySQL data
4. [ ] Update `.env` with Supabase credentials
5. [ ] Run data migration
6. [ ] Verify data transfer
7. [ ] Run Laravel tests
8. [ ] Update Docker/deployment configs
9. [ ] Deploy to production
10. [ ] Monitor logs for issues

---

**Questions?** Check Supabase docs: https://supabase.com/docs/guides/database
