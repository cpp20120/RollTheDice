Clients : Tg bot + web
Games:
* poker need to do FSM
* horse race
* slots
* black jack

Year  confirmation at User service
CrypotGraphic random(System.Security.Cryptography)
No real money just 
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
        BJ[Blackjack.Module]
        U[User.Module]
        N[Notifications]
        A[Analytics]
        Audit[Audit Module]
        AC[Accounting.Module with Saga]
    end

    A1 --> BFF_Tg --> APIGW
    A2 --> BFF_Web --> APIGW
    APIGW --> CB --> P & S & R & BJ & U & AC_Proxy
    CB --> RL
    AC_Proxy --> AC

    P -->|ReserveFundsCommand| AC
    S -->|DebitFundsCommand| AC
    BJ -->|SplitBetCommand| AC
    R -->|PayoutCommand| AC
    U -->|GetBalanceQuery| AC

    P -->|Events| Bus --> A & N & Audit & AC
    S -->|Events| Bus
    R -->|Events| Bus
    BJ -->|Event| Bus
    AC -->|TransactionEvents| Bus

    P --> Cache
    S --> Cache
    BJ --> Cache
    N --> Cache
    AC --> Cache

    Audit --> DB
    P --> DB
    BJ --> DB
    U --> DB
    AC --> LedgerDB

    Monitoring -.-> P & S & R & BJ & Bus & AC
    
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
	 **dead man's switch**


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
        B --> C[RunRace Kafca KSQL]
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

Blackcjack
```mermaid
flowchart TD
    subgraph Blackjack.Module
        A[TableManager] --> B[GameSession]
        B --> C[CardDealer]
        C --> D[BasicStrategyEngine]
        D --> E[PayoutCalculator]
        F[ShoeManager] --> C
        G[InsuranceService] --> B
    end
    
    B -->|Events| H[(EventStore)]
    E -->|Commands| Accounting
```


## Data Flow BlackJack
```mermaid
sequenceDiagram
    participant Player
    participant Blackjack
    participant Accounting
    participant RNG
    
    Player->>Blackjack: PlaceBet($10)
    Blackjack->>Accounting: ReserveFunds(txId, $10)
    Accounting-->>Blackjack: Reserved
    Blackjack->>RNG: GetShuffledDeck(6 decks)
    RNG-->>Blackjack: DeckId
    Blackjack->>Player: DealCards(Player: K♠,9♥ | Dealer: A♦)
    
    alt Player chooses hit
        Player->>Blackjack: Hit
        Blackjack->>RNG: DrawCard
        RNG-->>Blackjack: Q♣
        Blackjack->>Player: NewHand(K♠,9♥,Q♣)
    else Player stands
        Player->>Blackjack: Stand
        Blackjack->>Accounting: SettleBet(txId)
    end
```

## Data Flow Racing
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
flowchart LR
    A[Command] --> B{IdempotencyKey в Redis?}
    B -->|Нет| C[Обработать]
    B -->|Да| D[Вернуть прошлый результат]
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

Pocker FSM
```mermaid
stateDiagram-v2
    [*] --> WaitingForPlayers
    WaitingForPlayers --> PreFlop: min_players
    PreFlop --> Folded: Все кроме одного сфолдили
    PreFlop --> Flop: enough_calls
    Flop --> Turn
    Turn --> River
    River --> Showdown
    Folded --> [*]
    Showdown --> [*]
    
    state PlayerActions {
        [*] --> Acting
        Acting --> Raised
        Raised --> Calling
        Calling --> AllCalled
    }
```

Pocker auto fold timeout
```mermaid
flowchart LR
    A[HandSaga] --> B[Add DeadManSwitchTimer]
    B -->|Timeout| C[AutoFoldOrCancel]
    C --> D[CompensateViaAccounting]
```

Pocker waiting diagram
```mermaid
stateDiagram-v2
    [*] --> WaitingForPlayers
    WaitingForPlayers --> PreFlop: min 2 players
    PreFlop --> Flop: all acted/timeout
    Flop --> Turn: bets equalized
    Turn --> River: bets equalized
    River --> Showdown: last bet called
    Showdown --> [*]
    state "PlayerDecision" as PD {
        [*] --> Check
        Check --> Fold
        Check --> Call
        Call --> Raise
    }
    PreFlop --> PD
    Flop --> PD
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
    subgraph Poker
        A[TableActor] --> B[HandSaga]
        B --> C[FSM]
        C --> D[RecoveryService]
        D -->|"Checkpoint every N events"| E[SnapshotStorage]
    end
```


### BlackJack FSM
```mermaid
stateDiagram-v2
    [*] --> Betting
    Betting --> Dealing: All bets done
    Dealing --> PlayerTurn: The cards have been dealt
    PlayerTurn --> DealerTurn: All end
    DealerTurn --> Payout: Diller end
    Payout --> [*]
    
    state PlayerTurn {
        [*] --> HitOrStand
        HitOrStand --> DoubleDown: If allowed
        DoubleDown --> Stand
    }
```

