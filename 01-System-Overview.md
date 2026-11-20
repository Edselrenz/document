# 01 — System Overview and Introduction

## 1. What Is the System?

### Plain-Language Overview
**Trip Mo, Trip Ko** (repository: `Edselrenz/Itinerary-App`) is a multi-platform travel itinerary planning and destination discovery application built specifically for **Nueva Ecija Province, Philippines**. It helps travelers discover local attractions, cafes, restaurants, resorts, churches, and essential services across all 32 cities and municipalities of Nueva Ecija, automatically generate multi-day travel schedules tailored to their budget and interests, or build custom day-by-day itineraries with interactive maps, turn-by-turn route previews, and offline access.

### Technical Overview
The system is a **decoupled, multi-client, offline-resilient full-stack application** composed of four deployable runtime tiers:
1. **Web / PWA & Mobile Client (`Project/`)**: A standalone, zoneless **Angular 22.2.3 + Ionic 9** application wrapped with **Capacitor 8.6.0** for Android APK builds and configured with Angular Service Worker (`ngsw-config.json`) for Progressive Web App (PWA) caching. Uses **Leaflet 1.9.4** and `leaflet.markercluster` for geospatial mapping.
2. **Native Windows Desktop Client (`desktop_client/`)**: A **Python Flet 1.1.2+** desktop application achieving feature and visual parity with the web client, featuring a custom `SlippyMapCanvas` tile renderer, local tile caching, offline JSON storage, and PyInstaller + Inno Setup 6 single-file Windows packaging (`installer/NuevaEcijaTravel.iss`).
3. **REST API Backend (`backend/`)**: A **Python 3.12 FastAPI** service backed by **SQLAlchemy 2.0**, **Alembic**, and **PostgreSQL 16** (with automatic fallback to a local SQLite database `nueva_ecija_travel.db` when PostgreSQL is unreachable). Exposes `/api/v1` endpoints for authentication, POI search/discovery, deterministic itinerary generation, itinerary CRUD, and turn-by-turn route computation.
4. **Edge Reverse Proxy (`cloudflare-worker/`)**: A **Cloudflare Worker** (`worker.js`) backed by **Cloudflare KV** (`TUNNEL_KV`) that provides a permanent HTTPS endpoint for GitHub Pages and mobile clients, dynamically routing traffic to a host machine's active Cloudflare Tunnel or ngrok tunnel with 24-hour edge caching for offline resilience.

### Application Metadata
| Property | Verified Value | Source Reference |
|---|---|---|
| **Brand Name** | Trip Mo, Trip Ko | `Project/capacitor.config.ts`, `installer/NuevaEcijaTravel.iss` |
| **Android Package ID** | `ph.nuevaecijatravel.app` | `Project/capacitor.config.ts` |
| **API Title & Version** | Nueva Ecija Travel Itinerary API `v0.1.0` | `backend/app/main.py` |
| **Desktop Installer Version** | `1.0.0` | `installer/NuevaEcijaTravel.iss` |
| **Supported Platforms** | Web Browser / PWA (GitHub Pages), Android (Capacitor APK), Windows x64 Desktop (Flet EXE/Installer) | `.github/workflows/deploy.yml`, `.github/workflows/release-builds.yml` |

---

## 2. Problems the System Solves

