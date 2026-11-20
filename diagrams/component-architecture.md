# Application Component Architecture Diagram

Internal module dependencies across the Ionic Angular Web/Android client (`Project/`), Python Flet Desktop client (`desktop_client/`), and FastAPI backend (`backend/`).

```mermaid
flowchart LR
    subgraph Angular["Ionic Angular Frontend (Project/src/app/)"]
        AppComp["AppComponent\nSplash + Offline Banner + Back Btn"]
        AuthPages["Auth Pages\nWelcome, Guest, Login, Signup"]
        TabPages["Tab Pages\nHome, Map, History, Profile"]
        ItinPages["Itinerary Pages\nSuggestedWizard, CustomBuilder, Detail"]
        SharedComp["Shared Components\nDestinationAutocomplete\nItineraryRouteMap\nPlaceDetailSheet"]
        CoreSvc["Core Services\nAuth, Place, Itinerary, Connectivity\nFavorites, GuestStorage, GuestMigration\nOsmOverpass, RouteTracking, SecureToken"]
        Interceptor["authInterceptor\nBearer + 401 Refresh Queue"]
        AppComp --> TabPages
        AppComp --> AuthPages
        TabPages --> SharedComp
        ItinPages --> SharedComp
        AuthPages --> CoreSvc
        TabPages --> CoreSvc
        ItinPages --> CoreSvc
        CoreSvc --> Interceptor
    end

    subgraph Desktop["Python Flet Desktop (desktop_client/)"]
        DMain["main.py\nRoute Dispatcher + TabBar + Splash"]
        DViews["11 Views\nWelcome, Login, Signup, Guest, Dashboard\nPlaces, SavedTrips, Profile, Wizard\nCustomBuilder, TripDetail"]
        DComp["11 Components\nSlippyMapCanvas, ItineraryRouteMap\nDestinationPicker, Dialogs, Cards"]
        DSvc["Services\nAuthService, PlaceService\nItineraryService, FavoritesService"]
        DCore["Core\nApiClient, ConnectivityService\nSessionManager, GuestStorage, Export"]
        DMain --> DViews
        DViews --> DComp
        DViews --> DSvc
        DSvc --> DCore
    end

    subgraph Backend["FastAPI Backend (backend/app/)"]
        BMain["main.py\nSlowAPI + CORS"]
        BRouters["Routers (api/v1/)\nauth.py, places.py, itineraries.py"]
        BSchemas["Pydantic v2 Schemas\nauth.py, place.py, itinerary.py"]
        BSvc["Services & Algorithms\nauth_service.py\ngoogle_maps_service.py\nitinerary_generator.py"]
        BModels["ORM Models\nentities.py (8 tables)\ndatabase.py"]
        BMain --> BRouters
        BRouters --> BSchemas
        BRouters --> BSvc
        BSvc --> BModels
    end

    Interceptor -->|HTTPS JSON| BMain
    DCore -->|HTTPS / HTTP JSON| BMain
```
