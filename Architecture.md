Clients : Tg bot + web
Games:
* poker need to do FSM
* horse race
* slots

Year  confirmation at User service
CrypotGraphic random(System.Security.Cryptography)
No real money just virtual credits

### Modules
```mermaid
flowchart TD
    subgraph Clients
        A1[TG Bot]
        A2[React Web UI]
    end

    subgraph API Layer
        APIGW[Gateway API]
        BFF_Web[Web.BFF]
        BFF_Tg[Tg.BFF]
        CB[Circuit Breaker]
        RL[Rate Limiting]
        AC_Proxy[Accounting Proxy]
    end

    subgraph AppHost .NET Aspire
        DB[(KurrentDB\nEvent Store + Audit)]
        Cache[(Redis\nCache + Pub/Sub)]
        Bus[[RedPanda]]
        Monitoring[Grafana + Prometheus]
        LedgerDB[(Postgres\nAccounting Ledger)]
    end

    subgraph Modules
        P[Poker.Module]
        S[Slots.Module]
        R[Racing.Module]
        U[User.Module]
        N[Notifications]
        A[Analytics]
        Audit[Audit Module]
        AC[Accounting.Module with Saga]
    end

    A1 --> BFF_Tg --> APIGW
    A2 --> BFF_Web --> APIGW
    APIGW --> CB --> P & S & R & U & AC_Proxy
    CB --> RL
    AC_Proxy --> AC

    P -->|ReserveFundsCommand| AC
    S -->|DebitFundsCommand| AC
    R -->|PayoutCommand| AC
    U -->|GetBalanceQuery| AC

    P -->|Events| Bus --> A & N & Audit & AC
    S -->|Events| Bus
    R -->|Events| Bus
    AC -->|TransactionEvents| Bus

    P --> Cache
    S --> Cache
    N --> Cache
    AC --> Cache

    Audit --> DB
    P --> DB
    U --> DB
    AC --> LedgerDB

    Monitoring -.-> P & S & R & Bus & AC
    
    %% Новые политики CB
    CB -->|Fallback| CB_Policy["Accounting CB Policy:
    - Timeout: 1s
    - Retries: 2
    - Break: 1m after 5 failures
    - Fallback: LocalBalanceCache"]
```

## Poker
Tables - SignalR
Need to improve
Event Sourcing + snapshoting for history of players actions
Need to handle network drops
RNG for slots service


```mermaid
flowchart TD
    subgraph .NET Aspire[AppHost]
        A[KurrentDB] -->|Event Sourcing| B[Poker.Module]
        C[Redis] -->|Cache/PubSub| B[Poker.Module]
        C --> D[Slots.Module]
        E[RedPanda] -->|Async Events| F[Analytics]
    end

    subgraph Casino.Api
        B[Poker.Module DDD, Event Sourcing, CQRS]
        D[Slots.Module CQRS, Redis Cache]
        G[Racing.Module Saga, Hangfire]
    end

    H[React Frontend] -->|HTTP/WS| Casino.Api
```

```mermaid
flowchart LR
    subgraph AppHost
        A[KurrentDB] -->|Event Store| B[Poker]
        C[Redis] -->|Pub/Sub| D[SignalR]
        E[RedPanda] -->|BetPlacedEvent| F[Analytics]
    end

    subgraph Services
        B -->|HTTP| G[React Frontend]
        D -->|WebSockets| G
    end
```

Pocker
```mermaid
sequenceDiagram
    participant Player
    participant Poker
    participant Accounting
    participant Cache
    
    Player->>Poker: JoinTable($10)
    Poker->>Accounting: ReserveFunds(txId, $10)
    Accounting->>Cache: LockBalance(user: -$10)
    Accounting-->>Poker: FundsReserved
    Poker->>Player: SeatAssigned
    
    alt Игра завершена
        Poker->>Accounting: SettleBet(txId, win: $15)
    else Таймаут
        Poker->>Accounting: CancelReservation(txId)
    end
```

```mermaid
flowchart LR
    A[Poker.Service] -->|BetPlacedEvent| K[RedPanda]
    K --> B[Analytics.Service]
    K --> C[Notifications.Service]
```

```mermaid
flowchart TD
    A[TableManager] --> B[HandSaga]
    B --> C[SeatReservation]
    C --> D[BetRoundProcessor]
    D --> E[CardDealerRNG]
    E --> F[PayoutSaga]
```


Racing

```mermaid
flowchart TD
    subgraph Racing.Module
        A[StartRace] --> B[AcceptBets 30 sec]
        B --> C[RunRace Hangfire]
        C --> D[CalculateWinners]
        D --> E[PayoutRewards]
        E -->|Compensation| F[CancelRace]
    end
```

