
# VRP Framework Architecture

## High-Level Communication Flow

```mermaid
graph TD
    Client[Client (FiveM)]
    Server[Server (FiveM)]
    DB[(Database)]

    Client -- Tunnel (Network) --> Server
    Server -- Tunnel (Network) --> Client
    Server -- Proxy (Local) --> ServerResource[Other Server Resources]
    Client -- Proxy (Local) --> ClientResource[Other Client Resources]
    Server -- oxmysql/ghmattimysql --> DB
```

## User Identification Flow

```mermaid
sequenceDiagram
    participant Player
    participant Server
    participant DB

    Player->>Server: Connects (Steam/License)
    Server->>DB: Query User ID by Identifier
    alt Exists
        DB-->>Server: Returns user_id (e.g., 101)
    else New Player
        Server->>DB: Create User
        DB-->>Server: Returns new user_id
    end
    Server->>Player: Assign user_id to Source
    Server-->>Server: Triggers vRP:playerSpawned
```

## Core Module Relationships

```mermaid
classDiagram
    class vRP {
        +getUserId(source)
        +getUserSource(user_id)
        +tryPayment(user_id, amount)
        +giveInventoryItem(user_id, item, amount)
    }
    class vRPclient {
        +notify(msg)
        +teleport(x, y, z)
        +playAnim(upper, seq, looping)
    }
    class Tunnel {
        +bindInterface(name, table)
        +getInterface(name)
    }
    class Proxy {
        +addInterface(name, table)
        +getInterface(name)
    }

    vRP ..> Tunnel : Uses
    vRP ..> Proxy : Uses
```
