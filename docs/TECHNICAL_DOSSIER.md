# CryptoPay — Техническое досье
## Платёжная блокчейн-инфраструктура для лицензированных провайдеров крипто-услуг

**Версия документа:** 1.0 · Июль 2026
**Статус проекта:** Готовый MVP, прошедший внутренний security-аудит (30 исправлений)
**Предложение:** Продажа / white-label лицензирование / технологическое партнёрство

---

## 1. Что это

CryptoPay — готовая инфраструктура для крипто-финансовых операций:

- **Кастодиальные кошельки** — HD-деривация (BIP-44), автоматический sweep hot → cold storage
- **Приём депозитов on-chain** — реальный мониторинг USDT Transfer-событий на Polygon через ethers.js
- **Платёжный оркестратор** — state-machine (Saga-паттерн) с outbox, идемпотентностью и recovery-cron
- **Конвертация крипто ↔ фиат** — котировки в реальном времени через liquidity-adapter
- **KYC-пайплайн** — интеграция с Sumsub (webhooks с HMAC-верификацией)
- **Мерчант-расчёты** — settlement-service с расчётом комиссий
- **Мобильное приложение** — Flutter (89 Dart-файлов): кошелёк, QR-платежи, история, профиль

## 2. Архитектура

14 микросервисов на NestJS/TypeScript + 2 Python-сервиса:

```
Mobile App (Flutter)
        │
   API Gateway ──── Auth Service (JWT + TOTP + refresh rotation)
        │
   ┌────┼──────────────┬──────────────┐
Wallet  Orchestrator   KYC          User
Service (Saga/outbox) (Sumsub)     Service
   │        │
   │   ┌────┴────────┬─────────────┐
   │ Blockchain   Settlement    Signing Service
   │ Watcher      Service       (изолирован в docker-сети,
   │ (Polygon     (комиссии)     HD-ключи, spending limits,
   │  USDT)                      наружу закрыт)
   │
PostgreSQL · Redis · Kafka(staged) · MinIO · Prometheus/Grafana
```

### Ключевые инженерные решения

| Решение | Реализация |
|---|---|
| **Финансовая точность** | Вся арифметика — BigInt в minor units (NUMERIC 36,18). Ни одной float-операции с деньгами |
| **Атомарность** | SERIALIZABLE-транзакции + пессимистичные блокировки (SELECT FOR UPDATE) + двойная запись (ledger) |
| **Изоляция ключей** | Signing-service держит мнемонику отдельно, доступен только внутри docker-сети, авторизация по internal key + маскированный security-logging |
| **Лимиты расходов** | SpendingLimitsGuard: Redis-based дневные лимиты вывода, атомарный INCR с TTL до конца UTC-дня |
| **Отказоустойчивость** | Multi-RPC failover (3 эндпоинта), dead-letter queue для пропущенных событий, recovery-cron каждые 10 мин |
| **Graceful shutdown** | Очистка соединений (DB, Redis), unref() на интервалах, OnModuleDestroy во всех long-lived сервисах |

## 3. Безопасность (после внутреннего аудита)

- **Аутентификация:** JWT (HS256 enforced, TTL 15 мин) + ротация refresh-токенов в Redis + TOTP для выводов
- **PIN:** bcrypt (rounds=12), timing-safe сравнение, защита от user enumeration
- **Межсервисная связь:** x-internal-key (мин. 32 символа), HMAC-подписи с canonical JSON, timingSafeEqual
- **Валидация:** class-validator DTO на всех endpoint'ах, ValidationPipe (whitelist + forbidNonWhitelisted) глобально
- **Rate limiting:** Redis-backed guards, анти-brute-force по IP с TTL-based tracking
- **Webhooks:** HMAC-SHA256 верификация raw body (Sumsub), retry-семантика (500 на transient errors)
- **CORS:** env-based whitelist во всех сервисах, никаких wildcard
- **CI security:** Gitleaks, Semgrep, Trivy, Hadolint в GitHub Actions

Полный отчёт: [SECURITY_AUDIT_REPORT.md](./SECURITY_AUDIT_REPORT.md)

## 4. Инфраструктура и DevOps

- **Docker Compose** — полный локальный стек (13+ сервисов), профили: dev, prod, monitoring, kafka, backup
- **Kubernetes** — рабочие манифесты (deployments, ingress, configmaps, secrets)
- **Helm** — chart с шаблонами
- **CI/CD** — GitHub Actions: build matrix, тесты, линт, security-сканирование
- **Мониторинг** — Prometheus-метрики во всех сервисах + Grafana dashboards
- **Документация** — Swagger/OpenAPI, Postman-коллекция, архитектурные доки

## 5. Что получает покупатель / партнёр

| Актив | Описание |
|---|---|
| Исходный код | 14 NestJS-сервисов + 2 Python + Flutter-приложение, 0 TypeScript-ошибок |
| Инфраструктура | Docker/K8s/Helm/CI — готово к развёртыванию |
| Тесты | 120+ автотестов, включая adversarial-сценарии (double-spend, replay, race conditions) |
| Документация | Архитектура, API (Swagger), runbook'и, security-отчёты |
| Передача знаний | Onboarding-сессии по архитектуре и деплою |

## 6. Честные ограничения (due diligence)

Мы не скрываем ничего — вот что нужно доработать до полного продакшена:

1. **KMS/HSM** — сейчас мнемоника в env через Vault-заглушку; KmsSignerService (AWS KMS) написан, но не активирован
2. **Kafka** — топики определены, но межсервисная связь пока REST (миграция на event-driven — план)
3. **Внешний пентест** — внутренний аудит проведён (30 исправлений), независимый внешний — не заказывался
4. **Тестовое покрытие** — 120+ тестов в критических путях, но не все сервисы покрыты равномерно
5. **Нагрузочное тестирование** — базовые perf-тесты в CI, полноценный load-test не проводился

Оценка доработки до production-grade: **3–6 месяцев** силами 2–3 инженеров.

## 7. Сценарии использования для лицензиатов НАПП

- **Крипто-депозитарий:** кастодиальный стек (HD-кошельки, sweep, cold storage, signing изоляция) покрывает ~70% технических требований
- **Крипто-магазин:** конвертационный движок + KYC + фиатные расчёты
- **Крипто-биржа:** wallet-инфраструктура + депозитный пайплайн как основа

---

**Контакт:** см. [PITCH_ONE_PAGER.md](./PITCH_ONE_PAGER.md)