```mermaid
flowchart TD
    Start --> AcceptBets -->|Debit| Accounting
    AcceptBets --> RaceFailed -->|Compensate| Accounting
    RaceCompleted -->|CreditWinners| Accounting
```
## Data Flow
```mermaid
sequenceDiagram
participant User
participant Web as Frontend (React)
participant API
participant Racing
participant Redis
participant RedPanda
participant Notif as Notifications

User->>API: POST /race/start
API->>Racing: StartRaceCommand
Racing->>Redis: Store Race State
Racing->>RedPanda: RaceStartedEvent

Note right of RedPanda: Analytics, Notifications подписаны

Racing->>Hangfire: Schedule RunRaceJob

Hangfire->>Racing: RunRaceCommand
Racing->>Redis: Update Live Positions
Racing->>SignalR: push position
loop every 200ms
    Redis->>SignalR: Position update
end

Racing->>RedPanda: RaceFinishedEvent
RedPanda->>Notif: UserWon → push

```

### Circuit breaker
```mermaid
flowchart TD
    subgraph Clients
        A1[TG Bot]
        A2[React Web UI]
    end

    subgraph API Layer
        APIGW[Gateway API]
        BFF_Web[Web.BFF]
        BFF_Tg[Tg.BFF]
        CB[Circuit Breaker]
        RL[Rate Limiting]
        AC_Proxy[Accounting Proxy]
    end

    subgraph AppHost .NET Aspire
        DB[(KurrentDB\nEvent Store + Audit)]
        Cache[(Redis\nCache + Pub/Sub)]
        Bus[[RedPanda]]
        Monitoring[Grafana + Prometheus]
        LedgerDB[(Postgres\nAccounting Ledger)]
    end

    subgraph Modules
        P[Poker.Module]
        S[Slots.Module]
        R[Racing.Module]
        U[User.Module]
        N[Notifications]
        A[Analytics]
        Audit[Audit Module]
        AC[Accounting.Module with Saga]
    end

    A1 --> BFF_Tg --> APIGW
    A2 --> BFF_Web --> APIGW
    APIGW --> CB --> P & S & R & U & AC_Proxy
    CB --> RL
    AC_Proxy --> AC

    P -->|ReserveFundsCommand| AC
    S -->|DebitFundsCommand| AC
    R -->|PayoutCommand| AC
    U -->|GetBalanceQuery| AC

    P -->|Events| Bus --> A & N & Audit & AC
    S -->|Events| Bus
    R -->|Events| Bus
    AC -->|TransactionEvents| Bus

    P --> Cache
    S --> Cache
    N --> Cache
    AC --> Cache

    Audit --> DB
    P --> DB
    U --> DB
    AC --> LedgerDB

    Monitoring -.-> P & S & R & Bus & AC
    
    %% Новые политики CB
    CB -->|Fallback| CB_Policy["Accounting CB Policy:
    - Timeout: 1s
    - Retries: 2
    - Break: 1m after 5 failures
    - Fallback: LocalBalanceCache"]
```

Limits
```mermaid
flowchart LR
    CB -->|Max 10 req/sec| AC
    AC -->|429 Too Many Requests| CB
```

### Accounting module
```mermaid
flowchart TD
    subgraph Accounting.Module
        A[Account Aggregate] --> B[Transaction Process Manager]
        B --> C[Double-Entry Bookkeeping]
        C --> D[Ledger Event Sourcing]
        D --> E[Balance Cache]
    end
```

```mermaid
stateDiagram-v2
    [*] --> ReserveInitiated
    ReserveInitiated --> FundsLocked
    FundsLocked --> AwaitCommitOrCancel
    AwaitCommitOrCancel --> Committed : Commit
    AwaitCommitOrCancel --> Cancelled : Timeout / Cancel
    Committed --> [*]
    Cancelled --> [*]

```
 +kafka topics 
```
accounting.transaction-events
accounting.saga-events
```
Every Event has:
- `transactionId`
- `userId`
- `amount`
- `status`
- `timestamp`
- `originModule` (Poker, Slots…)

Poker 
```mermaid
flowchart TD
    subgraph Poker Saga
        A[StartHand] --> B[ReserveFunds]
        B --> C[DealCards]
        C --> D[ProcessBets]
        D --> E[FinalPayout]
        E --> F[EndHand]
    end

    subgraph Poker FSM
        G[PreFlop] --> H[Flop]
        H --> I[Turn]
        I --> J[River]
        J --> K[Showdown]
    end

    B -->|"FundsReserved"| G
    F -->|"HandCompleted"| L[(EventStore)]
```
FSM <-> Saga at pocker
```mermaid
sequenceDiagram
    participant Saga
    participant FSM
    participant EventStore

    Saga->>FSM: "HandStarted" (PreFlop)
    FSM->>EventStore: "FlopDealt"
    EventStore->>Saga: "BettingPhaseStarted"
    Saga->>FSM: "PlayerCalled"
    FSM->>Saga: "BettingPhaseEnded"
    Saga->>FSM: "TurnDealt"
```

Saga for bet FSM for rules
```mermaid
flowchart TD
    subgraph Улучшения Poker
        A[TableActor] --> B[HandSaga]
        B --> C[FSM]
        C --> D[RecoveryService]
        D -->|"Checkpoint every N events"| E[SnapshotStorage]
    end
```


### Racing module
```mermaid
flowchart TD
    A[RaceCoordinator] --> B[MarketMaker]
    B --> C[OddsCalculator]
    C --> D[PositionEngine]
    D --> E[ResultValidator]
```
