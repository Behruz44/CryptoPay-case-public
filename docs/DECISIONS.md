# Key Technical Decisions

## 1. Two Auth Services Instead of One

**Decision:** `wallet-service/src/auth.service.ts` handles wallet-local concerns (PIN, freeze, FCM, KYC submit) while the standalone `auth-service/` handles JWT issuance and refresh tokens.

**Reasoning:**
- Wallet operations (PIN verification, freeze checks) are high-frequency and need sub-millisecond access to user rows
- JWT operations (token rotation, TOTP) are lower-frequency and benefit from dedicated secret management
- Prevents coupling auth domain complexity with wallet transaction hot path

**Trade-off:** Confusing naming. If rebuilding, would call them `wallet-auth` and `jwt-service`.

---

## 2. BigInt-First Money Arithmetic

**Decision:** All monetary values pass through `toMinorUnits`/`fromMinorUnits` using BigInt, never raw JavaScript numbers.

**Reasoning:**
- PostgreSQL `NUMERIC(36,18)` returns strings
- `Number("100.50")` is safe but `user.balance += amount` was string-concatenating
- BigInt guarantees exact precision for deposits, transfers, and fee calculations

**Impact:** 29 regression tests for amount utilities; zero float-precision bugs since fix.

---

## 3. Shared PostgreSQL Instead of Per-Service DB

**Decision:** All services connect to one PostgreSQL instance.

**Reasoning:**
- Solo developer — operational simplicity matters more than theoretical purity
- Financial operations need cross-table consistency (user balance + transaction + payment in one tx)
- TypeORM migrations are centralized

**Trade-off:** Tight coupling at data layer. If scaling, would shard by tenant or introduce CQRS for reads.

---

## 4. REST Over Kafka (For Now)

**Decision:** Inter-service communication uses HTTP REST with `x-internal-key`, not Kafka events.

**Reasoning:**
- Easier to debug with `curl` and Postman
- Synchronous flows (transfer, QR payment) need immediate response
- Kafka topics are defined but not yet the primary transport

**Trade-off:** Tighter temporal coupling. Orchestrator state machine and dead-letter queue (`missed_events`) mitigate this.

---

## 5. Pessimistic Locking Over Optimistic

**Decision:** `SERIALIZABLE` transactions with `pessimistic_write` locks on user rows during transfers.

**Reasoning:**
- Double-spend is a catastrophic failure
- Optimistic locking requires retry logic that complicates the happy path
- PostgreSQL `SELECT FOR UPDATE` is fast enough for expected concurrency

**Validation:** Adversarial tests simulate 50 concurrent transfer attempts; all resolve correctly with no balance drift.

---

## 6. Flutter + NestJS Monorepo

**Decision:** Mobile app lives in same repo as backend (`cryptopay_app/`).

**Reasoning:**
- Single PR can change API contract and mobile consumer simultaneously
- Docker-compose can spin up full stack for E2E testing
- 89 Dart files share OpenAPI types via code generation

**Trade-off:** Repo size and CI time. Flutter build adds ~4 minutes to pipeline.
