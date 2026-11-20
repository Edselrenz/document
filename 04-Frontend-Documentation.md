# 04 — Frontend and User Interface Documentation

This document covers both client frontends implemented in the repository:
1. **Ionic Angular Web / PWA & Android Client** (`Project/`)
2. **Python Flet Windows Desktop Client** (`desktop_client/`)

---

## 1. Ionic Angular Frontend Architecture (`Project/`)

### 1.1 Application Initialization (`Project/src/main.ts` & `app.component.ts`)
- **Zoneless Standalone Angular 22.2.3**: Bootstrapped via `bootstrapApplication(AppComponent, { providers: [...] })` with `provideZonelessChangeDetection()`. Because Zone.js is omitted, pages explicitly inject `ChangeDetectorRef` / `ApplicationRef` and invoke `detectChanges()` (often via a helper `updateUI()`) after async state transitions.
- **Routing Strategy**: `provideRouter(routes, withHashLocation(), withPreloading(PreloadAllModules), withComponentInputBinding())` + `{ provide: RouteReuseStrategy, useClass: IonicRouteStrategy }`. Hash routing (`/#/tabs/home`) ensures deep links work on GitHub Pages alongside `404.html`.
- **Leaflet Global Guard**: `main.ts` intercepts writes to `window.L` via `Object.defineProperty` so module loads cannot wipe out `leaflet.markercluster` (`MarkerClusterGroup`).
- **PWA Service Worker**: `provideServiceWorker('ngsw-worker.js', { enabled: !isDevMode() && !Capacitor.isNativePlatform(), registrationStrategy: 'registerWhenStable:30000' })`. Caching rules in `Project/ngsw-config.json`:
  - `municipalities-api` (`/api/v1/municipalities`): `performance`, 7 days, max 100 entries.
  - `places-api` (`/api/v1/places`, `/api/v1/places/**`): `freshness`, 3 days, max 500 entries, 4s timeout.
  - `osm-tiles-nueva-ecija` (`https://*.tile.openstreetmap.org/**`): `performance`, 14 days, max 1,500 tiles.
- **Root Shell (`AppComponent`)**:
  - Restores dark/light theme from `localStorage.getItem('ne_theme_mode')` (`document.body.classList.toggle('dark')`).
  - On native Android (`Capacitor.isNativePlatform()`), configures `StatusBar` colors (`#0084ff` light / `#0b0f19` dark) and binds the Android hardware `backButton` listener (dismisses top modal → navigates back → exits app on root routes).
  - Probes backend health via `connectivityService.checkHostHealth()`.
  - Displays a cold-start splash screen (`showSplashScreen`), auto-dismissing after `2800ms` (or immediately on tap, 25px swipe, or `Escape` key) and remembered per tab via `sessionStorage.setItem('ne_splash_seen_v1', 'true')`.
  - Renders a non-blocking floating offline host banner when `connectivityService.isHostOffline$` emits `true` (with Retry and Dismiss actions).

### 1.2 State Management, Data Fetching & API Communication
- **Reactive Service State**: Singleton services (`providedIn: 'root'`) expose `BehaviorSubject` / `Subject` streams:
  - `AuthService` (`auth.service.ts`): `currentUser$`, JWT expiration checks (`isTokenExpired()` with 5s buffer), login/register/refresh/profile update.
  - `SecureTokenStorageService` (`secure-token-storage.service.ts`): Uses `capacitor-secure-storage-plugin` on native Android and `localStorage`/`sessionStorage` on web, backed by synchronous memory cache for the interceptor.
  - `ConnectivityService` (`connectivity.service.ts`): Tracks `consecutiveErrors` (threshold `ERROR_THRESHOLD = 2` before showing offline banner), manual dismissal flag, custom API URL validation (`ne_custom_api_url` restricted to trusted hostnames), and `/api/v1/health` probing.
  - `PlaceService` (`place.service.ts`): Merges API places into an in-memory 2,673-POI OSM cache (`osmPlacesCache`), resolves curated local photography (`localImageMap`), and formats the 16 canonical categories.
  - `ItineraryService` (`itinerary.service.ts`, 1,351 lines): Handles backend CRUD, active unsaved draft caching (`sessionStorage` key `ne_active_draft_itinerary`), guest storage fallback, and a full client-side offline clustering generator.
  - `GuestStorageService` (`guest-storage.service.ts`): Wraps `@ionic/storage-angular` (`ne_guest_itineraries_v1`, `ne_is_guest_mode_v1`).
  - `GuestMigrationService` (`guest-migration.service.ts`): Sequentially POSTs local guest itineraries to `/api/v1/itineraries` after login/signup.
  - `FavoritesService` (`favorites.service.ts`): Persists favorites in `localStorage` (`ne_favorite_places_v1`), seeding 3 iconic Nueva Ecija venues on first visit.
  - `OsmOverpassService` (`osm-overpass.service.ts`): Queries live Overpass endpoints (`overpass-api.de`, `maps.mail.ru`) for map viewport bounds with 12s timeout.
  - `RouteTrackingService` (`route-tracking.service.ts`): Uses `@capacitor/geolocation` (`watchPosition`) to snap GPS coordinates to the active day polyline, compute remaining distance/ETA, detect stop arrival (`50m` threshold), and trigger off-route alerts (`100m` threshold).