### BlackJack Event Sourcing
```mermaid
flowchart TD
    A[GameStarted] --> B[BetPlaced]
    B --> C[CardsDealt]
    C --> D[PlayerHit]
    D --> E[PlayerStand]
    E --> F[DealerRevealed]
    F --> G[PayoutCompleted]
    
    style A fill:#f9f,stroke:#333
    style G fill:#bbf,stroke:#333
```

### BlackJack Integration Points
```mermaid
flowchart LR
    BJ[Blackjack] -->| ReserveFunds| AC[Accounting]
    BJ -->| GetTrueRandom| RNG[RNG Service]
    BJ -->| PublishOutcome| ES[EventStore]
    BJ -->| NotifyResult| WS[WebSockets]
    WS -->|Push| Player[Player UI]
```

### Racing module
```mermaid
flowchart TD
    A[RaceCoordinator] --> B[MarketMaker]
    B --> C[OddsCalculator]
    C --> D[PositionEngine]
    D --> E[ResultValidator]
```

RNG verifications
```mermaid
flowchart TD
    A[PositionEngine] --> B[Add DeterministicRNG]
    B --> C[Seed=RaceId+Round]
    C --> D[Повторяемость результатов]
```

## Tournament Module
```mermaid
flowchart TD
    subgraph Tournament.Module
        TM[TournamentManager]
        TM -->|Manages| TS[TournamentState]
        TM -->|Coordinates| TR[TournamentRegistry]
        TM -->|Routing| TSched[TableScheduler]
        TM -->|Rules| TRules[TournamentRulesEngine]
        
        subgraph Games
            TSched -->|Create tables| Poker[Poker.Module]
            TSched -->|Create tables| Blackjack[Blackjack.Module]
        end

        TM -->|Payment logic| Accounting[Accounting.Module]
        TM -->|Notifications| Notify[Notifications]
    end

    TM -->|Events| Bus[[RedPanda]]
    Bus --> Analytics[Analytics]
    Bus --> Audit[Audit]
```

### Tournament states
```mermaid
stateDiagram-v2
    [*] --> Registration
    Registration --> Scheduled: Enough players
    Scheduled --> Running: Timer start
    Running --> Rebalancing: The player is eliminated
    Rebalancing --> Running
    Running --> FinalStage: There are less ≤  than M players
    FinalStage --> Completed: The winner has been
    Completed --> Payouts: Payouts
    Payouts --> [*]

    note left of Registration
        Rules:
        - Minimum number of players: N
        - Maximum: 10*N
        - Login via Accounting.ReserveFunds
    end note
```

### Tournament starts
```mermaid
sequenceDiagram
    participant User
    participant API
    participant Tournament
    participant Poker/Blackjack
    participant Accounting

    User->>API: Register for the tournament ($20)
    API->>Accounting: ReserveFunds(userId, $20)
    Accounting-->>API: OK
    API->>Tournament: AddPlayer(tournamentId, userId)
    Tournament->>Tournament: Check the minimum number 
    alt Enough players
        Tournament->>Poker/Blackjack: CreateTables(players, rules)
        Poker/Blackjack-->>Tournament: TablesReady
        Tournament->>Bus: TournamentStartedEvent
    else NotEnough
        Tournament->>API: WaitingForPlayers
    end
```

### Ребалансировка столов
```mermaid
flowchart LR
    subgraph Tournament
        A[Player eliminated] --> B{Check the number of players}
        B -->|Table < M/2| C[Combine tables]
        B -->|Table > M+1| D[Divide the table]
        C & D --> E[Update the Tournament Registry]
        E --> F[Notify players via WebSocket]
    end
```
```mermaid
sequenceDiagram
    TournamentManager->>TableManager: BalanceTablesRequest
    TableManager->>PokerEngine: GetCurrentStacks
    PokerEngine-->>TableManager: StackInfo
    TableManager->>TournamentManager: ProposedRebalance
```
###  Pocker Tournament
```mermaid
flowchart TD
    subgraph Poker.Module
        PT[PokerTable] -->|Events| TM[PokerTournamentManager]
        TM -->|Commands| PT
    end

    NoteP[PlayerEliminated/TableNeedsRebalance]
    PT --> NoteP

```

### Blackjack tournament
```mermaid
flowchart TD
    subgraph Blackjack.Module
        BT[BlackjackTable] -->|Events| TM[BlackjackTournamentManager]
        TM -->|Commands| BT
    end

    NoteB[PlayerBusted\TournamentRoundEnded]
    BT --> NoteB

```

| Parameter | Poker Tournament | Blackjack Tournament |
| ------------------ | -------------------- | ----------------------------------- |
| **Victory**         | Last survivor | Highest bankroll after N rounds |
| **Rebalancing** | Pool tables | Fixed tables |
| **Timings**       | Growing Blinds | Fixed rounds (3 min)        |
| **Payments**        | Prizes (1-3) | Top 10% of players |

### Kafka topics for tournaments
```
tournament.events
  - TournamentStarted
  - PlayerEliminated
  - PrizePoolUpdated

tournament.commands
  - AddPlayer
  - StartTournament
  - CancelTournament
```

### Cancelations
```mermaid
sequenceDiagram
    Tournament->>Accounting: CancelReservation(tournamentId)
    Accounting->>Tournament: FundsReleased
    Tournament->>Poker/Blackjack: DestroyAllTables()
```
