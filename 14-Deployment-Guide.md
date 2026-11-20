# 14 — Deployment and Production Management

This guide documents the **actual verified deployment pipelines** implemented in `.github/workflows/`, `backend/Dockerfile`, `docker-compose.prod.yml`, `cloudflare-worker/`, and `scripts/`, followed by **recommended production hardening** where cloud infrastructure is not yet provisioned.

---

## 1. Verified Deployment Targets

### 1.1 Web / PWA Deployment — GitHub Pages (Automated CI/CD)
- **Workflow**: `.github/workflows/deploy.yml` (triggers on push to `main` or `workflow_dispatch`).
- **Live URL**: `https://edselrenz.github.io/Itinerary-App/`.
- **Exact Pipeline Steps**:
  1. Python `3.12` setup → `pip install -r backend/requirements.txt` → `python -m pytest backend/tests/ -v`.
  2. Node.js `22` setup → `npm ci` (in `Project/`) → `npm run test:unit` (Vitest).
  3. Ensures `Project/src/environments/environment.prod.ts` exists (copies from `environment.example.ts` if absent).
  4. Builds production PWA bundle: `npm run build -- --base-href /Itinerary-App/` (outputs to `Project/www`, with `ngsw-config.json` Service Worker enabled).
  5. Prepares GitHub Pages SPA support: `touch Project/www/.nojekyll` and `cp Project/src/404.html Project/www/404.html`.
  6. Uploads `Project/www` via `actions/upload-pages-artifact@v3` and deploys via `actions/deploy-pages@v4`.

### 1.2 Android APK & Windows Desktop Installer — GitHub Releases (Automated CI/CD)
- **Workflow**: `.github/workflows/release-builds.yml` (triggers on `v*` tags or manual `workflow_dispatch`).
- **Job 1 — Android APK (`ubuntu-latest`, Node 22, Temurin JDK 21)**:
  - `npx ng build --configuration production --service-worker=false` (SW disabled for native WebView).
  - `npx cap sync android`
  - `./gradlew assembleDebug --no-daemon` → `TripMoTripKo-debug.apk`.
  - If repository secrets `ANDROID_KEYSTORE_BASE64`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, and `ANDROID_KEY_PASSWORD` are set, writes temporary `keystore.properties`, runs `./gradlew assembleRelease --no-daemon` → `TripMoTripKo-release-signed.apk`, and deletes keystore files immediately.
  - Local Windows equivalent: `.\scripts\build_android_apk.ps1 -BuildType debug` (or `release`, optional `-InstallToDevice`).
- **Job 2 — Windows Desktop Installer (`windows-latest`, Python 3.12, Inno Setup 6)**:
  - Caches Flet desktop Flutter engine (`flet_desktop.ensure_client_cached()`).
  - Runs `python scripts/build_standalone_exe.py` (PyInstaller → `dist/NuevaEcijaTravel/NuevaEcijaTravel.exe`).
  - Runs `ISCC.exe installer\NuevaEcijaTravel.iss` → `dist_installer/NuevaEcijaTravel_Setup_v1.0.0.exe` (LZMA2 ultra64, installs to `%LOCALAPPDATA%\Programs\NuevaEcijaTravel` with `PrivilegesRequired=lowest` — no UAC admin prompt).
  - Also archives `NuevaEcijaTravel_Portable_v1.0.0.zip`.
  - Local Windows equivalent: `.\scripts\build_desktop_installer.ps1`.
- **Job 3 — GitHub Release**: Generates `SHA256SUMS.txt` and publishes all `.apk`, `.exe`, and `.zip` assets via `softprops/action-gh-release@v2`.

### 1.3 Cloudflare Worker Edge Proxy (`cloudflare-worker/`)
- **Purpose**: Fixed HTTPS gateway (`https://ne-itinerary-api.edselrenz0.workers.dev`) routing traffic to the host machine's active Cloudflare Tunnel (`cfUrl`) or ngrok fallback (`ngrokUrl`) stored in Cloudflare KV (`TUNNEL_KV`).
- **Deployment Commands**:
  ```bash
  cd cloudflare-worker
  npx wrangler kv:namespace create TUNNEL_KV
  # Paste generated id / preview_id into wrangler.toml
  npx wrangler secret put ADMIN_SYNC_KEY
  npx wrangler deploy
  ```

### 1.4 Backend Container Deployment (`backend/Dockerfile` & `docker-compose.prod.yml`)
- **Dockerfile**: Multi-stage `python:3.12-slim` image copying `backend/app`, `backend/alembic`, `backend/alembic.ini`, and `scripts/`. Runs `uvicorn app.main:app --host 0.0.0.0 --port 8000` with `APP_ENV=production` and 30s `HEALTHCHECK` on `/api/v1/health`.
- **Startup Commands**:
  ```bash
  export POSTGRES_USER=ne_travel_user
  export POSTGRES_PASSWORD="<strong-unique-password>"
  export POSTGRES_DB=nueva_ecija_travel
  export JWT_SECRET_KEY="$(openssl rand -hex 32)"

  docker compose -f docker-compose.prod.yml up -d --build
  docker compose -f docker-compose.prod.yml exec backend alembic upgrade head
  docker compose -f docker-compose.prod.yml exec backend python scripts/seed_featured_places.py
  ```

---

## 2. Recommended Production Cloud Architecture (Not Yet Provisioned)

The current pilot architecture relies on a host machine running `start_server.bat` + Cloudflare Tunnel + Cloudflare Worker. For 24/7 public production availability, the following **recommended** migration is advised:
1. **Managed Backend Hosting**: Deploy `backend/Dockerfile` to Render, Koyeb, Railway, or Fly.io (note that `*.onrender.com` and `*.koyeb.app` are already in `Project/src/environments/environment.prod.ts`'s trusted hostname allowlist).
2. **Managed PostgreSQL**: Provision managed PostgreSQL 16 (e.g., Neon, Supabase, or Render Postgres) with automated daily snapshots and TLS (`sslmode=require`).
3. **HTTPS & Reverse Proxy**: Terminate TLS at the cloud load balancer / Cloudflare proxy; restrict `CORS_ORIGINS` strictly to `https://edselrenz.github.io` and `capacitor://localhost` (remove wildcard tunnel regex).
4. **Secrets & Observability**: Store `JWT_SECRET_KEY`, `DATABASE_URL`, `GOOGLE_MAPS_API_KEY`, and `GOOGLE_OAUTH_CLIENT_ID` in the cloud provider's secret manager; attach structured JSON logging and uptime alerting on `GET /api/v1/health`.
5. **Rollback Procedure**:
   - **Web/PWA**: Re-run previous passing commit on `.github/workflows/deploy.yml` or revert commit on `main`.
   - **Backend Container**: Re-deploy previous image tag and, if a migration caused regression, run `alembic downgrade -1` (after restoring from pre-migration backup via `scripts/db_backup_restore.py restore`).
