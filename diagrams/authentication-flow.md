# Authentication, Token Refresh, and Guest Migration Flow

Verified sequence across `LoginPage`/`SignupPage`, `authInterceptor`, `GuestMigrationService`, `backend/app/api/v1/auth.py`, and `backend/app/services/auth_service.py`.

```mermaid
sequenceDiagram
    autonumber
    actor User as Traveler
    participant Client as Web/Android or Desktop Client
    participant Storage as SecureTokenStorage / GuestStorage
    participant API as FastAPI (/api/v1/auth)
    participant DB as PostgreSQL / SQLite

    rect rgb(235, 245, 255)
        note right of User: 1. Login & Guest Trip Migration
        User->>Client: Submit email/username + password
        Client->>API: POST /api/v1/auth/login (30/min limit)
        API->>DB: SELECT user by normalized email/username
        API->>API: Verify Argon2id hash (pwdlib)
        alt Invalid credentials
            API-->>Client: 401 "Invalid email or password"
        else Valid & active
            API-->>Client: 200 TokenOut (30m access JWT + 7d refresh JWT + User)
            Client->>Storage: Persist tokens (Keystore / localStorage / session.json)
            Client->>Storage: Read local guest itineraries
            loop Each local guest itinerary
                Client->>API: POST /api/v1/itineraries (Bearer access JWT)
                API->>DB: INSERT itinerary, days, items
                API-->>Client: 201 Created
            end
            Client->>Storage: Clear migrated guest trips
            Client-->>User: Navigate to /tabs/home
        end
    end

    rect rgb(255, 248, 235)
        note right of User: 2. Transparent 401 Refresh Queue
        Client->>API: GET /api/v1/itineraries (expired access JWT)
        API-->>Client: 401 Unauthorized
        Client->>Client: Acquire refresh lock (BehaviorSubject / asyncio.Lock)
        Client->>API: POST /api/v1/auth/refresh (10/min limit)
        API->>API: decode_jwt_token(expected_type="refresh")
        API->>DB: Verify user exists & is_active
        alt Refresh valid
            API-->>Client: 200 Fresh TokenOut
            Client->>Storage: Update stored tokens
            Client->>API: Retry queued requests with new Bearer JWT
            API-->>Client: 200 OK
        else Refresh expired/invalid
            API-->>Client: 401 Unauthorized
            Client->>Storage: Clear tokens & logout
        end
    end
```
