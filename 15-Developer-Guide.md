# 15 — Developer Contribution and Continued Development Guide

This guide helps incoming engineers navigate the codebase quickly and implement new features consistently across the Backend, Ionic Angular Web/Android client, and Python Flet Desktop client.

---

## 1. Codebase Navigation Matrix

| Task | Where to Look First | Related Files |
|---|---|---|
| **Modify a Web/Android page** | `Project/src/app/app.routes.ts` or `tabs/tabs.routes.ts` | `features/*/*.page.{ts,html,scss,spec.ts}` or `tabs/*/*.page.*` |
| **Modify a Desktop view** | `desktop_client/main.py` (`navigate_to()`) | `desktop_client/views/*.py`, `desktop_client/components/*.py` |
| **Add or change an API endpoint** | `backend/app/api/v1/{auth,places,itineraries}.py` | `backend/app/schemas/*.py`, `backend/app/main.py` |
| **Modify itinerary generation logic** | `backend/app/algorithms/itinerary_generator.py` | `Project/src/app/core/services/itinerary.service.ts` & `desktop_client/services/itinerary_service.py` (offline fallbacks) |
| **Add a database table or column** | `backend/app/models/entities.py` | `backend/alembic/versions/` (`alembic revision --autogenerate`) |
| **Modify authentication or JWT rules** | `backend/app/services/auth_service.py` | `backend/app/api/v1/auth.py`, `auth.interceptor.ts`, `desktop_client/core/api_client.py` |
| **Integrate a new external API** | `backend/app/services/google_maps_service.py` | `backend/app/core/config.py`, `geospatial_cache` table |
| **Update offline POI data or categories** | `scripts/import_osm.py`, `backend/app/api/v1/places.py` (`CANONICAL_CATEGORIES`) | `scripts/export_osm_to_client_asset.py`, `osm_places.json` (Web + Desktop), `place.service.ts` |
| **Debug offline/tunnel connectivity** | `cloudflare-worker/worker.js`, `ConnectivityService` (Web + Desktop) | `scripts/start_server.ps1`, `environment.ts` |

---

## 2. Observed Coding Conventions & Recommended Workflow

### Verified Codebase Conventions
- **Backend (Python 3.12)**: Type-annotated SQLAlchemy 2.0 `Mapped[...]` models, Pydantic v2 `BaseModel` schemas with `ConfigDict(from_attributes=True)`, `ruff` and `mypy` listed in `backend/requirements.txt`, `pytest` for unit/integration tests.
- **Frontend (TypeScript / Angular 22)**: Standalone components (`standalone: true`), `inject()` dependency injection, `provideZonelessChangeDetection()` (explicit `ChangeDetectorRef.detectChanges()`), strict ESLint (`--max-warnings 0`), Prettier (`@ionic/prettier-config`), Vitest unit tests alongside every service/component (`*.spec.ts`).
- **Desktop (Python Flet)**: Modular separation into `config/`, `core/`, `models/`, `services/`, `components/`, `views/`, and `tests/`, mirroring Angular service/model contracts 1:1.

### Recommended Git & PR Workflow *(Recommendation — No formal team policy committed in repo)*
1. Create feature branches from `main` using descriptive prefixes (`feat/<topic>`, `fix/<issue>`, `docs/<topic>`).
2. Keep changes synchronized across Backend + Web (`Project/`) + Desktop (`desktop_client/`) when changing API contracts.
3. Run all local gates before opening a PR:
   ```bash
   python -m pytest backend/tests/ -v
   python -m pytest desktop_client/tests/ -v
   cd Project && npm run lint && npm run test:unit
   ```
4. Tag releases as `vX.Y.Z` to trigger `.github/workflows/release-builds.yml`.

---

## 3. End-to-End Walkthrough: Adding a New Feature (Example: Cloud-Synced Favorites)

Currently, Favorites are stored locally only (`localStorage` on Web, `storage/favorites.json` on Desktop). Here is how a developer would implement cloud-synced favorites consistently with the existing architecture:

1. **Define Scope**: Allow authenticated users to persist favorite `Place` UUIDs in PostgreSQL while keeping local fallback for guests/offline mode.
2. **Database Model (`backend/app/models/entities.py`)**:
   - Add a `UserFavorite(Base)` model with `id` (UUID PK), `user_id` (FK `users.id`, `ON DELETE CASCADE`), `place_id` (FK `places.id`, `ON DELETE CASCADE`), `created_at`, and `UniqueConstraint("user_id", "place_id", name="uq_user_favorite_place")`.
3. **Create Alembic Migration**:
   ```bash
   cd backend && alembic revision --autogenerate -m "add_user_favorites_table"
   alembic upgrade head
   ```
4. **Pydantic Schemas (`backend/app/schemas/place.py`)**: Add `FavoriteCreateIn(place_id: uuid.UUID)` and reuse `PlaceOut` for responses.
5. **Backend Router (`backend/app/api/v1/places.py` or new `favorites.py` mounted in `main.py`)**:
   - `GET /api/v1/favorites` (Bearer JWT, returns `list[PlaceOut]`).
   - `POST /api/v1/favorites` (Bearer JWT, idempotent insert, returns `201`).
   - `DELETE /api/v1/favorites/{place_id}` (Bearer JWT, BOLA-scoped to `current_user.id`, returns `204`).
6. **Web Frontend (`Project/src/app/core/services/favorites.service.ts`)**:
   - Inject `HttpClient`, `AuthService`, and `ConnectivityService`.
   - When authenticated and online, sync `addFavorite`/`removeFavorite` to `/api/v1/favorites` while retaining `localStorage` as offline cache.
7. **Desktop Client (`desktop_client/services/favorites_service.py`)**:
   - Inject `ApiClient` and `SessionManager`, mirroring the same online sync + `favorites.json` fallback.
8. **Automated Tests**:
   - Add BOLA & CRUD tests in `backend/tests/test_favorites.py`.
   - Update `Project/src/app/core/services/favorites.service.spec.ts` and `desktop_client/tests/test_parity_services_and_components.py`.
9. **Update Documentation**: Update `06-API-Reference.md`, `07-Database-Documentation.md`, and `diagrams/database-erd.md`.
