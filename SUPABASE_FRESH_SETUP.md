# Fresh Supabase Setup (No Railway) - QUICK START

**Total Time: ~10 minutes**

---

## Step 1: Create Supabase Project (3 min)

1. Go to **https://supabase.com**
2. Sign up / Log in
3. Click **New Project**
4. Fill in:
   - **Project Name**: `smch-api` (or your choice)
   - **Database Password**: Use a strong password (save it!)
   - **Region**: Choose closest to your users (or `us-east-1`)
   - **Pricing Plan**: Free tier is fine for now
5. Click **Create new project** and wait for setup (~2 min)

---

## Step 2: Get Your Connection Details (1 min)

Once Supabase project is ready:

1. Go to **Settings → Database**
2. Under **Connection String**, copy:
   - **Host**: `xxx.supabase.co`
   - **Port**: `5432`
   - **Database**: `postgres`
   - **User**: `postgres`
   - **Password**: (your database password)

**Save these somewhere safe**

---

## Step 3: Update Laravel .env (1 min)

Open `.env` in your project:

```bash
cd c:\Users\Admin\Documents\Thesis\smch-api
```

Replace your database section with:

```env
DB_CONNECTION=pgsql
DB_HOST=your-project-xxx.supabase.co
DB_PORT=5432
DB_DATABASE=postgres
DB_USERNAME=postgres
DB_PASSWORD=your-database-password
```

**Example (fill in your actual values):**
```env
DB_CONNECTION=pgsql
DB_HOST=abc123def456.supabase.co
DB_PORT=5432
DB_DATABASE=postgres
DB_USERNAME=postgres
DB_PASSWORD=SuperSecurePassword123!
```

---

## Step 4: Run Migrations (2 min)

```bash
cd c:\Users\Admin\Documents\Thesis\smch-api

# Clear cache
php artisan config:clear
php artisan cache:clear

# Run all migrations (creates all tables from your schema)
php artisan migrate
```

**Expected output:**
```
Migration table created successfully.
Migrating: 0001_01_01_000000_create_users_table
Migrated:  0001_01_01_000000_create_users_table (123ms)
...
```

---

## Step 5: Verify Connection (1 min)

```bash
php artisan tinker

# Test queries
>>> DB::select('SELECT 1')
>>> User::count()
>>> exit
```

**If you get output, you're connected!** ✅

---

## Step 6: Update Docker Config (Optional - 1 min)

Remove MySQL from `docker-compose.yml`:

**Find this section:**
```yaml
  db:
    image: mysql:8.0
    container_name: smch-mysql
    ...
  phpmyadmin:
    ...
```

**Replace with comment:**
```yaml
  # Database: Using Supabase PostgreSQL (cloud-hosted)
  # No local database service needed
```

---

## Done! 🎉

Your Laravel app is now connected to Supabase PostgreSQL.

**Next time you start the app:**
```bash
cd c:\Users\Admin\Documents\Thesis\smch-api
php artisan serve
```

---

## Troubleshooting

**Connection refused:**
- Verify host, port, password in `.env`
- Go to Supabase → **Networking** → Add your IP (or allow 0.0.0.0 for testing)

**Migration failed:**
```bash
# Check what migrations exist
php artisan migrate:status

# Rollback and retry
php artisan migrate:rollback
php artisan migrate
```

**Schema doesn't match:**
- Your migrations should match current schema
- Check `database/migrations/*` files
- Update migrations if needed before running migrate

---

## Your Supabase Project is Ready!

- **Dashboard**: https://supabase.com
- **SQL Editor**: Run raw queries
- **Table Editor**: Visual data management
- **Real-time**: Built-in subscriptions
- **Auth**: Built-in authentication (if needed)

That's it. Ready to go to SMC. 🚀
