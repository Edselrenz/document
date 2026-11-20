# 11 — System Administration and Maintenance

This manual documents the operational procedures supported by the actual scripts, containers, and edge infrastructure in `Edselrenz/Itinerary-App`.

---

## 1. Service Lifecycle Management

### Windows Host Operations (Pilot / Tunneled Mode)
- **Start Full Stack + Dual Tunnels**: Run `start_server.bat` (executes `scripts/start_server.ps1`).
  1. Checks `.env` for SQLite vs. PostgreSQL; starts Docker PostgreSQL container if needed.
  2. Runs `python scripts/setup_postgres.py` to verify schema & seed data.
  3. Starts Uvicorn on `0.0.0.0:8000`.
  4. Launches Cloudflare Quick Tunnel (`cloudflared`) and ngrok fallback tunnel, and POSTs active URLs to `WORKER_SYNC_URL/__sync_tunnel` using `X-Admin-Sync-Key`.
- **Start Local Only (No Tunnels)**: `powershell -File scripts/start_server.ps1 -NoTunnel`.
- **Stop All Services**: Run `stop_server.bat` (executes `scripts/stop_server.ps1`):
  - Terminates `cloudflared` and `ngrok` processes and removes `%TEMP%\ne_cloudflared.log` / `ne_ngrok.log`.
  - Terminates processes bound to ports `8000` (FastAPI), `4200` (Angular), and `8100` (Ionic).
  - Pass `-KeepDatabase` to leave PostgreSQL running.

### Docker Production Stack (`docker-compose.prod.yml`)
```bash
# Start PostgreSQL 16 + FastAPI container in background
docker compose -f docker-compose.prod.yml up -d --build

# Check container health & logs
docker compose -f docker-compose.prod.yml ps
docker compose -f docker-compose.prod.yml logs -f backend

# Restart backend after .env update
docker compose -f docker-compose.prod.yml restart backend

# Stop stack (preserves ne_postgres_prod_data volume)
docker compose -f docker-compose.prod.yml down
```

---

## 2. Health Monitoring & Logs

| Component | Health Endpoint / Command | Expected Healthy Output |
|---|---|---|
| **FastAPI Backend** | `curl -f http://localhost:8000/api/v1/health` (also Docker `HEALTHCHECK` every 30s) | `{"status":"ok","province_scope":"Nueva Ecija","environment":"..."}` |
| **Cloudflare Worker Proxy** | `curl https://ne-itinerary-api.<subdomain>.workers.dev/__status` | `{"status":"ok","worker":"nueva-ecija-api-proxy","active_tunnels":{...}}` |
| **PostgreSQL Container** | `docker exec ne_postgres pg_isready -U ne_travel_user -d nueva_ecija_travel` | `accepting connections` |
| **Backend Logs** | Docker: `docker compose logs -f backend`; Local: Uvicorn stdout/stderr | Standard Python `logging` + Uvicorn access logs |
| **Tunnel Logs (Windows)** | `%TEMP%\ne_cloudflared.log`, `%TEMP%\ne_ngrok.log`, `scratch/active_tunnel.txt` | Active `*.trycloudflare.com` and `*.ngrok-free.dev` URLs |

> **Administrative Limitation**: There is no in-app admin panel for managing users. User deactivation (`users.is_active = false`) or manual inspection must be performed directly via `psql` or SQLite CLI.

---

## 3. Database Backup, Restore & Migrations

### Automated Backup & Restore (`scripts/db_backup_restore.py`)
Supports both PostgreSQL (`pg_dump` / `psql`) and SQLite (binary snapshot or `.sql` `iterdump()`):
```bash
# Create timestamped backup in backups/
python scripts/db_backup_restore.py backup

# Create backup at explicit path
python scripts/db_backup_restore.py backup --output backups/pre_release_snapshot.sql

# Restore from backup
python scripts/db_backup_restore.py restore backups/pre_release_snapshot.sql
```
> **Recommended Maintenance Cadence**: Take a backup before every `alembic upgrade head`, OSM bulk import, or production release.

### Applying Schema Migrations
```bash
# Local / Host
cd backend && alembic upgrade head

# Inside Docker production container
docker compose -f docker-compose.prod.yml exec backend alembic upgrade head
```

---

## 4. POI Catalog & Offline Asset Refresh

When refreshing OpenStreetMap data for Nueva Ecija:
```bash
# 1. Fetch latest POIs from Overpass API -> data/raw/nueva_ecija_osm.json
python scripts/fetch_osm_places.py

# 2. Preview import changes (idempotent upsert by source_name + source_record_id)
python scripts/import_osm.py data/raw/nueva_ecija_osm.json --dry-run

# 3. Apply import and re-verify curated featured venues
python scripts/import_osm.py data/raw/nueva_ecija_osm.json
python scripts/seed_featured_places.py

# 4. Regenerate bundled offline JSON for Web/Android and mirror to Desktop
python scripts/export_osm_to_client_asset.py
cp Project/src/assets/data/osm_places.json desktop_client/assets/data/osm_places.json
```

---

## 5. Cache & Storage Maintenance

- **Backend `geospatial_cache` Table**: Expired entries are ignored by `expires_at > now()`, but no automatic background purge job exists. Periodically clean expired rows via SQL:
  ```sql
  DELETE FROM geospatial_cache WHERE expires_at < NOW();
  ```
- **Cloudflare KV Edge Cache**: Entries (`EDGE_CACHE_*`) auto-expire after `86400s` (24 hours).
- **Desktop Tile Cache**: Pre-seeded tiles live in `desktop_client/assets/tiles/`; runtime tiles cache in `desktop_client/assets/tiles_cache/` or `storage/tile_cache/`. Use `python desktop_client/scripts/clean_and_seed_tiles.py` to maintain offline tiles.
- **Dependency Updates**:
  - Backend: update `backend/requirements.txt` and run `python -m pytest backend/tests/ -v`.
  - Frontend: update `Project/package.json`, run `npm run test:unit && npm run lint`.
  - Desktop: update `desktop_client/requirements.txt` and run `python -m pytest desktop_client/tests/ -v`.
