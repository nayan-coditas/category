# Outbox Pattern Sync Flow Diagram

## Flow: Data Syncing When Machine Comes Online

```mermaid
sequenceDiagram
    participant Timer as Interval Timer<br/>(Every 2 seconds)
    participant OutboxService as OutboxService
    participant ConnectionService as ConnectionService
    participant Database as PostgreSQL<br/>(Outbox Actions)
    participant VesselService as VesselService
    participant VoyageService as VoyageService
    participant Redis as Redis Cache
    participant ExternalAPI as External API<br/>(api.stoltdev.com)
    participant WebSocket as EventsGateway<br/>(Socket.IO)
    participant Clients as WebSocket Clients

    Note over Timer,Clients: Machine Comes Online - Sync Flow Starts

    loop Every 2 seconds
        Timer->>OutboxService: startAutomaticProcessing()<br/>triggered
        OutboxService->>ConnectionService: getConnectionStatus()

        alt Connection Status = ONLINE
            ConnectionService-->>OutboxService: true (online)
            OutboxService->>Database: getPendingActions()<br/>WHERE status = 'PENDING'
            Database-->>OutboxService: [Action1, Action2, ...]

            alt Pending Actions Found
                OutboxService->>OutboxService: processAllPending()

                loop For each pending action
                    OutboxService->>Database: UPDATE status = 'PROCESSING'
                    OutboxService->>OutboxService: processAction(action)

                    alt Action Variant: VESSEL_CODES_FETCH
                        OutboxService->>VesselService: getVesselCodes()
                        VesselService->>Redis: getCachedVesselCodes()
                        Redis-->>VesselService: null (no cache)
                        VesselService->>ExternalAPI: GET /api/v1/vessels/vessel-codes
                        ExternalAPI-->>VesselService: Vessel Codes Data
                        VesselService->>Redis: cacheVesselCodes(data)
                        VesselService-->>OutboxService: Success with data

                    else Action Variant: VESSEL_TANK_LIST_FETCH
                        OutboxService->>VesselService: getVesselTankList(vesselCode)
                        VesselService->>Redis: getCachedVesselTankList(vesselCode)
                        Redis-->>VesselService: null (no cache)
                        VesselService->>ExternalAPI: GET /api/v1/vessels/tank-list<br/>?vesselCode={code}
                        ExternalAPI-->>VesselService: Tank List Data
                        VesselService->>Redis: cacheVesselTankList(vesselCode, data)
                        VesselService-->>OutboxService: Success with data

                    else Action Variant: VOYAGE_FAVORITE_UPDATE
                        OutboxService->>VoyageService: addOrRemoveVoyageFavorite(payload)
                        VoyageService->>ExternalAPI: POST /api/v1/voyages/favourite<br/>{vesselCode, voyageNo, isFavourite}
                        ExternalAPI-->>VoyageService: Success Response
                        VoyageService-->>OutboxService: Success

                    else Other Variants
                        OutboxService->>OutboxService: processShipStatusUpdate()<br/>processCargoUpdate()<br/>processVoyageUpdate()
                    end

                    alt Processing Successful
                        OutboxService->>Database: UPDATE status = 'COMPLETED'<br/>processedAt = now()
                        OutboxService->>WebSocket: broadcastEventsUpdate()
                        WebSocket->>Clients: eventsList (all actions)
                        WebSocket->>Clients: statsUpdate<br/>{total, pending, completed}
                    else Processing Failed
                        OutboxService->>Database: UPDATE status = 'FAILED'<br/>errorMessage = error.message
                        OutboxService->>WebSocket: broadcastEventsUpdate()
                        WebSocket->>Clients: eventsList (with failed action)
                    end
                end
            else No Pending Actions
                Note over OutboxService: No actions to process
            end

        else Connection Status = OFFLINE
            ConnectionService-->>OutboxService: false (offline)
            Note over OutboxService: Skip processing - waiting for connection
        end
    end
```

## State Diagram: Action Lifecycle