- **HTTP Interceptor (`Project/src/app/core/interceptors/auth.interceptor.ts`)**:
  - Attaches `ngrok-skip-browser-warning: true` and `Authorization: Bearer <token>` strictly to requests matching `activeApiBaseUrl` or `environment.apiBaseUrl`.
  - Ignores client-aborted requests (`AbortError`); triggers `connectivityService.setHostOffline(true)` only on gateway/network errors (`status === 0, 502, 503, 504`, or `404` on `/health`).
  - Implements a `BehaviorSubject` refresh queue on HTTP `401` (excluding `/auth/login`, `/auth/register`, `/auth/refresh`) so concurrent requests wait for one `/auth/refresh` call instead of logging out prematurely.

### 1.3 Design System & Styling (`Project/src/theme/variables.scss` & `global.scss`)
- **Typography**: `Inter` sans-serif stack.
- **Palette Tokens**:
  - Primary Azure Blue: `#2563eb` (Ionic) / `#0084ff` (brand highlights)
  - Secondary Deep Slate: `#0f172a`
  - Tertiary Scenic Teal: `#0d9488`
  - Success Emerald: `#16a34a`, Warning Amber: `#f59e0b`, Danger Red: `#dc2626`
  - Light Surface: `#f8fafc` background / `#ffffff` cards / `#e2e8f0` borders
  - Dark Surface: `#0b0f19` background / `#151e2e` cards (toggled via `body.dark`)

---

## 2. Complete Page, Screen, Tab & Shared Component Catalog (`Project/`)

