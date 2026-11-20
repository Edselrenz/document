# Trip Mo, Trip Ko — Nueva Ecija Travel Itinerary System Documentation

Welcome to the primary technical documentation package for **Trip Mo, Trip Ko (Nueva Ecija Travel Itinerary App)** (`https://github.com/Edselrenz/Itinerary-App`).

This documentation was produced through a complete source-code audit of the repository (`backend/`, `Project/`, `desktop_client/`, `cloudflare-worker/`, `scripts/`, `installer/`, and `.github/workflows/`). Every architecture diagram, API contract, database column, file path, command, and security finding reflects the **actual verified implementation** rather than assumptions.

---

## Quick System Snapshot

| Attribute | Verified Implementation |
|---|---|
| **Application Name** | Trip Mo, Trip Ko — Nueva Ecija Travel Itinerary Planner |
| **Geographic Scope** | Nueva Ecija Province, Philippines (32 LGUs, 2,673 bundled OSM POIs + 18 curated featured venues) |
| **Web / PWA / Mobile Client** | Ionic Angular 9 + Angular 22.2.3 (zoneless standalone) + Capacitor 8.6.0 (Android APK) + Leaflet 1.9.4 (`Project/`) |
| **Desktop Client** | Python Flet 1.1.2+ + `flet-map` + `httpx` + PyInstaller + Inno Setup 6 (`desktop_client/`) |
| **Backend API** | FastAPI 0.115+ + Uvicorn + SQLAlchemy 2.0 + Alembic + SlowAPI (`backend/`) |
| **Database** | PostgreSQL 16 (primary, Docker Compose) with automatic local SQLite fallback (`nueva_ecija_travel.db`) |
| **Edge Proxy & Tunneling** | Cloudflare Worker (`cloudflare-worker/worker.js`) + KV (`TUNNEL_KV`) routing to Cloudflare Tunnel & ngrok |
| **External Services** | Google Places API (New), Google Routes API v2, OpenStreetMap Overpass API, OSRM Public Router, OSM Tile Server |
| **Authentication** | Email/Username + Argon2id (`pwdlib`) + HS256 JWT (30-min access / 7-day refresh) + Guest Mode |

---

## Documentation Index

### Core Architecture & Codebase
1. **[01 — System Overview](./01-System-Overview.md)** — Purpose, problems solved, objectives, technology stack, target users, roles, and verified scope/limitations.
2. **[02 — System Architecture](./02-System-Architecture.md)** — High-level, component, communication, auth, database, external API, and deployment architecture.
3. **[03 — Project Structure](./03-Project-Structure.md)** — Complete annotated repository tree and navigation map for new developers.

### Application Subsystems
4. **[04 — Frontend Documentation](./04-Frontend-Documentation.md)** — Ionic Angular web/mobile client (`Project/`) and Flet desktop client (`desktop_client/`) architecture, pages/screens, workflows, and developer guide.
5. **[05 — Backend Documentation](./05-Backend-Documentation.md)** — FastAPI entry point, middleware stack, rate limiting, services, routing lifecycle, and external API integrations.
6. **[06 — API Reference](./06-API-Reference.md)** — Complete REST API specification for all 20 backend endpoints plus Cloudflare Worker edge routes, with verified payloads and error codes.
7. **[07 — Database Documentation](./07-Database-Documentation.md)** — Full 8-table schema, Alembic migrations (`0001`–`0004`), indexes, relationships, ORM operations, and DBA guide.
8. **[08 — Business Logic](./08-Business-Logic.md)** — Deep dive into the deterministic itinerary generator (Haversine clustering, Bayesian quality scoring, opening-hours gate, 2-Opt TSP), routing fallbacks, offline sync, and guest migration.

### Security, Operations & Lifecycle
9. **[09 — Authentication and Security](./09-Authentication-and-Security.md)** — Password hashing, JWT lifecycle, BOLA enforcement, rate limiting, CORS, secure storage, and confirmed security assessment.
10. **[10 — Installation and Setup](./10-Installation-and-Setup.md)** — End-to-end local environment setup for Backend, PostgreSQL/SQLite, Web/PWA, Android, Desktop, and Cloudflare Worker.
11. **[11 — System Administration](./11-System-Administration.md)** — Service startup/shutdown, dual-tunnel management, backups/restores, OSM catalog refresh, and maintenance procedures.
12. **[12 — Troubleshooting](./12-Troubleshooting.md)** — Structured diagnostic table covering frontend, desktop, backend, database, tunnels, CORS, and build failures.
13. **[13 — Testing and QA](./13-Testing-and-QA.md)** — Verified inventory of existing automated tests (Pytest, Vitest, Playwright, 24-point acceptance gate) and recommended test plan.
14. **[14 — Deployment Guide](./14-Deployment-Guide.md)** — GitHub Pages CI/CD, GitHub Releases (Android APK & Windows Setup.exe), Docker Compose production stack, and Cloudflare Worker deployment.
15. **[15 — Developer Guide](./15-Developer-Guide.md)** — Codebase navigation matrix, conventions, and end-to-end feature addition walkthrough.
16. **[16 — Technical Debt and Roadmap](./16-Technical-Debt-and-Roadmap.md)** — Prioritized (`CRITICAL`, `HIGH`, `MEDIUM`, `LOW`) architectural and code-level remediation roadmap.

### Architecture & Workflow Diagrams (`diagrams/`)
- **[System Architecture](./diagrams/system-architecture.md)** — Multi-client, edge proxy, backend, database, and external API topology.
- **[Component Architecture](./diagrams/component-architecture.md)** — Internal module relationships across Angular, Flet, and FastAPI.
- **[Database ERD](./diagrams/database-erd.md)** — Verified 8-table entity-relationship diagram with cardinalities and foreign keys.
- **[Authentication Flow](./diagrams/authentication-flow.md)** — Registration, login, JWT refresh queue, guest mode, and guest-to-account migration.
- **[API Request Lifecycle](./diagrams/api-request-lifecycle.md)** — End-to-end request pipeline through Cloudflare Worker, FastAPI middleware, and SQLAlchemy.
- **[Core Workflows](./diagrams/core-workflows.md)** — Suggested itinerary generation, custom builder, route computation, and offline failover flows.

---

## Implementation Verification Legend

Throughout this documentation, implementation status is explicitly distinguished using five labels:
- **Implemented & Verified**: Present in repository source code and traced end-to-end.
- **Partially Implemented**: Backend or UI exists, but end-to-end integration is incomplete (e.g., Google OAuth backend exists while frontend buttons are disabled; Forgot Password modal is UI-simulated).
- **Referenced / Optional**: Supported via optional configuration (e.g., `GOOGLE_MAPS_API_KEY` live discovery/routing).
- **Planned / Disabled**: Explicitly gated off in code (e.g., Apple Sign-In returning HTTP `501`).
- **Missing / Undocumented**: Not present in the repository (e.g., RBAC/admin roles, automated email delivery, centralized log aggregation).
