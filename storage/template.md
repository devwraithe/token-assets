# Architecture Diagram

## USERS LAYER
```
┌────────────────────────────────────────────────────────┐
│                                                          │
│  [Creator]    [Joiner]    [Resolver]    [Admin]         │
│                                                          │
└────────┬────────────┬──────────┬────────────┬───────────┘
         │            │          │            │
```

## PROGRAM LAYER (Smart Contract)
```
┌────────────────────────────────────────────────────────┐
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │create_market │  │ join_market  │  │resolve_market│  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                 │                 │           │
│  ┌──────┴───────┐  ┌──────┴───────┐  ┌──────┴───────┐  │
│  │cancel_market │  │ claim_funds  │  │ dispute_res  │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │ VALIDATION & SECURITY                           │   │
│  │ • Oracle staleness check (max age: 60s)         │   │
│  │ • Minimum market duration enforcement           │   │
│  │ • Signature verification for all participants   │   │
│  │ • Reentrancy guards on fund transfers           │   │
│  └─────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────┘
```

## ACCOUNT LAYER
```
┌────────────────────────────────────────────────────────┐
│                                                          │
│  [MarketState PDA] → stores market info:                │
│      • creator, joiner, resolver                        │
│      • start_price, end_price, target_price             │
│      • expiry_time, status, winner                      │
│      • created_at, resolved_at                          │
│      • dispute_window (24h after expiry)                │
│                                                          │
│  [Escrow PDA] → holds locked SOL from both              │
│      • auto-derived from market + participants          │
│      • only transferable by program logic               │
│                                                          │
│  [Treasury PDA] → collects protocol fees                │
│      • 2% platform fee on winnings                      │
│      • 0.5% resolver compensation                       │
│      • controlled by admin multisig                     │
│                                                          │
│  [Pyth Price Account] → external oracle (read-only)     │
│      • provides live price feed for asset (SOL/USD)     │
│      • confidence interval validation                   │
│      • staleness check enforced                         │
└────────────────────────────────────────────────────────┘
```

## RESOLUTION FLOW
```
┌────────────────────────────────────────────────────────┐
│                                                          │
│  1. CHECK: expiry_time reached?                         │
│  2. FETCH: latest oracle price (with staleness check)   │
│  3. VALIDATE: confidence interval acceptable?           │
│  4. COMPARE: start vs. end prices → determine winner    │
│  5. CALCULATE: winnings - protocol_fee - resolver_fee   │
│  6. TRANSFER:                                            │
│     • Net winnings → winner                             │
│     • Protocol fee → Treasury PDA                       │
│     • Resolver fee → Resolver                           │
│  7. UPDATE: market status → "Resolved"                  │
│                                                          │
│  [DISPUTE PATH]                                         │
│  • Available within 24h dispute window                  │
│  • Requires stake to prevent spam                       │
│  • Admin reviews and can override resolution            │
└────────────────────────────────────────────────────────┘
```

## EDGE CASES & ERROR HANDLING
```
┌────────────────────────────────────────────────────────┐
│                                                          │
│  • Stale Oracle: Retry with exponential backoff         │
│  • Failed Resolution: Market enters "Pending" state     │
│  • No Joiner: Creator can cancel after timeout period   │
│  • Price Manipulation: Use TWAP over 5-minute window    │
│  • Network Congestion: Priority fee adjustment logic    │
└────────────────────────────────────────────────────────┘
```

---

## Key Improvements Over Original Design

### 1. Enhanced User Roles
- Added **Admin** role for platform governance and dispute resolution
- Clear separation of concerns between all four roles

### 2. Additional Program Functions
- **cancel_market**: Allows creator to cancel if no joiner within timeout
- **claim_funds**: Separate claiming mechanism for winners
- **dispute_res**: Enables dispute resolution within 24-hour window

### 3. Security & Validation Layer
- Oracle staleness check (maximum 60 seconds old)
- Minimum market duration enforcement
- Signature verification for all participants
- Reentrancy guards on fund transfers

### 4. Fee Structure & Economics
- **Platform fee**: 2% on winnings → Treasury PDA
- **Resolver compensation**: 0.5% for executing resolution
- Treasury controlled by admin multisig for governance

### 5. Enhanced Market State
- Added `target_price` for more complex market types
- Timestamps: `created_at`, `resolved_at`
- Dispute window tracking (24 hours after expiry)

### 6. Dispute Mechanism
- 24-hour dispute window after resolution
- Requires stake to prevent spam disputes
- Admin oversight for final resolution

### 7. Detailed Resolution Process
- Step-by-step validation and execution flow
- Clear fee calculation and distribution
- Oracle confidence interval checks

### 8. Edge Case Handling
- **Stale Oracle**: Exponential backoff retry logic
- **Failed Resolution**: Graceful degradation to "Pending" state
- **No Joiner**: Timeout-based cancellation mechanism
- **Price Manipulation**: TWAP (Time-Weighted Average Price) over 5 minutes
- **Network Issues**: Dynamic priority fee adjustment

### 9. Oracle Improvements
- Confidence interval validation
- Staleness enforcement (max 60s)
- Read-only external account pattern

### 10. Treasury Management
- Dedicated PDA for protocol fees
- Multisig control for security
- Clear fee distribution logic

---

## Implementation Notes

### Security Considerations
1. All fund transfers should include reentrancy guards
2. Oracle data must be validated for both staleness and confidence
3. Admin actions should require multisig approval
4. Dispute stakes should be sufficient to deter frivolous disputes

### Scalability Considerations
1. Use TWAP to prevent flash-loan price manipulation
2. Implement priority fee logic for network congestion
3. Consider batch resolution for multiple markets
4. Cache oracle data when appropriate (within staleness window)

### Testing Priorities
1. Oracle failure scenarios
2. Dispute flow end-to-end
3. Fee calculation accuracy
4. Reentrancy attack vectors
5. Cancellation edge cases

---

## Future Enhancements
- Multi-asset market support
- Automated market makers (AMM) for liquidity
- Governance token for admin decisions
- Advanced market types (binary, scalar, categorical)
- Cross-program invocation for DeFi integrations
