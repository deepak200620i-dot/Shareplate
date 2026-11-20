# Deploying SharePlate (Agape) on Render

This guide provides complete instructions to deploy the entire SharePlate stack on [Render](https://render.com):
1. **Managed PostgreSQL Database**
2. **FastAPI Backend Web Service**
3. **React + Vite Frontend Static Site**

---

## Architecture Overview

```
               ┌───────────────────────────────┐
               │    SharePlate Frontend        │
               │  (Render Static Site)         │
               │  https://<frontend>.onrender  │
               └──────────────┬────────────────┘
                              │
               REST API calls │ Bearer Token + JSON/Multipart
               (/api/v1/*)    │
                              ▼
               ┌───────────────────────────────┐
               │     SharePlate Backend        │
               │  (Render Web Service/Python)  │
               │  https://<backend>.onrender   │
               └──────────────┬────────────────┘
                              │
                              │ Internal PostgreSQL Connection
                              ▼
               ┌───────────────────────────────┐
               │     Render PostgreSQL DB      │
               │       (Free / Starter)        │
               └───────────────────────────────┘
```

---

## Method 1: One-Click Blueprint Deployment (Recommended)

Render Blueprints use the included `render.yaml` to provision and link all 3 components automatically.

### Steps:
1. **Push your code to GitHub**:
   ```bash
   git add .
   git commit -m "Prepare repository for Render deployment"
   git push origin main
   ```
2. Log into your [Render Dashboard](https://dashboard.render.com).
3. Click **New +** in the top navigation and select **Blueprint**.
4. Connect your GitHub repository (`Shareplate`).
5. Render reads `render.yaml` and lists the 3 resources:
   - `shareplate-db` (PostgreSQL Database)
   - `shareplate-backend` (Web Service)
   - `shareplate-frontend` (Static Site)
6. Click **Apply**.
7. Render will build and deploy all services. Once completed:
   - Open your frontend URL (`https://shareplate-frontend.onrender.com`).
   - Log in with the pre-seeded admin credentials (see below).

---

## Method 2: Manual Dashboard Deployment (Step-by-Step)

If you prefer to configure services individually in the Render UI:

### Step 1: Create PostgreSQL Database
1. Go to **New +** -> **PostgreSQL**.
2. Set:
   - **Name**: `shareplate-db`
   - **Database**: `shareplate`
   - **User**: `shareplate_user`
   - **Region**: Choose the closest region (e.g., Oregon).
   - **Plan**: `Free`
3. Click **Create Database**.
4. Keep this page open; copy the **Internal Database URL** (e.g. `postgresql://shareplate_user:...@dpg-.../shareplate`).

---

### Step 2: Create Backend Web Service
1. Go to **New +** -> **Web Service**.
2. Select **Build and deploy from a Git repository** and connect your repository.
3. Configure settings:
   - **Name**: `shareplate-backend`
   - **Region**: Same region as database (e.g., Oregon).
   - **Branch**: `main`
   - **Root Directory**: `Agape/backend` (or `backend` if you deployed from within `Agape`)
   - **Runtime**: `Python`
   - **Build Command**: `pip install -r requirements.txt`
   - **Start Command**: `uvicorn app.main:app --host 0.0.0.0 --port $PORT`
   - **Plan**: `Free`
4. Add **Environment Variables** under *Advanced*:
   | Key | Value / Source |
   |---|---|
   | `DATABASE_URL` | Paste your Internal Database URL from Step 1 |
   | `APP_ENV` | `production` |
   | `JWT_SECRET` | Click *Generate* (or enter a strong random 32+ char secret) |
   | `CORS_ORIGINS` | `*` (or your frontend URL once deployed) |
   | `AUTO_CREATE_TABLES` | `1` |
   | `AUTO_SEED_ADMIN` | `1` |
   | `PRIVATE_UPLOAD_DIR` | `/tmp/agape_storage` |
   | `PYTHON_VERSION` | `3.11.9` |
5. Set **Health Check Path**: `/health`
6. Click **Create Web Service**. Wait for the status to show **Live**.
7. Copy your backend URL (e.g., `https://shareplate-backend.onrender.com`).

---

### Step 3: Create Frontend Static Site
1. Go to **New +** -> **Static Site**.
2. Connect your repository.
3. Configure settings:
   - **Name**: `shareplate-frontend`
   - **Branch**: `main`
   - **Root Directory**: `Agape/frontend`
   - **Build Command**: `npm install && npm run build`
   - **Publish Directory**: `dist`
4. Add **Environment Variables**:
   | Key | Value |
   |---|---|
   | `VITE_USE_MOCKS` | `false` |
   | `VITE_API_BASE` | `https://shareplate-backend.onrender.com/api/v1` *(replace with your backend URL from Step 2)* |
5. Add **Redirect / Rewrite Rules** (under Redirects/Rewrites tab):
   - **Type**: `Rewrite`
   - **Source**: `/*`
   - **Destination**: `/index.html`
   *(Note: The repository also includes `public/_redirects` which handles this automatically).*
6. Click **Create Static Site**.

---

## Default Admin Credentials

On first deployment, `AUTO_SEED_ADMIN=1` automatically provisions the demo administrator:

- **Email**: `admin@agape.demo`
- **Password**: `Admin@123`
- **Portal**: `/login` (select **Admin Portal** or log in directly)

---

## Verifying Deployment

1. Visit `https://<frontend-url>/login`.
2. Sign in with `admin@agape.demo` and `Admin@123`.
3. Open another browser/incognito window to test user flows:
   - Go to `/register` -> Register a new **Donor** (Restaurant, Grocery, etc.).
   - Go to `/register` -> Register a new **NGO**.
   - As **Admin**, approve their verification requests at `/admin/verifications`.
   - As **Donor**, create a food listing at `/provider`.
   - As **NGO**, view and reserve available listings on the map at `/ngo`.

---

## Maintenance & Free Tier Notes

- **Spin-down**: Render's free tier Web Services spin down after 15 minutes of inactivity. The first request after sleep may take ~30-50 seconds to wake up.
- **Storage**: Uploaded verification docs and proofs are stored in `/tmp/agape_storage`. For persistent uploads across container restarts on production, you can attach a Render Persistent Disk mounted to `/data/private` and set `PRIVATE_UPLOAD_DIR=/data/private`.
- **Database Sleep**: Render Free PostgreSQL databases are available for 30 days of continuous use or auto-expire unless upgraded to Starter ($7/mo).
