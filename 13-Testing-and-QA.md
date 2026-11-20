# 13 — Testing and Quality Assurance

This document catalogs the **actual automated test suites** present in the repository and provides a **recommended expansion test plan** for production hardening.

---

## 1. Existing Testing Infrastructure (Verified)

The repository has substantial automated test coverage across **four distinct test layers** (136 Pytest functions + 32 Vitest spec files + Playwright E2E + 24-point acceptance gate):

### 1.1 Backend Pytest Suite (`backend/tests/` — 21 test functions)
- **Framework**: `pytest>=8.3.0` with FastAPI `TestClient` and in-memory SQLite (`StaticPool`).
- **Execution Command**:
  ```bash
  python -m pytest backend/tests/ -v
  ```
- **CI Integration**: Executed automatically on every push to `main` in `.github/workflows/deploy.yml`.
- **Verified Modules**:
  - `test_auth.py` (6 tests): Full auth lifecycle, rate limiting, Cloudflare Tunnel IP resolution, IP-isolated rate limits, production JWT secret validation, Google OAuth security hardening.
  - `test_itineraries.py` (6 tests): Itinerary CRUD + BOLA protection, suggested itinerary engine, single-stop PATCH & place validation, radius/budget clustering, SQL bounding-box pre-filtering, chronological ordering & overnight lodging logic.
  - `test_google_maps_service.py` (7 tests): Polyline encode/decode, Haversine & duration helpers, Google type-to-category mapping, place normalization, DB cache integration, discover & route endpoints, Bayesian quality score & opening-hours gate.
  - `test_foundation.py` (1 test): Health endpoint & featured places pipeline.
  - `test_backup_restore.py` (1 test): End-to-end backup & restore drill using `tmp_path`.

### 1.2 Frontend Unit Tests (`Project/src/**/*.spec.ts` — 32 Vitest spec files)
- **Framework**: **Vitest `~5.0.5`** with `jsdom` (`Project/vitest.config.mts`, `fsModuleCache: false` to avoid stale Angular bundle caching).
- **Execution Commands** (from `Project/`):
  ```bash
  npm run test:unit        # ng test --no-watch (runs in CI)
  npm run test:unit:watch  # interactive watch mode
  npm run lint             # ESLint (0 warnings) + E2E tsc + Prettier check
  ```
- **Coverage**: Covers `AppComponent`, `pendingChangesGuard`, `authInterceptor`, 9 of 10 core services, 3 utilities (`date.utils`, `itinerary-export.util`, `polyline.util`), `commute-guide.data`, 3 auth pages, 3 itinerary pages, 3 shared components, and all 6 tab pages + `tabs.routes`. *(Missing spec: `guest.page.ts`, `guest-storage.service.ts`.)*

### 1.3 Frontend E2E Tests (`Project/e2e/app.spec.ts`)
- **Framework**: **Playwright `^1.64.1`** (`Project/playwright.config.ts`), configured for **Mobile Chrome (Pixel 7)** against `http://localhost:4200` (`webServer` auto-starts `npm start -- --port 4200`).
- **Execution Command** (from `Project/`):
  ```bash
  npm run test:e2e
  ```
- **Verified Scenarios**: Root redirect to `/#/welcome`, guest onboarding journey (`/#/welcome` → `/#/guest` → `/#/tabs/home`), Home ↔ Map tab navigation, Custom Itinerary Builder questionnaire verification.
- **Note**: Not currently executed in `.github/workflows/deploy.yml` (manual/local execution only).

### 1.4 Desktop Client Pytest Suite (`desktop_client/tests/` — 115 test functions)
- **Framework**: `pytest>=8.0.0` + `pytest-asyncio>=0.24.0`.
- **Execution Command**:
  ```bash
  python -m pytest desktop_client/tests/ -v
  ```
- **Verified Modules**: `test_api_client.py` (4), `test_auth_service.py` (6), `test_cycle2_regressions.py` (33), `test_itinerary_service.py` (3), `test_live_integration.py` (1), `test_parity_services_and_components.py` (42), `test_place_service.py` (4), `test_qa_remediation.py` (9), `test_storage_export.py` (4), `test_ui_consistency.py` (9).
- **Note**: Not currently executed in `.github/workflows/release-builds.yml`.

### 1.5 Automated 24-Point MVP Acceptance Gate (`scripts/verify_acceptance_gate.py`)
- **Execution Command**: `python scripts/verify_acceptance_gate.py`
- **Scope**: Runs 24 assertions across Auth & Guest Mode (7), Home & Places (3), Map Tab (7), and Itineraries & Operations (7) using in-memory SQLite + seeded curated places.

---

## 2. Recommended Test Plan (Gaps to Close)

Distinguished clearly from existing tests above, the following areas should be added before wide public release:

| Category | Recommended Test Cases | Target Layer |
|---|---|---|
| **1. Authentication & Security** | Expired refresh token rejection; password change invalidation check; `photo_name` path traversal rejection on `/places/photos/`; `avatar_url` oversized/malicious URI rejection; Cloudflare Worker `/__sync_tunnel` invalid key `401` test. | Backend Pytest + Worker Miniflare |
| **2. PostgreSQL Parity** | Run `backend/tests/` against a live PostgreSQL 16 service container in GitHub Actions (currently runs on SQLite in-memory) to catch PostgreSQL-specific `JSONB`, `UUID`, and `alembic upgrade head` regressions. | GitHub Actions CI |
| **3. Playwright Full-Stack E2E** | Add CI job running Playwright against live Uvicorn + Angular covering: Register → Create Custom Itinerary → Reorder Stops → Export `.ics` → Logout → Guest Create → Login & Verify Guest Migration. | `Project/e2e/` |
| **4. Desktop CI Gate** | Add `python -m pytest desktop_client/tests/ -v` step to `.github/workflows/release-builds.yml` before PyInstaller packaging. | `.github/workflows/release-builds.yml` |
| **5. Responsive & Accessibility QA** | Automated Lighthouse / axe-core checks on Web/PWA viewports (`375px`, `768px`, `1280px`) in light and dark modes. | Playwright + axe |
| **6. External Outage & Performance** | Load test `/api/v1/places?limit=5000` and `/api/v1/itineraries/suggest` (14-day trip) under simulated OSRM/Google timeouts; verify fallback latency stays `< 3s`. | Backend Pytest / Locust |