| Page / Component | Route | Purpose | Key UI Elements & Inputs | Validation / Guards | API / Storage Triggered | Source Files |
|---|---|---|---|---|---|---|
| **WelcomePage** | `/welcome` (default) | Landing/onboarding screen with brand hero (`Trip mo, trip ko`) | Hero banner, `Log in`, `Sign up`, `Continue as Guest` pill button | Auto-redirects if already authenticated (unless `?loggedOut=true`) | `AuthService` | `features/auth/welcome/welcome.page.*` |
| **GuestPage** | `/guest` | Explains local-only Guest Mode before entering tabs | Explanation copy, `Continue as Guest`, `Create Account`, Back | None | `GuestStorageService.setGuestMode(true)` | `features/auth/guest/guest.page.*` |
| **LoginPage** | `/login` | Email/password sign-in, social buttons (disabled), Forgot Password modal | `email` (required, email), `password` (required), `rememberMe` toggle, password visibility toggle, Forgot Password modal (`recoveryEmail`) | Angular Reactive `FormGroup`; Forgot Password validates email regex client-side only (UI-simulated) | `POST /api/v1/auth/login`, `GuestMigrationService.migrateGuestData()` | `features/auth/login/login.page.*` |
| **SignupPage** | `/signup` | New traveler registration | `firstName`, `lastName`, `username` (3–50 chars, `^[a-zA-Z0-9_.-]+$`, auto-suggested), `email`, `password` (min 8), `confirmPassword` (match) | Reactive `FormGroup` + confirm match check | `POST /api/v1/auth/register`, `GuestMigrationService.migrateGuestData()` | `features/auth/signup/signup.page.*` |
| **TabsPage** | `/tabs` | Persistent 4-tab bottom navigation shell | `Home`, `Map`, `Saved` (`history`), `Profile` tabs | Redirects `/tabs/favorites` → `/tabs/profile?view=favorites`, `/tabs/saved` → `/tabs/history` | None | `tabs/tabs.page.*`, `tabs/tabs.routes.ts` |
| **HomePage** | `/tabs/home` | Discovery hub & itinerary launcher | Global search bar (debounced), Notification bell drawer, `Suggested Itinerary` & `Custom Itinerary` cards, Popular Destinations carousel & "See All" catalog modal, Municipality quick filters | None | `GET /api/v1/places`, `ItineraryService`, `GuestStorageService` | `tabs/home/home.page.*` |
| **MapPage** | `/tabs/map` | Full-bleed Leaflet interactive map & route navigation | `#leaflet-map` with `MarkerClusterGroup`, top search bar, horizontal 16-category pills, filter bottom sheet (categories, municipality, price tier), Live OSM Overpass fetch, GPS tracking HUD, Place Detail Sheet | Bounds & filter sync via URL query params | `GET /api/v1/places`, `GET /api/v1/municipalities`, `OsmOverpassService`, `RouteTrackingService` | `tabs/map/map.page.*` |
| **HistoryPage** | `/tabs/history` (alias `/saved`) | Saved itineraries manager | Segment filter (`All`, `Suggested`, `Custom`), search bar, trip cards (days/nights badge, stop count, honest cost range), delete confirmation | None | `GET /api/v1/itineraries`, `DELETE /api/v1/itineraries/{id}`, `GuestStorageService` | `tabs/history/history.page.*` |
| **ProfilePage** | `/tabs/profile` | Account hub with in-page subviews | Main menu, Edit Profile subview (`first_name`, `middle_initial`, `last_name`, `username`, `phone_number`, `birthday`, `gender`, `avatar` upload, `default_daily_budget`, `current_password`, `new_password`), Settings (Dark Mode, custom API URL), Favorites subview, Guest sync trigger, Help & About | Reactive form (`first_name`/`last_name` max 80, `username` 3–50, `new_password` 8–128) | `GET/PATCH /api/v1/auth/me`, `FavoritesService`, `GuestMigrationService`, `ConnectivityService` | `tabs/profile/profile.page.*` |
| **FavoritesPage** | `/tabs/favorites` (legacy redirect) | Standalone favorites list (now routed into Profile subview) | Search, category filter, favorite cards, remove action | Redirected by `tabs.routes.ts` to `/tabs/profile?view=favorites` | `FavoritesService` | `tabs/favorites/favorites.page.*` |
| **SuggestedWizardPage** | `/suggested-wizard` | Automated multi-day itinerary generator & draft preview | `DestinationAutocompleteComponent`, date range picker with duration presets, budget slider/presets, preference pills, travel mode (`DRIVE`/`WALK`), min rating, generated day timeline, nearby suggestions, commute guide | `end_date >= start_date`, max 14 days | `POST /api/v1/itineraries/suggest`, `POST /api/v1/itineraries` (on Save), `ItineraryService` offline fallback | `features/itinerary/suggested-wizard/suggested-wizard.page.*` |
| **CustomBuilderPage** | `/custom-builder` & `/custom-builder/:id` | Step-by-step custom itinerary authoring & editing | Step 1 questionnaire (title, destination, dates, budget, preferences) → Step 2 day-by-day stop builder (search/add POIs, add custom off-grid stop, edit time/duration/cost/notes, reorder, auto-schedule, collapsible route map) | `pendingChangesGuard` (`canDeactivate` prompts before discarding unsaved edits) | `POST /api/v1/itineraries`, `PATCH /api/v1/itineraries/{id}`, `POST /api/v1/itineraries/compute-day-route`, `GuestStorageService` | `features/itinerary/custom-builder/custom-builder.page.*` |
| **ItineraryDetailPage** | `/itinerary/:id` | Full trip viewer, in-place editor, live nav launcher & exporter | Day tabs, `ItineraryRouteMapComponent`, chronological stop cards, budget summary, assumptions accordion, commute guide, Edit Mode (add/remove/reorder days & stops), Export (`.ics`, copy text, Web Share, print) | `pendingChangesGuard` in edit mode | `GET/PATCH/DELETE /api/v1/itineraries/{id}`, `POST .../compute-route`, `itinerary-export.util.ts` | `features/itinerary/itinerary-detail/itinerary-detail.page.*` |
| **DestinationAutocompleteComponent** | Shared | Smart destination input with 32-town regional modal | Live text suggestions, quick chips, "All 32 Towns" modal grouped by congressional district | None | `PlaceService` | `shared/components/destination-autocomplete/*` |
| **ItineraryRouteMapComponent** | Shared | Collapsible Leaflet route map card embedded in wizard/builder/detail | Expandable map, numbered stop pins, OSRM/encoded polyline, "Open Full Route on Map" | None | `polyline.util.ts`, OSRM | `shared/components/itinerary-route-map/*` |
| **PlaceDetailSheetComponent** | Shared | Bottom sheet / modal showing full POI metadata | Hero photo gallery, category badge, verified cost, opening hours toggle, reviews, favorite toggle, Google Maps directions, "Start Trip Here", "Add to Custom Trip" | None | `PlaceService`, `FavoritesService` | `shared/components/place-detail-sheet/*` |

---

