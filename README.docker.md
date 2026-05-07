# Quick Docker development

This repository contains two apps: the Laravel API (`smch-api`) and the frontend (`smch-web`). The supplied `docker-compose.yml` runs the API, frontend dev server, a MySQL database, and phpMyAdmin for convenience.

Files added:
- [docker-compose.yml](docker-compose.yml) : orchestration for local development
- [smch-api/.env.docker](smch-api/.env.docker) : example env file used by Docker

Quick start (Windows):

1. Copy the Laravel env and adjust secrets if needed:

```powershell
cd "c:\Users\Admin\Documents\Thesis\smch-api"
copy .env.example .env
```

2. (Optional) If you need to persist composer/vendor locally, run once locally or let the container run `composer install` on first start.

3. From the repo root, build and start everything:

```powershell
cd "c:\Users\Admin\Documents\Thesis"
docker compose up --build
```

4. After containers start:
- API: http://localhost:8000
- Frontend (Vite dev): http://localhost:5173
- phpMyAdmin: http://localhost:8080 (user: `root`, password: `rootpassword`)

Notes / next steps:
- You should run `php artisan key:generate` if it wasn't generated automatically.
- Update `smch-api/.env` to match DB credentials (DB_HOST=db, DB_DATABASE=smch, DB_USERNAME=smch, DB_PASSWORD=secret).
- If you prefer a production-ready setup (nginx + php-fpm), adapt `docker-compose.yml` and the API Dockerfile accordingly.
