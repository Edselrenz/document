# 03 — Project Directory Structure

This guide maps the complete repository layout of `Edselrenz/Itinerary-App` and explains the exact purpose, responsibilities, and operational criticality of every directory and key file.

---

## 1. Complete Repository Tree

```text
Itinerary-App/
├── .env.example                          # Backend environment variable template (Essential)
├── .gitignore                            # Git exclusion rules for secrets, caches, builds
├── .github/
│   └── workflows/
│       ├── deploy.yml                    # CI/CD: Backend pytest + Vitest + GitHub Pages deploy
│       └── release-builds.yml            # CI/CD: Android APK + Windows Setup.exe + GitHub Release
├── .vscode/                              # Shared VS Code workspace settings & recommended extensions
├── docker-compose.yml                    # Local PostgreSQL 16 container (Essential for PG dev)
├── docker-compose.prod.yml               # Production multi-container PostgreSQL 16 + FastAPI stack
├── start_server.bat                      # Windows launcher invoking scripts/start_server.ps1
├── stop_server.bat                       # Windows shutdown invoking scripts/stop_server.ps1
├── run_desktop_app.bat                   # Windows dev launcher: python desktop_client\main.py
├── Run_NuevaEcijaTravel_EXE.bat          # Windows launcher for compiled dist\NuevaEcijaTravel\*.exe
│
├── backend/                              # FastAPI REST API & Database Layer (Essential)
│   ├── Dockerfile                        # Multi-stage Python 3.12 production container image
│   ├── alembic.ini                       # Alembic CLI & logging configuration
│   ├── requirements.txt                  # Python backend dependencies
│   ├── alembic/
│   │   ├── env.py                        # Alembic migration runner bound to Base.metadata
│   │   └── versions/
│   │       ├── 0001_initial_nueva_ecija_schema.py   # 7 core tables
│   │       ├── 0002_google_places_and_routing.py    # Google GIS fields + geospatial_cache table
│   │       ├── 0003_widen_phone_column.py           # places.phone VARCHAR(64) -> VARCHAR(255)
│   │       └── 0004_add_user_profile_fields.py      # Extended users profile columns
│   ├── app/
│   │   ├── main.py                       # FastAPI app init, SlowAPI, CORS, router mounts, /health
│   │   ├── core/
│   │   │   ├── config.py                 # Pydantic BaseSettings & production secret validator
│   │   │   ├── database.py               # SQLAlchemy engine, SQLite fallback, auto-migration helper
│   │   │   └── limiter.py                # SlowAPI rate limiter with Cloudflare/proxy IP resolution
│   │   ├── models/
│   │   │   └── entities.py               # 8 SQLAlchemy 2.0 ORM models
│   │   ├── schemas/
│   │   │   ├── auth.py                   # Register, Login, Refresh, GoogleOAuth, UserUpdate, TokenOut
│   │   │   ├── place.py                  # MunicipalityOut, PlaceOut, LivePlaceSearch*, PaginatedPlacesOut
│   │   │   └── itinerary.py              # Itinerary/Day/Item CRUD, Suggest, Reorder, RouteCompute schemas
│   │   ├── api/v1/
│   │   │   ├── auth.py                   # /api/v1/auth/* endpoints (7 routes)
│   │   │   ├── places.py                 # /api/v1/municipalities & /api/v1/places/* (6 routes)
│   │   │   └── itineraries.py            # /api/v1/itineraries/* (12 routes, BOLA-enforced)
│   │   ├── services/
│   │   │   ├── auth_service.py           # Argon2id hashing, PyJWT, Google ID token verification
│   │   │   └── google_maps_service.py    # Places v1, Routes v2, OSRM, Haversine, KV/DB cache, upsert
│   │   └── algorithms/
│   │       └── itinerary_generator.py    # Clustering, Bayesian scoring, hours gate, 2-Opt TSP
│   └── tests/                            # 21 Pytest functions across 5 modules
│       ├── test_auth.py
│       ├── test_backup_restore.py
│       ├── test_foundation.py
│       ├── test_google_maps_service.py
│       └── test_itineraries.py
│
├── Project/                              # Ionic Angular 22 Web/PWA & Android Frontend (Essential)
│   ├── angular.json                      # Angular build, budgets, fileReplacements, NGSW config
│   ├── capacitor.config.ts               # Android appId (ph.nuevaecijatravel.app), webDir: www
│   ├── eslint.config.js                  # Flat ESLint config
│   ├── ionic.config.json                 # Ionic Angular standalone integration
│   ├── ngsw-config.json                  # PWA Service Worker asset & API/OSM tile cache policies
│   ├── package.json                      # NPM dependencies & build/lint/test scripts
│   ├── playwright.config.ts              # Mobile Chrome (Pixel 7) E2E test configuration
│   ├── vitest.config.mts                 # Vitest runner config (fsModuleCache: false)
│   ├── tsconfig*.json                    # Base, app, spec, and e2e TypeScript configs
│   ├── custom_itinerary_qa.mjs           # Standalone Node QA verification script
│   ├── android/                          # Capacitor Android Gradle project (JDK 21)
│   ├── e2e/
│   │   └── app.spec.ts                   # Playwright E2E user journey specs
│   └── src/
│       ├── main.ts                       # Zoneless Angular bootstrap, Leaflet window.L guard, SW
│       ├── index.html                    # SPA HTML shell
│       ├── 404.html                      # GitHub Pages SPA fallback redirect
│       ├── global.scss                   # Global Ionic & custom layout styles
│       ├── theme/variables.scss          # Figma color tokens & CSS custom properties
│       ├── environments/                 # environment.ts, environment.prod.ts, environment.example.ts
│       ├── assets/
│       │   ├── data/osm_places.json      # 2,673 pre-indexed Nueva Ecija OSM POIs (offline dataset)
│       │   ├── images/                   # Curated destination photography & splash/avatar assets
│       │   └── icon/favicon.png          # App icon
│       └── app/
│           ├── app.component.*           # Root shell: splash overlay, offline banner, Android back btn
│           ├── app.routes.ts             # Top-level lazy routes & redirects
│           ├── core/
│           │   ├── data/                 # commute-guide.data.ts, offline-places.data.ts
│           │   ├── guards/               # pending-changes.guard.ts
│           │   ├── interceptors/         # auth.interceptor.ts (Bearer injection + 401 refresh queue)
│           │   ├── models/               # auth.models.ts, place.models.ts, itinerary.models.ts
│           │   ├── services/             # 10 singleton services (+ 9 .spec.ts files)
│           │   └── utils/                # date.utils.ts, itinerary-export.util.ts, polyline.util.ts
│           ├── features/
│           │   ├── auth/                 # welcome/, login/, signup/, guest/
│           │   └── itinerary/            # custom-builder/, suggested-wizard/, itinerary-detail/
│           ├── shared/components/        # destination-autocomplete/, itinerary-route-map/, place-detail-sheet/
│           └── tabs/                     # tabs.page.*, tabs.routes.ts, home/, map/, history/, profile/, favorites/
│
├── desktop_client/                       # Python Flet 1.1.2 Desktop Application (Essential for Desktop)
│   ├── main.py                           # Entry point, route dispatcher, theme, splash, tab bar
│   ├── requirements.txt                  # Flet, flet-map, httpx, pydantic, pytest, python-dotenv
│   ├── .env.example                      # Desktop client environment template
│   ├── config/settings.py                # Palette tokens, window sizes, API base URL, categories
│   ├── core/                             # api_client.py, connectivity_service.py, session.py, storage.py,
│   │                                     # geo.py, polyline.py, commute_guide.py, offline_data.py, export.py
│   ├── models/                           # user.py, place.py, itinerary.py
│   ├── services/                         # auth_service.py, place_service.py, itinerary_service.py, favorites_service.py
│   ├── components/                       # 11 reusable UI widgets including slippy_map_canvas.py
│   ├── views/                            # 11 full-screen views mirroring Ionic Web pages
│   ├── assets/                           # osm_places.json (2,673 POIs), tiles/ (pre-seeded), tiles_cache/, images/, icon/
│   ├── storage/                          # Runtime local JSON persistence (session, guest_storage, favorites, avatars)
│   ├── scripts/clean_and_seed_tiles.py   # Offline tile maintenance utility
│   └── tests/                            # 115 Pytest functions across 10 test modules
│
├── cloudflare-worker/                    # Edge Reverse Proxy (Optional for Local, Essential for GH Pages)
│   ├── worker.js                         # Dynamic tunnel failover + 24h KV edge caching + CORS
│   ├── wrangler.toml                     # Worker name, KV namespace placeholder, dev sync key
│   └── README.md                         # Worker setup notes
│
├── installer/
│   └── NuevaEcijaTravel.iss              # Inno Setup 6 script producing NuevaEcijaTravel_Setup_v1.0.0.exe
│
├── scripts/                              # Operational, Data, Build & Verification Scripts
│   ├── setup_postgres.py                 # DB connectivity check, Alembic upgrade, automatic seeding
│   ├── start_postgres.ps1                # Starts Docker PostgreSQL and runs setup_postgres.py
│   ├── start_server.ps1                  # Starts DB, Uvicorn, ngrok + Cloudflare tunnels, Worker sync
│   ├── stop_server.ps1                   # Cleanly stops tunnels, Uvicorn (:8000), dev servers (:4200/:8100)
│   ├── run_daemon.py                     # Background Python daemon launching Uvicorn + dual tunnels
│   ├── db_backup_restore.py              # PostgreSQL (pg_dump/psql) & SQLite backup/restore CLI
│   ├── fetch_osm_places.py               # Queries Overpass API for Nueva Ecija POIs -> data/raw/
│   ├── import_osm.py                     # Idempotent Overpass JSON -> DB importer (16 categories)
│   ├── seed_featured_places.py           # Seeds 12 core municipalities & 18 curated featured POIs
│   ├── export_osm_to_client_asset.py     # Exports active SQLite places -> Project/src/assets/data/osm_places.json
│   ├── build_standalone_exe.py           # PyInstaller compiler bundling Flet runtime -> dist/NuevaEcijaTravel/
│   ├── build_desktop_installer.ps1       # Orchestrates PyInstaller + Inno Setup 6
│   ├── build_android_apk.ps1             # Orchestrates ng build + cap sync + Gradle APK build
│   └── verify_acceptance_gate.py         # 24-point automated MVP acceptance test script
│
├── build/                                # Committed PyInstaller build artifacts (non-essential; see Tech Debt)
├── docs/                                 # Historical project documentation & QA reports (32 files)
├── for_updates/                          # Phase 60–61 remediation notes
└── *.md, *.pdf, *.docx                   # Root-level historical QA reports, proposals, and parity plans (non-essential)
```