```mermaid
stateDiagram-v2
    [*] --> PENDING: Action Created
    PENDING --> PROCESSING: Auto-processing starts<br/>(when online)
    PROCESSING --> COMPLETED: External API call succeeds
    PROCESSING --> FAILED: External API call fails
    COMPLETED --> [*]: Action finished
    FAILED --> [*]: Action finished (with error)

    note right of PENDING
        Actions remain in PENDING
        while machine is offline
        or connection is lost
    end note

    note right of PROCESSING
        Status updated before
        calling external API
    end note
```

## Component Interaction Flow

```mermaid
graph TB
    subgraph "OutboxService Module"
        A[onModuleInit] --> B[startAutomaticProcessing]
        B --> C[setInterval every 2s]
        C --> D{Connection Status?}
    end

    subgraph "ConnectionService"
        D --> E[getConnectionStatus]
        E -->|Online| F[Return true]
        E -->|Offline| G[Return false]
    end

    subgraph "Database Layer"
        F --> H[getPendingActions]
        H --> I[(PostgreSQL<br/>outbox_actions)]
        I --> J[Return pending actions]
    end

    subgraph "Processing Logic"
        J --> K{Has pending<br/>actions?}
        K -->|Yes| L[processAllPending]
        K -->|No| C
        L --> M[For each action:<br/>processAction]
        M --> N[Update status:<br/>PENDING → PROCESSING]
    end

    subgraph "Service Layer"
        N --> O{Action Variant?}
        O -->|VESSEL_CODES_FETCH| P[VesselService<br/>getVesselCodes]
        O -->|VESSEL_TANK_LIST_FETCH| Q[VesselService<br/>getVesselTankList]
        O -->|VOYAGE_FAVORITE_UPDATE| R[VoyageService<br/>addOrRemoveVoyageFavorite]
    end

    subgraph "External Integration"
        P --> S[HTTP GET<br/>External API]
        Q --> T[HTTP GET<br/>External API]
        R --> U[HTTP POST<br/>External API]
        S --> V[Redis Cache<br/>Store results]
        T --> V
    end

    subgraph "Status Update"
        V --> W{Success?}
        U --> W
        W -->|Yes| X[Update status:<br/>PROCESSING → COMPLETED]
        W -->|No| Y[Update status:<br/>PROCESSING → FAILED]
    end

    subgraph "Real-time Updates"
        X --> Z[broadcastEventsUpdate]
        Y --> Z
        Z --> AA[EventsGateway<br/>broadcastEventsList]
        Z --> AB[EventsGateway<br/>broadcastStats]
        AA --> AC[WebSocket Clients]
        AB --> AC
    end

    X --> C
    Y --> C
    G --> C

    style A fill:#e1f5ff
    style B fill:#e1f5ff
    style F fill:#c8e6c9
    style G fill:#ffcdd2
    style I fill:#fff9c4
    style V fill:#fff9c4
    style S fill:#f3e5f5
    style T fill:#f3e5f5
    style U fill:#f3e5f5
    style X fill:#c8e6c9
    style Y fill:#ffcdd2
    style AC fill:#e1f5ff
```

## Key Points

1. **Automatic Processing**: The `startAutomaticProcessing()` method runs every 2 seconds via `setInterval`
2. **Connection Check**: Before processing, it checks `ConnectionService.getConnectionStatus()` to ensure machine is online
3. **Database Query**: Fetches all actions with `status = 'PENDING'` from PostgreSQL
4. **Sequential Processing**: Each action is processed one by one (not in parallel)
5. **Status Transitions**: Actions move through states: `PENDING` → `PROCESSING` → `COMPLETED`/`FAILED`
6. **External API Calls**: Based on action variant, calls appropriate service method which makes HTTP requests to external APIs
7. **Caching**: Successful API responses are cached in Redis for offline access
8. **Real-time Updates**: After each action is processed, WebSocket broadcasts update all connected clients
9. **Error Handling**: Failed actions are marked as `FAILED` with error message stored
10. **Offline Resilience**: When offline, processing is skipped but actions remain in `PENDING` state for when connection is restored
