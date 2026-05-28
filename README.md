# Project Overview

This monorepo contains three main applications:

- **smch-api**: A Laravel-based backend API.
- **smch-mobile-app**: An Expo-based mobile application.
- **smch-web**: A React-Vite web application.

## Table of Contents

- [Project Overview](#project-overview)
- [Setup and Installation](#setup-and-installation)
  - [Docker Development](#docker-development)
  - [smch-api (Laravel API)](#smch-api-laravel-api)
  - [smch-mobile-app (Expo Mobile App)](#smch-mobile-app-expo-mobile-app)
  - [smch-web (React-Vite Web App)](#smch-web-react-vite-web-app)
- [Deployment](#deployment)
  - [smch-api Deployment to Vercel](#smch-api-deployment-to-vercel)
  - [smch-web Deployment to Vercel](#smch-web-deployment-to-vercel)

## Setup and Installation

### Docker Development

For quick local development of the `smch-api` and `smch-web` applications, you can use Docker Compose. This sets up the API, frontend dev server, a MySQL database, and phpMyAdmin.

**Quick Start (Windows):**

1.  **Copy Laravel environment file:**

    ```powershell
    cd "c:\Users\Admin\Documents\Thesis\smch-api"
    copy .env.example .env
    ```

2.  **(Optional) Install Composer dependencies locally:**
    If you need to persist `composer/vendor` locally, run `composer install` once. Otherwise, the Docker container will handle it on first start.

3.  **Build and start all services:**
    From the repository root (`c:\Users\Admin\Documents\Thesis`):

    ```powershell
    docker compose up --build
    ```

4.  **Access applications after containers start:**
    -   **API:** `http://localhost:8000`
    -   **Frontend (Vite dev):** `http://localhost:5173`
    -   **phpMyAdmin:** `http://localhost:8080` (User: `root`, Password: `rootpassword`)

**Notes:**

-   Run `php artisan key:generate` if it wasn't generated automatically.
-   Update `smch-api/.env` with Docker database credentials (`DB_HOST=db`, `DB_DATABASE=smch`, `DB_USERNAME=smch`, `DB_PASSWORD=secret`).

### smch-api (Laravel API)

Refer to `smch-api/README.md` for specific Laravel development details.

**API URL Configuration:**

-   **Local Development:** `http://localhost:8000/api`

### smch-mobile-app (Expo Mobile App)

Refer to `smch-mobile-app/README.md` for specific Expo development details.

**API URL Configuration:**

To set the backend API URL, create a `.env` file in `smch-mobile-app`:

```
EXPO_PUBLIC_API_URL=https://api.yourdomain.com/api
```

For local development:

```
EXPO_PUBLIC_API_URL=http://localhost:8000/api
```

### smch-web (React-Vite Web App)

Refer to `smch-web/README.md` for specific React-Vite development details.

**API URL Configuration:**

To set the backend API URL, create a `.env` file in `smch-web`:

```
VITE_API_BASE_URL=https://api.yourdomain.com
```

For local development:

```
VITE_API_BASE_URL=http://localhost:8000
```

## Deployment

### smch-api Deployment to Vercel

1.  Push this repository to GitHub, GitLab, or Bitbucket.
2.  Go to [vercel.com/import](https://vercel.com/import) and import your project.
3.  Set the **Framework Preset** to `Other`.
4.  Set the build command to `composer install && php artisan config:cache && php artisan route:cache`.
5.  Set the output directory to `public`.
6.  Add environment variables in **Project Settings > Environment Variables** (copy from `.env.example`).
7.  For storage (uploads, logs), use a cloud storage provider or Vercel's recommended solutions.
8.  Your API will be live on your Vercel domain after deployment.

### smch-web Deployment to Vercel

1.  Push this repository to GitHub, GitLab, or Bitbucket.
2.  Go to [vercel.com/import](https://vercel.com/import) and import your project.
3.  Set the **Framework Preset** to `Vite`.
4.  If you use environment variables, add them in **Project Settings > Environment Variables**.
5.  The default build command is `npm run build` and the output directory is `dist`.
6.  Your site will be live on your Vercel domain after deployment.
