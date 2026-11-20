# 08 — Business Logic and Core System Functionality

This document explains every core domain process and algorithm implemented in `Edselrenz/Itinerary-App`, traced directly from the source code.

---

## 1. Deterministic Suggested Itinerary Generation (`backend/app/algorithms/itinerary_generator.py`)

### Purpose & Trigger
- **Trigger**: `POST /api/v1/itineraries/suggest` (invoked by `SuggestedWizardPage` on Web/Android and `SuggestedWizardView` on Desktop).
- **Inputs**: `destination`, `start_date`, `end_date` (max 14 days), `daily_budget` (PHP), `preferences[]`, `travel_mode` (`DRIVE` | `WALK`), `pacing`, `include_live_places`, optional `anchor_place_id` / `anchor_place_payload`, `save_immediately`.

### Step-by-Step Processing Pipeline
1. **Date Expansion & Guard**: Builds inclusive `travel_dates[]` from `start_date` to `end_date`. Raises `ValueError` (HTTP `400`) if `total_days > 14`.
2. **Geographic Anchor Resolution (`resolve_geographic_anchor`)**:
   - Priority 1: Explicit `anchor_place_id` or `anchor_place_payload` (if provided, pins this POI as Day 1, Stop 1).
   - Priority 2: Case-insensitive `Municipality` match by `name` or `slug`.
   - Priority 3: `Place` name match.
   - Priority 4: Default provincial center `(15.5800, 121.0000)` (`"Nueva Ecija"`).
3. **Preference Expansion (`resolve_preference_categories`)**: Maps user keywords (e.g., `nature` → `nature_park`, `tourist_attraction`; `heritage` → `church_religious`, `museum_cultural`; `food` → `restaurant`, `cafe`; `relaxation` → `resort_swimming`, `hotel_accommodation`).
4. **SQL Bounding-Box Pre-Filtering & Live Discovery Fast-Path**:
   - Computes a coordinate bounding box around `(anchor_lat, anchor_lng)` using `cluster_radius_km = settings.default_dest_radius_km` (`25.0 km`) + `10.0 km` buffer, then refines via `haversine_distance_km()`.
   - Only calls `google_maps_service.search_places_nearby()` when `GOOGLE_MAPS_API_KEY` is configured AND either `include_live_places=True` with `< max(total_days * 3, 15)` local POIs, or `< total_days * 3` total local candidates exist.
   - Falls back to all active provincial POIs if the bounding box yields insufficient candidates.
5. **Cross-Day Fuzzy Deduplication (`is_place_duplicate`)**:
   - Rejects candidates matching any already-scheduled `place.id`, any normalized venue name (`normalize_venue_name()` strips generic suffixes like `resort`, `cafe`, `viewdeck`, `park`), OR any coordinate within `0.2 km` (`200m`) of a scheduled stop.
6. **Multi-Factor Scoring (`score_place`)** — Targets 3 stops per day (`09:00` Morning Activity, `12:00` Lunch Dining, `14:30` Afternoon Activity):

| Scoring Factor | Rule / Formula | Points |
|---|---|---|
| **Base Slot Score** | Dining slot (`lunch_dining`): `restaurant`/`cafe` = `80`, others = `10`; Non-dining slot = `50` | `+10` to `+80` |
| **Preference Match** | `place.category in preferred_categories` (`+40`); matching tag (`+15`) | up to `+55` |
| **Featured Quality** | `place.is_featured == True` | `+20` |
| **Bayesian Rating** | `((v/(v+15))*r + (15/(v+15))*3.8) / 5.0 * 20.0` (prevents 1-review 5-star bias) | up to `+20` |
| **Proximity Decay** | Within `25 km`: `max(0, 30 - dist * 1.2)`; Beyond `25 km`: `-(dist - 25) * 2.0` | `+30` down to negative |
| **Opening-Hours Gate** | `is_open_during_slot()` checks Google `regular_opening_hours.periods` or OSM `HH:MM-HH:MM` | `-50` if closed |
| **Category Diversity** | `place.category` already scheduled today | `-25` |
| **Cumulative Budget Fit** | Stop cost `<= remaining_daily_budget` (`+10`); exceeds remaining budget (`-30`) | `-30` to `+10` |
| **Mountain Peak Guard** | `day_has_mountain` and name matches `\b(mt\.?|mount)\b` (max 1 strenuous hike/day) | `-60` |
| **Overnight Lodging Guard** | Single-day trip: `-100`; Multi-day first slot: `-40`; Multi-day later slot: `+20` | `-100` to `+20` |