## 3. Python Flet Desktop Client Architecture (`desktop_client/`)

The desktop client (`desktop_client/main.py`) achieves 1:1 workflow and design parity with the Ionic Web client:
- **Route Dispatcher**: `navigate_to(route)` handles `/welcome`, `/guest`, `/login`, `/signup`, `/home` (`/dashboard`), `/map` (`/places`), `/saved` (`/trips`), `/profile`, `/suggested`, `/builder`, and `/trips/<id>`.
- **11 Views (`desktop_client/views/`)**: `welcome_view.py`, `guest_view.py`, `login_view.py`, `signup_view.py`, `dashboard_view.py`, `places_view.py`, `saved_trips_view.py`, `profile_view.py`, `suggested_wizard_view.py`, `custom_builder_view.py`, `trip_detail_view.py`.
- **11 Components (`desktop_client/components/`)**: `bottom_tab_bar.py`, `workflow_header.py`, `offline_banner.py`, `splash_overlay.py`, `slippy_map_canvas.py` (1,618 lines — custom pan/zoom tile renderer with bundled tiles + disk tile cache + spatial index + OSRM polylines), `itinerary_route_map.py`, `destination_picker.py`, `date_range_dialog.py`, `place_card.py`, `stop_card.py`, `dialogs.py`.
- **Local Persistence (`desktop_client/storage/`)**: `session.json` (JWT + user), `guest_storage.json` (guest itineraries), `favorites.json` (favorite POIs), `profile_extra.json` (extended profile fields), `connectivity_settings.json` (custom API URL), `avatars/`, and `tile_cache/`.

---

## 4. Key User Interface Workflows

```mermaid
flowchart LR
    Start([App Launch]) --> Splash[2.8s Splash Overlay\n+ Health Probe]
    Splash --> Welcome[/welcome]
    Welcome -->|Log in| Login[/login]
    Welcome -->|Sign up| Signup[/signup]
    Welcome -->|Continue as Guest| Guest[/guest]
    Login -->|Success + Migrate Guest Trips| Home[/tabs/home]
    Signup -->|Success + Migrate Guest Trips| Home
    Guest -->|Set Guest Mode| Home
    Home -->|Suggested Card| Wizard[/suggested-wizard]
    Home -->|Custom Card| Builder[/custom-builder]
    Home -->|Explore Map| Map[/tabs/map]
    Wizard -->|Save / Preview| Detail[/itinerary/:id]
    Builder -->|Save| Detail
    Detail -->|Launch Nav| Map
    Detail -->|Export .ics / Text| Export([Calendar / Clipboard])
```

---

## 5. Frontend Developer Guide (`Project/`)

### Adding a New Page & Route
1. Create a standalone component under `Project/src/app/features/<domain>/<page-name>/` with `.ts`, `.html`, `.scss`, and `.spec.ts`.
2. Register a lazy route in `Project/src/app/app.routes.ts` (or inside `tabs.routes.ts` if it belongs in the tab bar):
   ```ts
   {
     path: 'my-feature',
     loadComponent: () =>
       import('./features/my-domain/my-feature/my-feature.page').then((m) => m.MyFeaturePage),
   }
   ```
3. Because `provideZonelessChangeDetection()` is enabled, inject `ChangeDetectorRef` and call `this.cdr.detectChanges()` after async HTTP callbacks or timers if not using Angular signals / `async` pipe.

### Connecting a Page to the Backend API
- Never hardcode URLs. Inject `ConnectivityService` and resolve `${this.connectivityService.activeApiBaseUrl}/<resource>`.
- Use `HttpClient`; `authInterceptor` automatically attaches `Authorization: Bearer <token>` and `ngrok-skip-browser-warning: true` and handles 401 refresh queuing.
- Always implement an offline fallback or graceful error state (`catchError`) so rural/offline users retain access to cached `PlaceService` or `GuestStorageService` data.

### Adding Unsaved-Changes Protection
- Implement `ComponentCanDeactivate` (`canDeactivate(): boolean | Promise<boolean>`) and attach `canDeactivate: [pendingChangesGuard]` in `app.routes.ts`.

### Debugging Frontend Problems
- Run `npm start` inside `Project/` (serves on `http://localhost:4200` or `ionic serve` on `:8100`).
- Inspect `localStorage` keys: `ne_access_token`, `ne_refresh_token`, `ne_user_profile`, `ne_custom_api_url`, `ne_theme_mode`, `ne_favorite_places_v1`.
- Inspect `sessionStorage` keys: `ne_splash_seen_v1`, `ne_active_draft_itinerary`.
- Run `npm run test:unit` (Vitest) and `npm run lint` (ESLint + Prettier + E2E typecheck) before committing.