| Problem | Why It Matters | How the System Solves It | Handling Modules | Technical Mechanism |
|---|---|---|---|---|
| **Fragmented local tourism & POI data in Nueva Ecija** | Mainstream travel apps often lack structured, categorized, and locally curated data for provincial municipalities outside major hubs. | Consolidates 2,673 OpenStreetMap POIs and 18 curated featured destinations across 32 Nueva Ecija LGUs into 16 canonical categories, optionally augmented by live Google Places API (New). | `scripts/fetch_osm_places.py`, `scripts/import_osm.py`, `scripts/seed_featured_places.py`, `backend/app/api/v1/places.py`, `PlaceService` | Overpass QL bounding-box extraction (`15.15–16.15 N, 120.70–121.40 E`), deterministic tag-to-category mapping, PostgreSQL/SQLite persistence, and static JSON export (`osm_places.json`) bundled into Web & Desktop clients. |
| **Inefficient cross-province backtracking & unrealistic trip pacing** | Manual provincial trip planning frequently pairs distant towns on the same day or schedules closed/overnight venues during day trips. | Generates geographically clustered, road-paced itineraries automatically with transparent assumptions. | `backend/app/algorithms/itinerary_generator.py`, `SuggestedWizardPage`, `SuggestedWizardView` | SQL bounding-box pre-filtering (`25 km` default radius), multi-factor `score_place()` (preference match, Bayesian rating, Haversine proximity, opening-hours gate, budget fit, overnight/mountain guards), and 2-Opt TSP permutation ordering (`itertools.permutations`). |
| **Unreliable rural connectivity & intermittent host backend availability** | Travelers in rural Nueva Ecija (e.g., Minalungao, Pantabangan, Gabaldon) experience intermittent mobile data; pilot deployments also run on a tunneled local host. | Multi-layer offline resilience across Edge, Web/PWA, and Desktop clients. | `ConnectivityService`, `GuestStorageService`, `cloudflare-worker/worker.js`, `Project/ngsw-config.json`, `SlippyMapCanvas` | Bundled 2,673-POI `osm_places.json`, local guest storage (`@ionic/storage-angular` / `guest_storage.json`), offline itinerary clustering fallback, 24h Cloudflare KV edge cache, PWA Service Worker tile/API caching, and bundled/cached desktop map tiles. |
| **Friction of mandatory account creation before planning** | Requiring sign-up up front causes user drop-off when travelers just want to test routes or plan a quick trip. | Full Guest Mode with seamless, non-blocking migration to an authenticated cloud account upon login/registration. | `GuestPage`, `GuestStorageService`, `GuestMigrationService`, `desktop_client/core/storage.py` | Stores guest itineraries locally; upon authentication, `GuestMigrationService.migrateGuestData()` iterates local trips and POSTs them to `/api/v1/itineraries`. |
| **Lack of provincial commute & budget transparency** | Travelers need honest cost ranges (without treating missing fees as ₱0) and public transit guidance from provincial terminals. | Displays explicit verified vs. unverified cost summaries and municipality-level commute guides. | `commute-guide.data.ts`, `desktop_client/core/commute_guide.py`, `ItineraryBudgetSummary` | Tracks `unverifiedCount` alongside `verifiedMin`/`verifiedMax`, emits pricing notices in `itinerary.assumptions`, and maps municipalities to bus/UV Express/jeepney hub, drop-off, travel time, and fare ranges. |

---

## 3. System Objectives

Based on verified code and repository artifacts:
- **General Objective**: Provide an accessible, offline-capable, cross-platform travel itinerary planner tailored to Nueva Ecija Province that supports both automated trip generation and custom trip authoring.
- **Functional Goals**:
  - Search, filter, and map POIs across 16 canonical categories and 32 municipalities (`GET /api/v1/places`, `GET /api/v1/municipalities`).
  - Generate multi-day suggested itineraries up to 14 days with geographic clustering, budget awareness, and route polylines (`POST /api/v1/itineraries/suggest`).
  - Create, edit, reorder, and delete custom multi-day itineraries with linked POIs or custom off-grid stops (`POST/GET/PATCH/DELETE /api/v1/itineraries`).
  - Compute driving/walking routes with turn-by-turn legs and live GPS route tracking on mobile/web (`POST /api/v1/itineraries/compute-day-route`, `RouteTrackingService`).
  - Export itineraries to RFC 5545 iCalendar (`.ics`), Markdown/text summary, and JSON (`itinerary-export.util.ts`, `desktop_client/core/export.py`).
- **Technical Goals**:
  - Maintain strict data isolation (BOLA protection) for user-owned itineraries and Argon2id + JWT authentication.
  - Operate with zero paid API dependency by falling back gracefully from Google Maps Platform to OSRM + local OSM datasets + Haversine routing.
  - Achieve UI, workflow, and data-model parity across Web/PWA, Android APK, and Windows Desktop.
- **User Experience Goals**:
  - Non-blocking offline status banner (`OfflineHostBanner`), unsaved-changes route guard (`pendingChangesGuard`), dark/light theme persistence, and mobile-first 4-tab bottom navigation (`Home`, `Map`, `Saved`, `Profile`).

---

## 4. Technology Stack (Verified)