7. **2-Opt TSP Permutation Ordering & Overnight Placement**:
   - Separates overnight accommodations (`is_accommodation_place`) to the end of the day.
   - Evaluates all permutations (`itertools.permutations`) of non-accommodation stops (keeping Day 1's `anchor_place` fixed at index 0) to minimize total Haversine travel distance.
8. **Dynamic Road Pacing & Route Computation**:
   - Assigns realistic stay durations: `120 min` (lodging), `75 min` (dining), `90 min` (activities).
   - Starts Day at `09:00`, invokes `google_maps_service.compute_route()` between consecutive stops, falls back to `straight_km * 1.28` road factor at `32 km/h` if route distance is `0`, and rounds arrival times to the nearest 5-minute increment.
   - Computes full-day `day_overview_polyline`, `total_distance_meters`, `total_travel_duration_seconds`, and `route_legs`.
9. **Transparent Assumptions**: Appends pacing, geographic clustering, transit-exclusion, operating-hours, and honest unverified-pricing (`unverified_cost_count` stops not counted as `₱0`) notices.

---

## 2. Multi-Tier Turn-by-Turn Routing & Caching (`backend/app/services/google_maps_service.py`)

- **Cache Key**: `routes:{mode}:{orig_lat:.4f},{orig_lng:.4f}:{dest_lat:.4f},{dest_lng:.4f}:{intermediates}` checked against `geospatial_cache` (`TTL = 336 hours` / 14 days).
- **Tier 1 — Google Routes API v2**: Called when `GOOGLE_MAPS_API_KEY` is set (`TRAFFIC_AWARE` for `DRIVE`), returning `source: "google_routes"`.
- **Tier 2 — OSRM Public Router**: Called at `router.project-osrm.org/route/v1/driving/{coords}` with `3.0s` timeout, returning `source: "osrm"`.
- **Tier 3 — Offline Haversine Direct**: Computes straight-line legs, encodes a Google precision-5 polyline via `encode_polyline()`, and estimates duration at `11.11 m/s` (`40 km/h` drive) or `1.25 m/s` (`4.5 km/h` walk), returning `source: "haversine_direct"`.

---

## 3. Cross-Client Place Resolution & Upsert (`resolve_or_upsert_item_place`)

Because clients bundle 2,673 offline OSM POIs (some with non-UUID synthetic IDs like `dest-minalungao`) and support live Google Places discovery, `ItineraryItemCreate` and `resolve_or_upsert_item_place()` (`backend/app/api/v1/itineraries.py`):
1. Coerce non-UUID `place_id` strings into `place_payload["id"]` while setting `place_id = None`.
2. Look up existing `Place` by UUID, then by `google_place_id`, then by `source_record_id`, then by name + `±0.002°` coordinate tolerance.
3. If no match exists, upsert a new `Place` row (`upsert_google_place_to_db`), automatically assigning the closest `Municipality` via Haversine distance so foreign key integrity is always preserved.

---

## 4. Offline Resilience, Guest Mode & Account Migration

- **Offline Detection (`ConnectivityService`)**: Tracks consecutive gateway errors (`status === 0, 502, 503, 504`, or `404` on `/health`). Once `consecutiveErrors >= 2` (`ERROR_THRESHOLD`), displays the non-blocking offline host banner without interrupting local data access.
- **Offline Fallback Generation**: Both `ItineraryService` (Web) and `desktop_client/services/itinerary_service.py` (Desktop) contain local clustering fallbacks using the bundled 2,673-POI `osm_places.json` dataset when the backend is unreachable.
- **Guest-to-Account Migration (`GuestMigrationService`)**: Triggered automatically after login/registration (and manually from Profile). Reads all trips from `GuestStorageService` (`ne_guest_itineraries_v1`), normalizes `planned_time` via `toBackendIsoTime()`, POSTs each trip to `/api/v1/itineraries`, and removes migrated trips from local storage.

---

## 5. Live GPS Route Tracking (`Project/src/app/core/services/route-tracking.service.ts`)

- Uses `@capacitor/geolocation` (`watchPosition`) against the active day's decoded polyline (`decodePolyline()`).
- **Thresholds**: `stopArrivalThresholdMeters = 50m`, `offRouteThresholdMeters = 100m`, `movementThrottleMeters = 10m`, default speed `10 m/s` (`~36 km/h`).
- Emits `state$` (snapped position, distance & ETA to next stop, total remaining distance/duration, `completedStopIds`), `stopReached$`, and `rerouteNeeded$` observables.

---

## 6. Calendar & Text Export (`itinerary-export.util.ts` & `desktop_client/core/export.py`)

- Generates **RFC 5545 iCalendar (`.ics`)** files with `VEVENT` entries per stop (`DTSTART`, `DTEND` from `planned_time + duration_minutes`, `LOCATION`, `DESCRIPTION` with costs, notes, and Google Maps directions URL).
- Generates structured Markdown summaries, clipboard copy, Web Share API payloads, and printable views.
