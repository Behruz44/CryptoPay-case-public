# CryptoPay — Technical Case Study

> A production-oriented crypto payment platform built solo in 10 weeks. 14 microservices. Real blockchain integration. Atomic financial operations. Zero test downtime.

**Author:** Behruz (solo developer, 15 y/o)  
**Duration:** April 2026 – June 2026 (10 weeks, 81 commits)  
**Stack:** NestJS · TypeScript · PostgreSQL · Redis · Docker · Kubernetes · Flutter · Python

---

## What is CryptoPay?

CryptoPay is a platform for:
- **Users**: crypto/fiat wallets, QR/NFC payments, deposit detection on-chain, transaction history
- **Merchants**: settlement with fee calculation, balance tracking, payout orchestration
- **Operators**: KYC processing, blockchain monitoring, hot-to-cold wallet sweeps

Built as a learning project with production-grade discipline.

---

## Architecture at a Glance

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Mobile App │────▶│ API Gateway │────▶│ Auth Service│
│  (Flutter)  │     │   (NestJS)  │     │  (JWT + 2FA)│
└─────────────┘     └──────┬──────┘     └─────────────┘
                           │
       ┌───────────────────┼───────────────────┐
       ▼                   ▼                   ▼
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│   Wallet    │   │ Orchestrator│   │    KYC      │
│  Service    │   │  (Saga)     │   │  (Sumsub)   │
│(Atomic Txns)│   │(State Mach) │   │  (Webhooks) │
└─────────────┘   └──────┬──────┘   └─────────────┘
                         │
       ┌─────────────────┼─────────────────┐
       ▼                 ▼                 ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Blockchain  │  │  Settlement │  │   Signing   │
│  Watcher    │  │  Service    │  │  Service    │
│ (Polygon    │  │ (Fee + DL)│  │  (HD Keys)  │
│  USDT)      │  │             │  │             │
└─────────────┘  └─────────────┘  └─────────────┘
```

### Infrastructure
- **Docker + docker-compose**: local development with 13+ services
- **Kubernetes**: real manifests for all services, ingress, configmaps, secrets
- **Helm**: Chart with deployment templates
- **CI/CD**: GitHub Actions — build matrix, TS compile, tests, lint, security scanning (Gitleaks, Semgrep, Trivy, Hadolint)
- **Monitoring**: Prometheus metrics endpoint + Grafana provisioning (docker-compose)

---

## Three Things I’m Proud Of

### 1. Atomic Financial Transfers with BigInt Precision

The core `atomicTransfer` method uses:
- **TypeORM `SERIALIZABLE`** transactions
- **Pessimistic locking** (`SELECT FOR UPDATE`) on user rows
- **BigInt-safe arithmetic** (`toMinorUnits`/`fromMinorUnits`) to eliminate float-precision bugs
- **Double-entry ledger**: every debit has a matching credit record

This was validated with **120 automated tests**, including adversarial scenarios: double-spend attempts, replay attacks, partial failure mid-transaction, and ledger invariant checks.

> **Critical bug found during development:** `user.balance += depositData.amount` was doing string concatenation (`'100' + 50 = '10050'`). Caught during TypeScript type-checking and fixed with BigInt-safe utilities across 4 services.

### 2. Real Blockchain Deposit Pipeline

The `blockchain-watcher` is not a mock. It:
- Listens to **USDT `Transfer` events on Polygon** via `ethers.js`
- Implements **multi-RPC failover** (3 endpoints with automatic rotation)
- Tracks **block confirmations** (configurable)
- Writes to a **dead-letter queue** (`missed_events`) for failed operations
- Runs a **recovery cron** every 10 minutes for stuck payments
- Exposes **Prometheus metrics** and health endpoint

### 3. Security Hardening Discovered During Build

Found and fixed a **plaintext PIN storage vulnerability** during routine type-checking:
- `pinHash` from the client was saved verbatim (no hashing)
- Compared with `!==` (timing attack vulnerability)
- **Fixed with bcrypt (rounds=12)**, timing-safe comparison, and a legacy migration script
- Added 9 regression tests proving the fix

This shows the value of treating type-checking and testing as security gates, not afterthoughts.

---

## Tech Stack (Honest Assessment)

| Technology | Integration |
|---|---|
| NestJS + TypeScript | 13 services, fully typed, 0 TS errors |
| PostgreSQL | Shared DB, TypeORM + raw SQL where needed |
| Redis | Rate limiting, OTP, refresh tokens, quote caching, idempotency |
| Kafka | Topics defined in docker-compose, but **services communicate via REST** (future event-driven migration) |
| Docker / docker-compose | Fully working local stack |
| Kubernetes | Real manifests (not placeholders) |
| Helm | Chart with values and deployment templates |
| GitHub Actions | Build, test, lint, security scan — deploy steps are staged |
| Flutter | 89 Dart files, real screens (wallet, QR, history, profile) |
| Python | Conversion engine + analytics |

---

## Security Practices

- **Auth**: JWT (15-min TTL) + refresh token rotation (Redis, 7 days) + optional TOTP
- **PIN**: bcrypt (rounds=12), timing-safe comparison, legacy migration
- **Rate limiting**: Redis-backed per-route guards
- **Service-to-service**: `x-internal-key` header (min 32 chars)
- **Input validation**: `class-validator` DTOs, regex amount validation, BigInt arithmetic
- **CI security**: Gitleaks (secret scan), Semgrep (SAST), Trivy (vulns), Hadolint (Dockerfile lint)
- **Secrets**: Env-only, `.env` gitignored, CI checks for accidental commits

---

## What I Learned

1. **Floats and money don't mix.** JavaScript `0.1 + 0.2 !== 0.3` is not a joke when it’s someone’s balance.
2. **TypeScript strict mode is a security tool.** It caught string-concat bugs and precision loss that would have caused real financial errors.
3. **Testing adversarial scenarios matters.** Happy-path tests don’t catch double-spends. You need to simulate race conditions and malicious inputs.
4. **A 15-year-old can build production-oriented architecture.** It’s about discipline, not age: type safety, atomic operations, secret scanning, and honest documentation.

---

## Repository Status

- **Private source repo**: Contains full service code, env templates, and CI configs
- **This case-study repo**: Public documentation, architecture diagrams, and proof of work
- **Deployment**: Docker-compose stack runs locally; Kubernetes manifests are ready for cloud deployment

---

## Contact

Built by Behruz — open to questions on architecture, NestJS patterns, or blockchain integration.

> "The goal was not to launch a unicorn. The goal was to prove I can think like an engineer."