---

## 2. Directory Responsibilities & Criticality

| Path | Responsibility | Interacts With | Essential to Runtime? |
|---|---|---|---|
| `backend/app/` | Core REST API, auth, GIS service, itinerary algorithm, ORM models. | PostgreSQL/SQLite, Google Maps APIs, OSRM, Web/Desktop clients | **Yes** (required for cloud features) |
| `backend/alembic/` | Database schema version control (`0001`–`0004`). | `backend/app/models/entities.py`, PostgreSQL | **Yes** (for PostgreSQL deployments) |
| `Project/src/` | Ionic Angular Web/PWA & Android UI, services, Leaflet map, offline data. | Backend `/api/v1`, OSM Overpass/Tiles, Capacitor native plugins | **Yes** (Web/Mobile client) |
| `Project/android/` | Capacitor Android native shell and Gradle build scripts. | `Project/www/` (via `npx cap sync android`) | **Yes** (Android APK only) |
| `desktop_client/` | Standalone Python Flet Windows desktop app with offline tile map. | Backend `/api/v1`, `desktop_client/storage/`, OSM Tiles | **Yes** (Desktop client) |
| `cloudflare-worker/` | Permanent HTTPS reverse proxy + KV tunnel registry + 24h edge cache. | GitHub Pages, Android, Desktop, Host Tunnels | **Yes** (for GitHub Pages / remote pilot) |
| `scripts/` | Automation for DB setup, seeding, OSM ETL, server lifecycle, backups, and builds. | `backend/`, `Project/`, `desktop_client/`, Docker, Cloudflare Worker | **Yes** (for DevOps & local setup) |
| `installer/` | Inno Setup 6 definition for Windows `Setup.exe`. | `dist/NuevaEcijaTravel/` | **Optional** (Windows release packaging) |
| `.github/workflows/` | Automated CI tests, GitHub Pages deploy, and release binary builds. | `backend/`, `Project/`, `desktop_client/`, `installer/` | **Optional** (CI/CD automation) |
| `build/`, `docs/`, root `*.pdf`/`*.md` | Historical QA reports, proposals, and leftover PyInstaller intermediate `build/` artifacts. | None at runtime | **No** (reference / cleanup candidates) |

---

## 3. Developer Navigation Guide

- **Need to trace an API endpoint?** Start at `backend/app/main.py` → `backend/app/api/v1/{auth,places,itineraries}.py` → `backend/app/schemas/` → `backend/app/services/` or `backend/app/algorithms/` → `backend/app/models/entities.py`.
- **Need to modify a Web/Android screen?** Start at `Project/src/app/app.routes.ts` or `Project/src/app/tabs/tabs.routes.ts` → locate the page under `features/` or `tabs/` → check injected services in `core/services/`.
- **Need to modify a Desktop screen?** Start at `desktop_client/main.py` (route dispatcher in `navigate_to()`) → open the matching file in `desktop_client/views/` → check widgets in `desktop_client/components/` and services in `desktop_client/services/`.
- **Need to update offline POI data?** Run `scripts/fetch_osm_places.py` → `scripts/import_osm.py` → `scripts/seed_featured_places.py` → `scripts/export_osm_to_client_asset.py` (and mirror `osm_places.json` into `desktop_client/assets/data/osm_places.json`).
