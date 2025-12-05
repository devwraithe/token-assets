```
                    ┌──────────────────────────────────────┐
                    │         User Interface Layer         │
                    └──────────────────────────────────────┘
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        │                             │                             │
        ▼                             ▼                             ▼
┌───────────────┐            ┌───────────────┐            ┌───────────────┐
│  Web3 Wallet  │            │   Frontend    │            │  Mobile App   │
│  (Phantom/    │ 1. Connect │   (React)     │            │   (React      │
│   Solflare)   │ & Sign     │               │            │    Native)    │
└───────┬───────┘            └───────┬───────┘            └───────┬───────┘
        │                            │                            │
        └────────────────────────────┼────────────────────────────┘
                                     │ 2. Deposit/Withdraw Request
                                     │    (Amount, Token Address)
                                     ▼
                            ┌────────────────┐
                            │   API Gateway  │
                            │  (Rate Limit,  │ 3. Authenticate
                            │   Auth Check)  │    & Validate
                            └────────┬───────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │ 4. Route       │                │
                    ▼                ▼                ▼
        ┌──────────────────┐ ┌──────────────┐ ┌─────────────────┐
        │  Vault Service   │ │   Oracle     │ │  Notification   │
        │  (Business Logic)│ │   Service    │ │    Service      │
        └────────┬─────────┘ └──────┬───────┘ └─────────────────┘
                 │                  │
                 │ 5. Check vault   │ 6. Fetch price
                 │    state & build │    feeds (optional)
                 │    transaction   │
                 │                  │
                 └────────┬─────────┘
                          │ 7. Serialize transaction
                          ▼
                 ┌────────────────┐
                 │   Transaction  │ 8. Add compute budget,
                 │    Builder     │    priority fees,
                 │                │    recent blockhash
                 └────────┬───────┘
                          │
                          ▼
        ╔═════════════════════════════════════════╗
        ║         Solana Blockchain Layer         ║
        ╚═════════════════════════════════════════╝
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│  RPC Endpoint │ │  Vault Program│ │  Token Program│
│   (Helius/    │ │   (On-chain   │ │   (SPL Token) │
│    Triton)    │ │    Rust/      │ │               │
└───────┬───────┘ │    Anchor)    │ └───────┬───────┘
        │         └───────┬───────┘         │
        │ 9. Submit Tx    │                 │
        │                 │ 10. Execute:    │
        │                 │  • Verify signer│
        │                 │  • Check vault  │ 11. Transfer
        │                 │    constraints  │     tokens
        │                 │  • Update state │
        │                 │                 │
        │                 ▼                 ▼
        │         ┌───────────────┐ ┌──────────────┐
        │         │  Vault State  │ │  User Token  │
        │         │   (PDA)       │ │   Accounts   │
        │         │  • Total TVL  │ │  (ATA)       │
        │         │  • Share price│ │              │
        │         │  • Strategies │ │              │
        │         └───────────────┘ └──────────────┘
        │
        │ 12. Tx confirmation (signature)
        ▼
┌────────────────┐
│   Indexer/     │ 13. Parse & store
│   Database     │     transaction data
│  (PostgreSQL)  │     for historical queries
└────────┬───────┘
         │
         │ 14. Event emission
         ▼
┌────────────────┐
│   Monitoring   │ • Alert on failures
│   & Logging    │ • Track metrics (TVL, APY)
│   (Datadog)    │ • Security monitoring
└────────────────┘
```
```
Layer separation
┌──────────────────┐
│  Presentation    │  ← User-facing
├──────────────────┤
│  Application     │  ← Business logic
├──────────────────┤
│  Blockchain      │  ← On-chain state
├──────────────────┤
│  Infrastructure  │  ← Monitoring/Storage
└──────────────────┘

Program to Program interaction
┌─────────────┐
│ Vault       │ Cross-Program
│ Program     │ Invocation (CPI)
└──────┬──────┘        │
       │ ◄─────────────┘
       ▼
┌─────────────┐
│ Token       │
│ Program     │
└─────────────┘

State management (pda pattern)
Vault Program Address
       │
       ├─► Vault State PDA (seeds: ["vault", authority])
       │      • Total deposits
       │      • Share calculation
       │
       └─► User Deposit PDA (seeds: ["deposit", user, vault])
              • User shares
              • Last interaction
```
