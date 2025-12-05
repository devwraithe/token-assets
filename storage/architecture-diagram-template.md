```
 ════════════════════════════════════════════════════════════════════════════
                          SOLANA VAULT ARCHITECTURE
════════════════════════════════════════════════════════════════════════════

┌──────────────────────────────────────────────────────────────────────────┐
│                         1. PRESENTATION LAYER                            │
└──────────────────────────────────────────────────────────────────────────┘

                               User Interface
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
                    ▼                ▼                ▼
            ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
            │   Phantom    │ │   Frontend   │ │  Mobile App  │
            │   Wallet     │ │   (React)    │ │              │
            └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
                   │                │                │
                   └────────────────┼────────────────┘
                                    │
                           [1] User Action
                           • Connect wallet
                           • Sign transaction
                                    │
                                    ▼

┌──────────────────────────────────────────────────────────────────────────┐
│                         2. API GATEWAY LAYER                             │
└──────────────────────────────────────────────────────────────────────────┘

                          ┌──────────────┐
                          │ Load Balancer│
                          └──────┬───────┘
                                 │
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
            ┌──────────┐  ┌──────────┐  ┌──────────┐
            │   API    │  │   API    │  │   API    │
            │ Gateway 1│  │ Gateway 2│  │ Gateway 3│
            └────┬─────┘  └────┬─────┘  └────┬─────┘
                 │             │             │
                 └─────────────┼─────────────┘
                               │
                      [2] Authentication
                      • Verify signature
                      • Rate limiting
                      • Input validation
                               │
                               ▼
                         ┌──────────┐
                         │  Cache   │────── [Cache Hit] ───► Return
                         │ (Redis)  │
                         └────┬─────┘
                              │
                        [Cache Miss]
                              │
                              ▼

┌──────────────────────────────────────────────────────────────────────────┐
│                        3. APPLICATION LAYER                              │
└──────────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────────┐
                    │   Vault Service     │
                    │                     │
                    │ [3] Business Logic  │
                    │ • Calculate shares  │
                    │ • Validate amounts  │
                    │ • Check capacity    │
                    │ • Apply fees        │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Transaction Builder │
                    │                     │
                    │ [4] Build TX        │
                    │ • Recent blockhash  │
                    │ • Compute budget    │
                    │ • Priority fees     │
                    │ • Instructions      │
                    └──────────┬──────────┘
                               │
                               ▼

╔══════════════════════════════════════════════════════════════════════════╗
║                        4. BLOCKCHAIN LAYER                               ║
╚══════════════════════════════════════════════════════════════════════════╝

                         [5] Submit TX
                               │
                    ┌──────────┼──────────┐
                    ▼          ▼          ▼
            ┌──────────┐ ┌──────────┐ ┌──────────┐
            │   RPC    │ │   RPC    │ │   RPC    │
            │ Primary  │ │ Backup 1 │ │ Backup 2 │
            └────┬─────┘ └────┬─────┘ └────┬─────┘
                 │            │            │
                 └────────────┼────────────┘
                              │
                    [6] Simulate & Validate
                              │
                ┌─────────────┴─────────────┐
                │                           │
           [Failed]                    [Passed]
                │                           │
                ▼                           ▼
         ┌──────────┐              ┌─────────────────┐
         │  Return  │              │ Execute On-Chain│
         │  Error   │              └────────┬────────┘
         └──────────┘                       │
                                            ▼
                              ┌──────────────────────┐
                              │   Vault Program      │
                              │   (Rust/Anchor)      │
                              │                      │
                              │ [7] Execute:         │
                              │  a) Verify signer    │
                              │  b) Validate accounts│
                              │  c) Check constraints│
                              │  d) Calculate shares │
                              │  e) Update state     │
                              └──────────┬───────────┘
                                         │
                              [8] Cross-Program Call
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                    │
                    ▼                    ▼                    ▼
          ┌──────────────┐     ┌──────────────┐    ┌──────────────┐
          │ Token Program│     │Vault State   │    │ User Deposit │
          │              │     │    (PDA)     │    │    (PDA)     │
          │[9] Transfer  │     │              │    │              │
          │   Tokens     │     │• Total: 1.2M │    │• User shares │
          └──────────────┘     │• Price: 1.02 │    │• Timestamp   │
                               └──────────────┘    └──────────────┘
                                         │
                              [10] Emit Event
                                         │
                                         ▼

┌──────────────────────────────────────────────────────────────────────────┐
│                      5. INFRASTRUCTURE LAYER                             │
└──────────────────────────────────────────────────────────────────────────┘

                         Async Processing
                               │
            ┌──────────────────┼──────────────────┐
            │                  │                  │
            ▼                  ▼                  ▼
    ┌──────────────┐   ┌──────────────┐  ┌──────────────┐
    │   Indexer    │   │   Database   │  │  Monitoring  │
    │              │   │  (Postgres)  │  │              │
    │[11] Parse TX │   │              │  │[13] Alerts   │
    │   & Store    │   │[12] Persist  │  │   & Metrics  │
    │              │   │   History    │  │              │
    └──────┬───────┘   └──────┬───────┘  └──────────────┘
           │                  │
           └─────────┬────────┘
                     │
                     ▼
              ┌──────────────┐
              │Update Cache  │
              │& Notify User │
              └──────────────┘


════════════════════════════════════════════════════════════════════════════
                            ERROR HANDLING FLOW
════════════════════════════════════════════════════════════════════════════

    RPC Failure ──► Retry (1s) ──► Fallback RPC ──► Retry (2s) ──► Error
         │                                                            │
         └─────────────────────► Log & Alert ◄───────────────────────┘

    TX Failed ──► Analyze Reason ──┬──► Insufficient Funds ──► Notify User
                                    ├──► Slippage Error ──────► Notify User
                                    ├──► Vault Full ──────────► Notify User
                                    └──► Unknown ─────────────► Alert Team


════════════════════════════════════════════════════════════════════════════
                           SECURITY BOUNDARIES
════════════════════════════════════════════════════════════════════════════

    [Boundary 1] Client ←──────────→ API Gateway
                 • HTTPS/TLS 1.3
                 • Signature verification
                 • Rate limiting

    [Boundary 2] Off-Chain ←───────→ On-Chain
                 • All validation in smart contract
                 • PDA seed verification
                 • Signer checks

    [Boundary 3] Vault Program ←───→ Token Program
                 • CPI guards
                 • Account ownership validation


════════════════════════════════════════════════════════════════════════════
                              KEY METRICS
════════════════════════════════════════════════════════════════════════════

    Performance:
    • API Response Time: < 100ms (cached), < 500ms (uncached)
    • Transaction Confirmation: ~400ms (optimistic), ~13s (finalized)
    • Cache TTL: 30 seconds
    • Indexer Lag: < 500ms

    Reliability:
    • Uptime SLA: 99.9%
    • RPC Failover: < 2s
    • Rate Limit: 100 requests/min per wallet
    • Max Retry Attempts: 3

    Cost:
    • Protocol Fee: 0.1%
    • Priority Fee: 0.0001 SOL
    • Compute Units: 200,000
    • Min Deposit: 0.01 SOL


════════════════════════════════════════════════════════════════════════════
                            COMPONENT LEGEND
════════════════════════════════════════════════════════════════════════════

    ┌─────────┐    = Off-chain component
    │         │
    └─────────┘

    ╔═════════╗    = On-chain component / Security boundary
    ║         ║
    ╚═════════╝

    ──────►        = Synchronous flow
    ─ ─ ─►         = Asynchronous flow
    [N]            = Step number
    •              = Action or data point
    │              = Vertical flow
    ┌──┴──┐        = Branching decision


════════════════════════════════════════════════════════════════════════════
                          DEPLOYMENT ARCHITECTURE
════════════════════════════════════════════════════════════════════════════

    Production Environment:
    
    ┌─────────────────────────────────────────────────────────────┐
    │  Region: US-East                                            │
    ├─────────────────────────────────────────────────────────────┤
    │                                                             │
    │  API Gateway Cluster:        3 instances (auto-scaling)    │
    │  Redis Cache:                Cluster mode (HA)             │
    │  PostgreSQL:                 Primary + 2 read replicas     │
    │  Indexer:                    2 instances (active-active)   │
    │  Monitoring:                 Datadog + PagerDuty           │
    │                                                             │
    │  RPC Providers:                                            │
    │  • Primary:   Helius (dedicated)                           │
    │  • Backup 1:  Triton (shared)                              │
    │  • Backup 2:  QuickNode (shared)                           │
    │                                                             │
    │  Solana Network:            Mainnet-Beta                   │
    │  Program ID:                Vault1111111111111111111111111 │
    │                                                             │
    └─────────────────────────────────────────────────────────────┘


════════════════════════════════════════════════════════════════════════════
                            DATA FLOW SUMMARY
════════════════════════════════════════════════════════════════════════════

    Write Flow (Deposit):
    User → Wallet Sign → API → Validate → Build TX → RPC → 
    Vault Program → Token Transfer → Update State → Emit Event → 
    Indexer → Database → Cache Update → User Notification

    Read Flow (Balance):
    User → API → Cache Check → [Hit: Return] | [Miss: Database → 
    Cache → Return]

    Time to Finality:
    • Optimistic: ~400ms (processed confirmation)
    • Confirmed: ~1s (confirmed by supermajority)
    • Finalized: ~13s (finalized by cluster)


════════════════════════════════════════════════════════════════════════════
```
