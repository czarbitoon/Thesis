# SMC SURVIVAL: Supabase Migration via SQL Editor (No Tools Needed)

**For Windows Users - Zero Installation Required**
This approach uses only Supabase's web SQL Editor + Laravel Artisan commands.

---

## Step 1: Create Supabase Project (2 min)

1. Go to https://supabase.com → Create new project
2. Choose PostgreSQL, same region as your users
3. Note the connection details:
   - **Host**: `xxx.supabase.co`
   - **Port**: `5432`
   - **Database**: `postgres`
   - **User**: `postgres`
   - **Password**: (your secure password)

---

## Step 2: Get Your Schema Ready (5 min)

**In your Rails/Laravel project, run:**

```bash
cd c:\Users\Admin\Documents\Thesis\smch-api

# View your current database structure
php artisan tinker
>>> Schema::getConnection()->getDoctrineSchemaManager()->listTableNames()
```

**Or generate SQL of your schema:**
```bash
# Export current MySQL schema (schema only, no data)
mysql -h mysql.railway.internal -u root -p"TetAAOjCSjJSUiOMucyoyqEnAcrtbAUW" railway --no-data -e "SHOW CREATE TABLE users; SHOW CREATE TABLE reports; SHOW CREATE TABLE devices;" > schema.txt
```

---

## Step 3: Create Tables in Supabase (5 min)

### Go to: https://supabase.com → Your Project → SQL Editor

**Create tables one by one. Convert MySQL syntax to PostgreSQL:**

### Example: Users Table

**MySQL Original:**
```sql
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  profile_picture VARCHAR(255),
  email_verified_at TIMESTAMP NULL,
  password VARCHAR(255) NOT NULL,
  user_role VARCHAR(255) DEFAULT 'user',
  api_token VARCHAR(255) UNIQUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

**PostgreSQL for Supabase:**
```sql
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  profile_picture VARCHAR(255),
  email_verified_at TIMESTAMP NULL,
  password VARCHAR(255) NOT NULL,
  user_role VARCHAR(255) DEFAULT 'user',
  api_token VARCHAR(255) UNIQUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Key MySQL → PostgreSQL Changes:**
- `AUTO_INCREMENT` → `BIGSERIAL` or `SERIAL`
- `INT` → `INTEGER` or `BIGINT`
- `TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP` → `TIMESTAMP DEFAULT CURRENT_TIMESTAMP`
- Remove `utf8mb4` (PostgreSQL uses UTF-8 by default)
- Indexes: Create separately with `CREATE INDEX`

### Convert all your tables this way

**For reference, check your existing schema:**
```bash
# List all tables in Railway MySQL
mysql -h mysql.railway.internal -u root -p"TetAAOjCSjJSUiOMucyoyqEnAcrtbAUW" railway -e "SHOW TABLES;"
```

---

## Step 4: Export Data from Railway MySQL (10 min)

**Use CSV export (easiest for Windows):**

### Query in Railway MySQL:
```bash
mysql -h mysql.railway.internal -u root -p"TetAAOjCSjJSUiOMucyoyqEnAcrtbAUW" railway \
  -e "SELECT * FROM users;" > users.csv
```

**Or use TablePlus/DBeaver GUI:**
- Right-click table → Export → CSV

**Save files:**
- `users.csv`
- `reports.csv`
- `devices.csv`
- `issues.csv`
- (etc. for each table)

---

## Step 5: Import CSV Data into Supabase (10 min)

### In Supabase SQL Editor:

**Copy the CSV contents and run:**

```sql
-- Example: Import users
COPY users (id, name, email, profile_picture, email_verified_at, password, user_role, api_token, created_at, updated_at) 
FROM STDIN 
WITH (FORMAT csv, HEADER);
```

**Or use Supabase's UI:**
1. Go to **Table Editor** in Supabase
2. Click your table → **Import data**
3. Upload CSV file
4. Map columns
5. Click **Import**

---

## Step 6: Verify Data Transfer (5 min)

**In Supabase SQL Editor, run:**

```sql
-- Check record counts
SELECT COUNT(*) as user_count FROM users;
SELECT COUNT(*) as report_count FROM reports;
SELECT COUNT(*) as device_count FROM devices;

-- Sample check
SELECT * FROM users LIMIT 1;
SELECT * FROM reports LIMIT 1;
```

**Compare with Railway:**
```bash
mysql -h mysql.railway.internal -u root -p"TetAAOjCSjJSUiOMucyoyqEnAcrtbAUW" railway -e "SELECT COUNT(*) FROM users;"
```

---

## Step 7: Update Laravel .env (2 min)

**Your current `.env.example`:**
```env
DB_CONNECTION=mysql
DB_HOST=mysql.railway.internal
DB_PORT=3306
DB_DATABASE=railway
DB_USERNAME=root
DB_PASSWORD=TetAAOjCSjJSUiOMucyoyqEnAcrtbAUW
```

**Change to:**
```env
DB_CONNECTION=pgsql
DB_HOST=your-project.supabase.co
DB_PORT=5432
DB_DATABASE=postgres
DB_USERNAME=postgres
DB_PASSWORD=your-supabase-password
```

**Save to `.env` (your active file)**

---

## Step 8: Clear Laravel Cache & Test (2 min)

```bash
cd c:\Users\Admin\Documents\Thesis\smch-api

# Clear config
php artisan config:clear
php artisan cache:clear

# Test connection
php artisan tinker
>>> DB::select('SELECT 1')
>>> User::count()
>>> Report::count()
>>> exit
```

**Expected output: Numbers should match Supabase counts**

---

## Step 9: Run Laravel Migrations (Optional - 1 min)

**If your schema differs from migrations:**
```bash
php artisan migrate --step
# or
php artisan migrate:fresh --seed
```

---

## Done! No Installation Required ✅

---

## If Something Goes Wrong

### Can't connect to Supabase:
1. Check host is correct in `.env`
2. Verify password is correct
3. Click **Networking** in Supabase → Add your IP to whitelist (or allow 0.0.0.0)
4. Test: `php artisan tinker` → `DB::select('SELECT 1')`

### Data didn't transfer:
1. Verify counts match in Supabase SQL Editor
2. Check CSV formatting (quotes, commas, newlines)
3. Re-export from Railway MySQL with correct delimiter

### Laravel migrations failing:
```bash
# Check schema
php artisan tinker
>>> Schema::hasTable('users')
>>> Schema::getColumns('users')

# If schema is correct, migrations aren't needed
# Just update .env and go
```

### Rollback to MySQL (if needed):
```env
DB_CONNECTION=mysql
DB_HOST=mysql.railway.internal
DB_PORT=3306
DB_DATABASE=railway
DB_USERNAME=root
DB_PASSWORD=TetAAOjCSjJSUiOMucyoyqEnAcrtbAUW
```

Then: `php artisan config:clear` and test

---

## Summary: 5 Simple Steps

1. ✅ Create Supabase project
2. ✅ Create tables in Supabase SQL Editor (copy-paste MySQL schema + convert syntax)
3. ✅ Export CSV from Railway MySQL
4. ✅ Import CSV to Supabase
5. ✅ Update `.env` + Test with `php artisan tinker`

**Total time: ~30 minutes. Ready for SMC tomorrow.**
