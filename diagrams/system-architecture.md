# System Architecture Diagram

This diagram illustrates the verified multi-client, edge proxy, backend, database, and external API topology of **Trip Mo, Trip Ko (Nueva Ecija Travel Itinerary App)**.

```mermaid
flowchart TB
    subgraph Clients["Client Tier (Offline-Resilient)"]
        Web["Ionic Angular 22 Web/PWA\nGitHub Pages (edselrenz.github.io/Itinerary-App)\nNGSW + Leaflet 1.9.4 + 2,673 OSM POIs"]
        Android["Android APK (Capacitor 8.6)\nph.nuevaecijatravel.app\nKeystore + Capacitor Geolocation"]
        Desktop["Windows Desktop App (Flet 1.1.2)\nPyInstaller + Inno Setup 6\nSlippyMapCanvas + Local Tile & JSON Cache"]
    end

    subgraph Edge["Cloudflare Edge Proxy"]
        Worker["Cloudflare Worker\ncloudflare-worker/worker.js\nCORS + Failover + 24h Edge Cache"]
        KV[("Cloudflare KV\nTUNNEL_KV\nACTIVE_TUNNELS + EDGE_CACHE_*")]
        Worker <--> KV
    end

    subgraph HostTunnels["Host Tunnel Layer (start_server.ps1)"]
        CF["Cloudflare Quick Tunnel\n*.trycloudflare.com (Primary)"]
        Ngrok["ngrok Tunnel\n*.ngrok-free.dev (Failover)"]
    end

    subgraph Backend["FastAPI Backend (Uvicorn :8000)"]
        FastAPI["app/main.py\nSlowAPI (120/min) + CORS"]
        AuthAPI["/api/v1/auth\nArgon2id + HS256 JWT + Google OAuth"]
        PlacesAPI["/api/v1/places & /municipalities\n16 Categories + Photo Proxy"]
        ItinAPI["/api/v1/itineraries\nBOLA CRUD + Generator + Routes"]
        FastAPI --> AuthAPI
        FastAPI --> PlacesAPI
        FastAPI --> ItinAPI
    end

    subgraph Storage["Persistence Layer"]
        PG[("PostgreSQL 16\nDocker :5432 (Primary)")]
        SQLite[("SQLite 3\nnueva_ecija_travel.db (Auto-Fallback)")]
    end

    subgraph External["External APIs"]
        GoogleMaps["Google Places v1 & Routes v2\n(Optional: GOOGLE_MAPS_API_KEY)"]
        OSRM["OSRM Public Router\nrouter.project-osrm.org"]
        Overpass["OSM Overpass API\n(Client & ETL Scripts)"]
        OSMTiles["OSM Tile Server\ntile.openstreetmap.org"]
        GoogleAuth["Google OAuth2\nToken Verification"]
    end

    Web -->|HTTPS| Worker
    Android -->|HTTPS| Worker
    Desktop -->|HTTPS / Local HTTP| Worker
    Desktop -.->|Local Dev| FastAPI

    Worker --> CF
    Worker -.->|Failover| Ngrok
    CF --> FastAPI
    Ngrok --> FastAPI

    FastAPI --> PG
    FastAPI -.->|Timeout Fallback| SQLite

    PlacesAPI --> GoogleMaps
    ItinAPI --> GoogleMaps
    ItinAPI -->|Fallback 1| OSRM
    AuthAPI --> GoogleAuth
    Web --> Overpass
    Web --> OSMTiles
    Android --> OSMTiles
    Desktop --> OSMTiles
```
