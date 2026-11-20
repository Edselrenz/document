# 05 — Backend Architecture Documentation

The backend (`backend/`) is a **Python 3.12 FastAPI** REST service providing authentication, Nueva Ecija POI discovery, deterministic itinerary generation, itinerary persistence, and resilient turn-by-turn routing.

---

## 1. Application Entry Point & Initialization (`backend/app/main.py`)

When Uvicorn starts `app.main:app`:
1. **Settings Hydration (`backend/app/core/config.py`)**: Instantiates `Settings(BaseSettings)`, loading `<repo_root>/.env` and `.env` (UTF-8, `extra="ignore"`). A Pydantic `@model_validator(mode="after")` (`validate_secrets()`) raises a `ValueError` at startup if `APP_ENV != "development"` and `JWT_SECRET_KEY` is blank, shorter than 32 characters, or matches known insecure defaults.
2. **Database Engine & Fallback (`backend/app/core/database.py`)**:
   - Calls `resolve_db_url(settings.database_url)` to convert relative SQLite paths to `<repo_root>/nueva_ecija_travel.db` and inject `connect_timeout=2` on PostgreSQL URLs.
   - Creates the SQLAlchemy 2.0 `engine` (`pool_pre_ping=True`) and immediately executes a live test connection for non-SQLite URLs.
   - If PostgreSQL is unreachable, logs a warning and falls back to `sqlite:///<repo_root>/nueva_ecija_travel.db` (`check_same_thread=False`).
   - Executes `ensure_user_profile_columns(engine)` at import time to add missing extended `users` columns (`middle_initial`, `phone_number`, `birthday`, `gender`, `avatar_url`, `default_daily_budget`) across both PostgreSQL and SQLite. *(See [16 — Technical Debt](./16-Technical-Debt-and-Roadmap.md) regarding the hardcoded default backfill values inside this function.)*
3. **FastAPI Instance**:
   - `title="Nueva Ecija Travel Itinerary API"`, `version="0.1.0"`.
   - OpenAPI Swagger (`/docs`) and ReDoc (`/redoc`) are **enabled only when `settings.app_env == "development"`** and set to `None` in production.
4. **Middleware Stack**:
   - **SlowAPI Rate Limiter (`backend/app/core/limiter.py`)**: Attached via `app.state.limiter = limiter`, `RateLimitExceeded` exception handler, and `SlowAPIMiddleware`. Default limit is `120/minute` per resolved client IP (`headers_enabled=False`).
   - **CORS Middleware**: Configured with `allow_origins=settings.cors_origins`, `allow_origin_regex=settings.cors_origin_regex`, `allow_credentials=True`, `allow_methods=["GET", "POST", "PATCH", "PUT", "DELETE", "OPTIONS"]`, `allow_headers=["*"]`.
5. **Router Mounting**:
   - `auth_router` → `/api/v1/auth`
   - `places_router` → `/api/v1` (`/municipalities`, `/places/*`)
   - `itineraries_router` → `/api/v1/itineraries`
   - `GET /api/v1/health` directly on `app` (`tags=["system"]`).

---

## 2. Module Responsibilities

| Module | File Path | Core Responsibilities |
|---|---|---|
| **Config** | `backend/app/core/config.py` | Environment variable schema, defaults, CORS allowlist/regex, Google cache TTLs (`168h` places, `24h` details, `336h` routes), production JWT secret validation. |
| **Database** | `backend/app/core/database.py` | Engine creation, PostgreSQL → SQLite fallback, `SessionLocal`, `Base(DeclarativeBase)`, `get_db()` generator dependency, `ensure_user_profile_columns()`. |
| **Rate Limiter** | `backend/app/core/limiter.py` | `get_client_ip(request)` extracts `CF-Connecting-IP` or first IP in `X-Forwarded-For` **only** when direct socket host is loopback (`127.0.0.1`, `localhost`, `::1`, `testclient`) to prevent spoofing on direct connections. |
| **Auth Router & Service** | `backend/app/api/v1/auth.py`, `backend/app/services/auth_service.py` | Argon2id password hashing (`pwdlib.PasswordHash.recommended()`), HS256 JWT creation/decoding, `get_current_user` & `get_current_user_optional` dependencies, Google ID token verification (`google.oauth2.id_token`), Apple 501 gate. |
| **Places Router** | `backend/app/api/v1/places.py` | `CANONICAL_CATEGORIES` (16 categories), `escape_like()` SQL wildcard escaping, municipality search, multi-filter paginated POI search (max `limit=5000`), live Google discovery, photo proxy. |
| **Google Maps Service** | `backend/app/services/google_maps_service.py` | `GoogleMapsService` singleton (`google_maps_service`): Places v1 Text & Nearby Search (Basic FieldMask), Place Details (Rich FieldMask), Routes v2 Directions, OSRM fallback, Haversine fallback, `GeospatialCache` read/write, `upsert_google_place_to_db()`, polyline encode/decode. |
| **Itineraries Router** | `backend/app/api/v1/itineraries.py` | Strict BOLA-protected CRUD (`Itinerary.user_id == current_user.id`), eager loading (`selectinload` across `days → items → place`), `resolve_or_upsert_item_place()`, stop reorder, stateless & persistent day route computation. |
| **Itinerary Generator** | `backend/app/algorithms/itinerary_generator.py` | Deterministic multi-day planner: SQL bounding-box pre-filter, preference expansion, `score_place()`, Bayesian rating, opening-hours gate, fuzzy deduplication, 2-Opt TSP permutation ordering, dynamic road pacing, and transparent `assumptions` generation. |

---

## 3. Request-Response Lifecycle