| Category | Technology & Version | Where Used | Why Used (Implementation Evidence) | Dependent Components |
|---|---|---|---|---|
| **Languages** | Python `3.12+` | `backend/`, `desktop_client/`, `scripts/` | Unified language for FastAPI backend, Flet desktop app, and OSM/DB/packaging scripts. | Backend API, Desktop Client, Build/DB Scripts |
| **Languages** | TypeScript `~6.0.0`, SCSS, HTML | `Project/` | Type-safe Ionic Angular frontend and Playwright E2E tests. | Web/PWA & Android Client |
| **Languages** | JavaScript (ES Modules) | `cloudflare-worker/worker.js` | Native Cloudflare Workers runtime for edge reverse proxy. | Edge Proxy |
| **Web/Mobile Framework** | Angular `22.2.3` (zoneless, standalone) + Ionic Angular `^9.0.0` | `Project/src/` | Mobile-first UI components, cross-platform Web/PWA + Android parity, lazy-loaded standalone routes. | All pages, tabs, shared components |
| **Mobile Wrapper** | Capacitor `8.6.0` (`@capacitor/android`, `app`, `geolocation`, `haptics`, `keyboard`, `status-bar`) | `Project/capacitor.config.ts`, `Project/android/` | Compiles Ionic Angular `www/` bundle into native Android APK with GPS, status bar, back button, and native secure storage. | `AppComponent`, `RouteTrackingService`, `SecureTokenStorageService` |
| **Desktop Framework** | Flet `>=1.1.2` + `flet-map>=0.1.0` | `desktop_client/` | Flutter-backed Python desktop UI enabling rapid parity with Ionic views and standalone Windows `.exe` packaging. | All desktop views & components |
| **Backend Framework** | FastAPI `>=0.115.0` + Uvicorn `>=0.30.0` | `backend/app/main.py` | Async/sync REST API with automatic OpenAPI docs (dev-only) and Pydantic v2 validation. | All `/api/v1/*` endpoints |
| **ORM & Migrations** | SQLAlchemy `>=2.0.30` + Alembic `>=1.13.2` + `psycopg[binary]>=3.2.0` | `backend/app/models/`, `backend/alembic/` | Portable `JsonType` (`JSONB` on PostgreSQL, `JSON` on SQLite) and versioned schema migrations (`0001`–`0004`). | All DB models, repositories, tests |
| **Database** | PostgreSQL `16-alpine` (Docker) & SQLite 3 (fallback) | `docker-compose.yml`, `backend/app/core/database.py` | Production relational store with zero-config local SQLite fallback (`nueva_ecija_travel.db`) if Postgres is unreachable. | Backend API, Backup/Restore scripts |
| **Auth & Security** | `pwdlib[argon2]>=0.2.1`, `pyjwt>=2.9.0`, `google-auth>=2.34.0`, `slowapi>=0.1.9`, `capacitor-secure-storage-plugin` | `backend/app/services/auth_service.py`, `backend/app/core/limiter.py`, `SecureTokenStorageService` | Argon2id password hashing, HS256 JWT issuance, server-side Google ID token verification, proxy-aware rate limiting, native Keystore token storage. | Auth routes, HTTP interceptor |
| **Mapping & GIS** | Leaflet `^1.9.4`, `leaflet.markercluster ^1.5.3`, Custom `SlippyMapCanvas` | `MapPage`, `ItineraryRouteMapComponent`, `desktop_client/components/slippy_map_canvas.py` | Interactive slippy tile maps, marker clustering for 2,673 POIs, encoded polyline rendering, and offline tile caching. | Map tab, Route previews, Live tracking |
| **HTTP Clients** | Angular `HttpClient` + RxJS `~7.8.0`, Python `httpx>=0.27.0` / `>=0.28.1` | `Project/src/app/core/services/`, `backend/app/services/google_maps_service.py`, `desktop_client/core/api_client.py` | Interceptor-based JWT refresh on web; async `httpx.AsyncClient` with refresh lock on desktop; sync `httpx.Client` on backend. | All API consumers |
| **Testing & Linting** | Pytest `>=8.3.0`, Vitest `~5.0.5`, Playwright `^1.64.1`, ESLint `10.10.0`, Prettier `3.9.10`, Ruff, Mypy | `backend/tests/`, `desktop_client/tests/`, `Project/src/**/*.spec.ts`, `Project/e2e/` | Automated backend, desktop, frontend unit, and mobile E2E testing plus CI quality gates. | CI workflows, local QA |
| **Packaging & DevOps** | Docker / Docker Compose, GitHub Actions, PyInstaller, Inno Setup 6, Wrangler | `backend/Dockerfile`, `.github/workflows/`, `scripts/build_*`, `installer/NuevaEcijaTravel.iss`, `cloudflare-worker/` | Automated GitHub Pages deployment, Android APK builds, Windows Setup.exe releases, and edge proxy deployment. | Release & deployment pipelines |

---

## 5. Target Users and User Roles

**Role-Based Access Control (RBAC) Status**: **Not Implemented.**
The `users` table (`backend/app/models/entities.py`) contains no `role`, `is_admin`, or `permissions` column. Authorization is strictly binary (**Unauthenticated Guest** vs. **Authenticated Traveler**) combined with object-level ownership (`Itinerary.user_id == current_user.id`).

