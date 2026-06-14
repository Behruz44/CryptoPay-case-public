# Architecture Deep Dive

## Service Inventory (14 services)

| Service | Port | Language | Responsibility |
|---|---|---|---|
| `api-gateway` | 3000 | TypeScript/NestJS | JWT validation, rate limiting, routing, CORS |
| `auth-service` | — | TypeScript/NestJS | JWT issuance, refresh token rotation, TOTP |
| `wallet-service` | 4300 | TypeScript/NestJS | Balances, atomic transfers, PIN, daily limits, QR/NFC |
| `orchestrator` | — | TypeScript/NestJS | Payment state machine, saga coordination, websocket gateway |
| `blockchain-watcher` | 4400 | TypeScript/Node | On-chain USDT event listener, multi-RPC failover |
| `settlement-service` | — | TypeScript/NestJS | Merchant settlement, fee deduction, idempotency |
| `signing-service` | — | TypeScript/NestJS | HD wallet derivation, transaction signing |
| `sweep-service` | — | TypeScript/NestJS | Hot-to-cold wallet fund sweeps |
| `gas-station` | — | TypeScript/NestJS | MATIC gas top-up for deposit addresses |
| `kyc-service` | — | TypeScript/NestJS | Sumsub API integration, webhook verification |
| `user-service` | — | TypeScript/NestJS | User CRUD |
| `liquidity-adapter` | — | Python | Exchange rate aggregation (Binance P2P) |
| `conversion-engine` | — | Python | Crypto/fiat price conversion |
| `audit-service` | — | TypeScript/NestJS | Audit logging |

## Communication Patterns

### Current State
- **REST HTTP** with `x-internal-key` header for inter-service auth
- **EventEmitter2** (in-process) in orchestrator for websocket broadcasts
- **PostgreSQL** as shared data store

### Future State (Staged)
- **Kafka** topics defined in docker-compose for event-driven migration
- Services already emit structured events; switching to Kafka is a transport change, not a logic change

## Data Model (Wallet Service)

```sql
users              -- UUID, balance (NUMERIC 36,18), pinHash, kycStatus, freeze, dailyLimit
wallets            -- wallet metadata
cards              -- linked payment cards
transactions       -- double-entry audit (debit/credit pairs)
transaction_history -- denormalized query view
payments           -- QR/NFC payment records
sweep_records      -- sweep audit trail
settlement_outbox  -- raw SQL table for merchant settlements
```

## Blockchain Watcher Flow

```
Polygon USDT Transfer Event
        │
        ▼
┌───────────────┐
│ Multi-RPC Pool│── failover on error
└───────┬───────┘
        │
        ▼
┌───────────────┐    ┌───────────────┐
│ Check address │───▶│ updatePayment  │─── PATCH /orchestrator
│  in DB        │    │   Detected     │
└───────────────┘    └───────┬───────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ Confirmation  │   │  Gas Station  │   │ Sweep Service │
│   tracking    │   │  (MATIC topup)│   │ (hot→cold)    │
└───────────────┘   └───────────────┘   └───────────────┘
```

## CI/CD Pipeline

```
Push / PR
    │
    ▼
┌───────────────┐
│ Gitleaks      │── Secret scan
│ Semgrep       │── SAST
│ Hadolint      │── Dockerfile lint
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ Build Matrix  │── 9 NestJS services + Flutter + Python
│ TS Compile    │── 0 type errors enforced
│ npm audit     │── Dependency vuln check
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ Unit Tests    │── wallet-service (120 tests)
│ Trivy Scan    │── Container vuln scan
│ Flutter Build │── APK + Web
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ Deploy Stage  │── Staged (placeholder for prod)
└───────────────┘
```

## Infrastructure

- **Docker**: Multi-stage builds (`node:20-alpine`), non-root user (`nestjs`)
- **docker-compose**: 4 variants (main, prod, kafka, monitoring)
- **K8s**: 14+ YAML files — deployments, ingress, configmaps, secrets, migration job
- **Helm**: `helm/cryptopay/` with templated deployments