1. **Client Request**: Web/Android (`authInterceptor`) or Desktop (`ApiClient`) sends HTTPS JSON to the Cloudflare Worker (or local Uvicorn `:8000`).
2. **Edge Proxy (Remote Mode)**: `cloudflare-worker/worker.js` handles `OPTIONS` CORS preflight, reads `ACTIVE_TUNNELS` from `TUNNEL_KV`, forwards to `cfUrl` (failing over to `ngrokUrl`), and caches `200 OK` responses for `GET /api/v1/places*` and `POST */compute-day-route` for 24 hours.
3. **CORS & Rate Limit Middleware**: FastAPI evaluates `CORSMiddleware` and `SlowAPIMiddleware` (`get_client_ip` checks proxy headers on loopback; returns HTTP `429 Too Many Requests` if quota exceeded).
4. **Dependency Injection**:
   - `get_db()` opens a scoped `SessionLocal` and guarantees `db.close()` in `finally`.
   - Protected routes invoke `get_current_user()` (or `get_current_user_optional()` on `/itineraries/suggest`), extracting the `Authorization: Bearer` token via `OAuth2PasswordBearer`, verifying signature, `exp`, and `type == "access"`, and loading the active `User`.
5. **Pydantic v2 Validation & Coercion**: Request bodies are validated and normalized (e.g., `ItineraryItemCreate` accepts both Web `order_index` and Desktop `sort_order`, parses 12-hour AM/PM or 24-hour `planned_time`, and preserves non-UUID offline place IDs inside `place_payload`).
6. **Controller, Service & Database Execution**:
   - Controllers enforce BOLA (`where(Itinerary.id == id, Itinerary.user_id == current_user.id)`), returning HTTP `404` on ownership mismatch to prevent resource existence leakage.
   - Database writes commit atomically via `db.commit()` (or rollback on `IntegrityError`).
7. **Serialization & Response**: Controllers return Pydantic `response_model` schemas (`TokenOut`, `PlaceOut`, `PaginatedPlacesOut`, `ItineraryOut`, `RouteComputeResponse`) as JSON.

---

## 4. Error Handling & Logging

- **HTTP Exceptions**: Explicit `HTTPException` responses with `detail` strings (`400`, `401`, `403`, `404`, `409`, `422`, `429`, `501`, `503`).
- **Validation Errors**: FastAPI automatically returns HTTP `422 Unprocessable Entity` with Pydantic field-level error arrays.
- **Logging**: Standard Python `logging.getLogger(...)` used in `database.py`, `google_maps_service.py`, and `itinerary_generator.py`. No centralized log shipper, structured JSON formatter, or request correlation ID middleware is currently configured.

---

## 5. External API Integrations

| Service | Provider | Base URL / Endpoint | Auth Method | Caching / TTL | Fallback Behavior on Outage | Source File |
|---|---|---|---|---|---|---|
| **Google Places API (New) — Text & Nearby Search** | Google Maps Platform | `https://places.googleapis.com/v1/places:searchText`, `:searchNearby` | `X-Goog-Api-Key` header (`GOOGLE_MAPS_API_KEY`) + `DISCOVERY_FIELD_MASK` | `geospatial_cache` (`168h` / 7 days) | Falls back automatically to local DB search (`_search_places_local` / `_search_places_local_nearby`) returning `source: "local_db"`. | `backend/app/services/google_maps_service.py` |
| **Google Places API (New) — Details & Photos** | Google Maps Platform | `https://places.googleapis.com/v1/places/{id}` & `/media` | `X-Goog-Api-Key` + `DETAILS_FIELD_MASK` (photo key kept server-side via `/api/v1/places/photos/{photo_name:path}`) | `geospatial_cache` (`24h`), `Cache-Control: public, max-age=86400` on photo proxy | Details fall back to local `places` table; photo proxy returns a 1×1 transparent SVG placeholder. | `google_maps_service.py`, `places.py` |
| **Google Routes API (Directions v2)** | Google Maps Platform | `https://routes.googleapis.com/directions/v2:computeRoutes` | `X-Goog-Api-Key` + `ROUTES_FIELD_MASK` | `geospatial_cache` (`336h` / 14 days) | Falls back to **OSRM Public Router**, then to **Offline Haversine Direct** polyline + estimated speed (`40 km/h` drive, `4.5 km/h` walk). | `google_maps_service.py` |
| **OSRM Public Routing** | Project OSRM | `https://router.project-osrm.org/route/v1/driving/{coords}` | None (public, `3.0s` timeout) | `geospatial_cache` (`336h`) | Falls back to `_compute_haversine_route()` (`source: "haversine_direct"`). | `google_maps_service.py` |
| **Google OAuth2 Token Verification** | Google Identity | Verified via `google.oauth2.id_token.verify_oauth2_token` | `GOOGLE_OAUTH_CLIENT_ID` audience check + issuer check | Google public cert cache (`google-auth`) | Returns HTTP `503` if client ID unconfigured; HTTP `401` on verification failure. | `backend/app/services/auth_service.py` |
| **OpenStreetMap Overpass API** | OpenStreetMap (Client & Scripts) | `overpass-api.de`, `overpass.kumi.systems`, `maps.mail.ru` | None (`User-Agent` header) | Client in-memory / bundled `osm_places.json` | Cycles through mirror endpoints; frontend falls back to bundled 2,673-POI dataset. | `osm-overpass.service.ts`, `scripts/fetch_osm_places.py` |
| **OpenStreetMap Tile Server** | OpenStreetMap (Clients) | `https://tile.openstreetmap.org/{z}/{x}/{y}.png` | None (`User-Agent` on desktop) | PWA Service Worker (14d, 1,500 tiles) + Desktop `assets/tiles/` & `tiles_cache/` | Serves cached/bundled tiles or inline SVG placeholder on desktop. | `environment.ts`, `ngsw-config.json`, `slippy_map_canvas.py` |