| User Type | Status | Responsibilities & Capabilities | Accessible Modules | Restricted Actions |
|---|---|---|---|---|
| **Guest Traveler (Unauthenticated)** | Implemented & Verified | Browse/search 2,673+ POIs, explore interactive map, generate suggested itinerary drafts (`POST /api/v1/itineraries/suggest`), build custom itineraries stored locally on device, manage local favorites, export `.ics`/text, compute stateless routes (`POST /api/v1/itineraries/compute-day-route`). | Welcome, Guest Onboarding, Home, Map, Saved (local), Profile (guest view + favorites/settings), Suggested Wizard, Custom Builder, Itinerary Detail (local/draft). | Cannot persist itineraries to PostgreSQL/cloud (`POST/GET/PATCH/DELETE /api/v1/itineraries` require Bearer JWT), cannot edit cloud user profile (`GET/PATCH /api/v1/auth/me`), cannot sync across devices until signing in. |
| **Authenticated Traveler** | Implemented & Verified | All Guest capabilities plus cloud-persisted multi-day itineraries, cross-device sync, automatic migration of local guest trips on login/signup, profile editing (name, username, middle initial, phone, birthday, gender, avatar, default daily budget), and password changes. | All frontend and desktop views plus authenticated `/api/v1/auth/me` and `/api/v1/itineraries/*` endpoints. | Strictly restricted to own itineraries via BOLA checks (`user_id == current_user.id`); cannot view/modify other users' trips; no admin moderation panel. |
| **System Administrator / Operator** | Infrastructure-Only (No In-App Role) | Manages backend server, PostgreSQL/SQLite database, backups/restores, OSM imports, Cloudflare Worker KV tunnel sync (`X-Admin-Sync-Key`), and CI/CD releases via CLI scripts. | Host OS / Docker / `scripts/*` / Cloudflare Worker `/__sync_tunnel` & `/__status`. | No dedicated admin UI or admin API endpoints exist inside FastAPI. |

---

## 6. Scope and Limitations (Honest Implementation Audit)

| Capability | Status | Evidence & Notes |
|---|---|---|
| Email/Username + Password Auth & JWT Refresh | **Implemented & Verified** | `backend/app/api/v1/auth.py`, `auth_service.py`, `auth.interceptor.ts`, `desktop_client/core/api_client.py`. |
| Guest Mode & Post-Login Migration | **Implemented & Verified** | `GuestStorageService`, `GuestMigrationService`, `desktop_client/core/storage.py`. |
| POI Search, Filtering, Map Clustering & Live OSM | **Implemented & Verified** | 16 canonical categories, 2,673 bundled OSM POIs, `OsmOverpassService`, Leaflet MarkerCluster, Desktop `SlippyMapCanvas`. |
| Deterministic Suggested Itinerary Generator | **Implemented & Verified** | `backend/app/algorithms/itinerary_generator.py` (up to 14 days, Haversine clustering, Bayesian scoring, 2-Opt TSP) + offline client fallback. |
| Custom Itinerary Builder & In-Place Editor | **Implemented & Verified** | `CustomBuilderPage`, `ItineraryDetailPage`, `CustomBuilderView`, `TripDetailView` with `pendingChangesGuard`. |
| Turn-by-Turn Routing & GPS Tracking | **Implemented & Verified** | Google Routes v2 → OSRM → Haversine fallback; `RouteTrackingService` via `@capacitor/geolocation` on web/Android. |
| Google OAuth Sign-In | **Partially Implemented** | Backend `POST /api/v1/auth/oauth/google` is implemented and tested, but web (`login.page.ts`) and desktop (`login_view.py`) Google Sign-In buttons currently show a "disabled in this pilot build" notice. |
| Forgot Password / Password Recovery | **Partially Implemented (UI-Simulated)** | Web `LoginPage.submitRecoveryRequest()` and Desktop `LoginView._send_recovery()` simulate submission client-side; no backend recovery endpoint or email service exists. |
| Apple Sign-In | **Planned / Disabled** | `POST /api/v1/auth/oauth/apple` explicitly returns HTTP `501 Not Implemented` (`ENABLE_APPLE_AUTH=false`). |
| Favorites Cloud Sync | **Local-Only** | Favorites are stored only in `localStorage` (`ne_favorite_places_v1`) on web and `storage/favorites.json` on desktop; there is no `favorites` database table or backend endpoint. |
| Live Google Places & Routes API | **Referenced / Optional** | Requires `GOOGLE_MAPS_API_KEY`; when omitted, system gracefully uses local DB, OSM Overpass, OSRM, and Haversine fallbacks. |
| iOS Native Build | **Not Implemented** | Only `@capacitor/android` and `Project/android/` exist; no iOS Capacitor platform is configured. |
| Role-Based Access Control (RBAC) / Admin Panel | **Missing** | No user roles or admin endpoints exist. |
