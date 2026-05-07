# Quick Migration Commands - Copy & Paste

## 1. BACKUP CURRENT MySQL DATA
```bash
cd c:\Users\Admin\Documents\Thesis\smch-api

# Create backup directory
mkdir backups

# Export MySQL database
mysqldump -h mysql.railway.internal -u root -p"TetAAOjCSjJSUiOMucyoyqEnAcrtbAUW" railway > backups/mysql_backup.sql
```

---

## 2. SWITCH TO SUPABASE CONNECTION

**Update your `.env` file with:**
```env
DB_CONNECTION=pgsql
DB_HOST=xxx.supabase.co
DB_PORT=5432
DB_DATABASE=postgres
DB_USERNAME=postgres
DB_PASSWORD=your_password
```

---

## 3. VERIFY POSTGRESQL PHP EXTENSION

```bash
# Check if PostgreSQL PHP driver is installed
php -m | findstr pgsql

# If not installed, uncomment in php.ini:
# extension=pdo_pgsql
```

---

## 4. MIGRATE DATA - Option A (Recommended): pgloader

**Install pgloader:**
```bash
choco install pgloader
# or download from https://pgloader.io/
```

**Run migration:**
```bash
pgloader \
  mysql://root:TetAAOjCSjJSUiOMucyoyqEnAcrtbAUW@mysql.railway.internal:3306/railway \
  postgresql://postgres:YOUR_SUPABASE_PASSWORD@YOUR_PROJECT.supabase.co:5432/postgres
```

---

## 4b. MIGRATE DATA - Option B: mysqldump + psql

```bash
# Export with PostgreSQL formatting
mysqldump -h mysql.railway.internal -u root -p"TetAAOjCSjJSUiOMucyoyqEnAcrtbAUW" \
  --skip-extended-insert --compatible=postgresql railway > backup.sql

# Convert MySQL syntax to PostgreSQL
(Get-Content backup.sql) -replace 'utf8mb4', 'utf8' | Set-Content backup.sql

# Import to Supabase
psql -h your-project.supabase.co -U postgres -d postgres -f backup.sql
```

---

## 5. VERIFY DATA TRANSFER

```bash
# In Supabase SQL Editor (https://supabase.com):
SELECT COUNT(*) as user_count FROM users;
SELECT COUNT(*) as report_count FROM reports;
SELECT COUNT(*) as device_count FROM devices;
```

---

## 6. RUN LARAVEL MIGRATIONS

```bash
cd c:\Users\Admin\Documents\Thesis\smch-api

# Clear config cache
php artisan config:clear

# Test connection
php artisan tinker
>>> DB::select('SELECT 1')
>>> exit

# Run migrations
php artisan migrate

# Seed if needed
php artisan db:seed
```

---

## 7. RUN TESTS

```bash
# Test suite
php artisan test

# Check specific database operations
php artisan tinker
>>> User::count()
>>> Report::count()
>>> Device::count()
>>> exit
```

---

## 8. UPDATE DOCKER DEPLOYMENT

```bash
# Replace docker-compose.yml with docker-compose.yml.supabase
# Or update existing docker-compose.yml to remove 'db' service

# Update .env.docker with Supabase connection
# Update .env.railway with Supabase connection if using Railway
```

---

## QUICK ROLLBACK (if issues)

```bash
# Revert to MySQL
DB_CONNECTION=mysql
DB_HOST=mysql.railway.internal
DB_PORT=3306
DB_DATABASE=railway
DB_USERNAME=root
DB_PASSWORD=TetAAOjCSjJSUiOMucyoyqEnAcrtbAUW

# Clear cache
php artisan config:clear
php artisan optimize:clear

# Test connection
php artisan tinker
>>> DB::connection('mysql')->select('SELECT 1')
```

---

## COMMON ISSUES & FIXES

**Connection refused:**
```bash
# Check firewall/IP whitelist in Supabase dashboard
# Add your IP to allowed connections
# Or allow all IPs for testing: 0.0.0.0/0
```

**psql not found:**
```bash
# Install PostgreSQL client tools
choco install postgresql
```

**SSL certificate error:**
```bash
# Add to .env:
DB_SSLMODE=require
# or disable (not recommended): DB_SSLMODE=disable
```

**Laravel migrations failed:**
```bash
# Check schema syntax
php artisan migrate --step
php artisan migrate:rollback
php artisan migrate:fresh --seed
```

---

## SUPPORT DOCS

- Supabase Docs: https://supabase.com/docs
- Laravel Database: https://laravel.com/docs/database
- PostgreSQL Documentation: https://www.postgresql.org/docs/
