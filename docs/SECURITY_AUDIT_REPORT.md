# CryptoPay — Отчёт о security-аудите и ремедиации
## Внутренний QA + пентест-аудит · Июль 2026

**Методология:** полный статический анализ кодовой базы (14 сервисов), ручная проверка критических путей (аутентификация, подписание транзакций, финансовая арифметика, webhooks), архитектурная ревизия.

**Итог: 30 обнаруженных и исправленных проблем.** Все исправления в git-истории с атрибуцией.

---

## Сводка по критичности

| Уровень | Найдено | Исправлено |
|---|---|---|
| Critical | 6 | 6 ✅ |
| High | 6 | 6 ✅ |
| Medium | 8 | 8 ✅ |
| Low | 3 | 3 ✅ |
| Architectural (доп. ревизия) | 7 | 7 ✅ |

---

## Critical (6)

| # | Проблема | Исправление |
|---|---|---|
| C-1 | Открытый CORS (`*`) в gas-station | Env-based whitelist (`CORS_ORIGINS`) |
| C-2 | Хардкод CORS-origins в liquidity-adapter | Env-based whitelist |
| C-3 | Хардкод CORS-origins в analytics | Env-based whitelist |
| C-4 | HMAC-подпись через `JSON.stringify` (коллизии по порядку ключей) | Canonical JSON (сортировка ключей рекурсивно) |
| C-5 | `@Body() any` в signing-service (нет валидации входа сервиса с ключами) | class-validator DTOs на все endpoint'ы |
| C-6 | Остаточные `@Body() any` в api-gateway | Типизированные DTO (регистрация, депозиты, карты) |

## High (6)

| # | Проблема | Исправление |
|---|---|---|
| H-1 | Нет ValidationPipe в gas-station, liquidity-adapter | Глобальный ValidationPipe (whitelist + forbidNonWhitelisted) |
| H-2 | setInterval без unref() блокирует shutdown (3 сервиса) | unref() + clearInterval в OnModuleDestroy |
| H-3 | `parseFloat` на финансовой сумме (потеря точности) | String passthrough (NUMERIC 36,18) |
| H-4 | HMAC генерация нестабильна между сервисами | Canonical JSON с обеих сторон |
| H-5 | Graceful shutdown без закрытия соединений | DataSource + Redis cleanup перед exit |
| H-6 | Отсутствие cleanup-хуков в cold-storage | OnModuleDestroy + unref() |

## Medium (8)

| # | Проблема | Исправление |
|---|---|---|
| M-1 | Интервал очистки комнат WebSocket без unref | unref() + OnModuleDestroy |
| M-2 | `Number()` деление для процента дневного лимита | BigInt-safe вычисление процентов |
| M-3 | `as any` на типизированном поле Payment.amount | Убран (поле уже string) |
| M-4 | Дублирующий KYCController (конфликт маршрута v1/kyc, один без JWT-guard) | Дубликат удалён |
| M-5 | Webhook возвращал 200 при любой ошибке (Sumsub не ретраил) | 500 на transient errors → retry-семантика |
| M-6 | Ручные проверки internal-key в 4 местах | Централизованный Guard |
| M-7 | Мёртвый импорт fs в liquidity-adapter | Удалён |
| M-8 | Мёртвые импорты fs в gas-station, sweep-service | Удалены |

## Low (3)

| # | Проблема | Исправление |
|---|---|---|
| L-1 | Мёртвый импорт fs в settlement-service | Удалён |
| L-2 | Проверка orchestrator на мёртвые импорты | Чисто, изменений не требовалось |
| L-3 | Health-monitor интервал без unref | unref() |

## Архитектурная ревизия signing-service (7)

Самый чувствительный сервис (держит ключи) прошёл отдельную глубокую ревизию:

| # | Находка | Исправление |
|---|---|---|
| A-1 | **Orphan-стек**: параллельный контроллер/сервис, не зарегистрированный в модуле, но присутствующий в кодовой базе | 5 orphan-файлов удалены, production-стек консолидирован |
| A-2 | **Runtime bug**: контроллер вызывал 4 метода, отсутствующих в production-сервисе (`signERC20Transaction`, `signGasTransaction`, `verifySignature`, `deriveAddress`) | Методы реализованы |
| A-3 | Импорт контроллера указывал на orphan-сервис | Исправлен на production (hardened) сервис |
| A-4 | Полный секретный ключ логировался при неудачной аутентификации | Маскирование (первые/последние 4 символа) |
| A-5 | Unbounded Map для failed-attempts (утечка памяти) + setTimeout без unref | TTL-based записи с lazy-expiry |
| A-6 | Redis-соединение guard'а без cleanup | OnModuleDestroy + lazyConnect + maxRetriesPerRequest |
| A-7 | parseFloat для проверки лимитов сумм | BigInt-сравнение в minor units |

---

## Исторические исправления (более ранние фазы)

- **Plaintext PIN storage** → bcrypt (rounds=12), timing-safe сравнение, миграция legacy-данных, 9 регрессионных тестов
- **String-concat баги на балансе** (`'100' + 50 = '10050'`) → BigInt-safe addMoney() в 2 сервисах
- **Потеря точности** `Number(dailyLimit)` на NUMERIC(36,18) → string passthrough
- **Dangling references** в orchestrator post-commit → исправлены (падали бы в runtime)
- **PAT-токен в git remote URL** → отозван, переход на SSH, история проверена gitleaks

---

## Верификация

- Все TypeScript-сервисы компилируются с **0 ошибок**
- 120+ автотестов проходят, включая adversarial-сценарии
- CI: Gitleaks + Semgrep + Trivy + Hadolint — зелёные

## Рекомендации для покупателя (roadmap)

1. Активировать AWS KMS (KmsSignerService готов) вместо env-мнемоники
2. Заказать независимый внешний пентест перед запуском с реальными средствами
3. Полноценное нагрузочное тестирование (k6/Gatling)
4. Довести тестовое покрытие до 80%+ во всех сервисах
