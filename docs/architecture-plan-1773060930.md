## Анализ задачи: Доработка существующего архитектурного плана plan-1773060179: Нужно использовать Symfony. Так же закрыть security-контур webhook (подмена запросов), Не хватает явной защиты при загрузке/обработке изображений

Обязательно учти предыдущий план и PR этой ветки, сохрани результат как architecture-plan-1773060930.md

### Текущее состояние системы
- По состоянию на 2026-03-09 в ветке `agent/plan-1773060930` кодовая база минимальна: в репозитории присутствует только `README.md`.
- Продакшн-код (`src/`, `config/`, `tests/`), модули, типы, интерфейсы и инфраструктурные адаптеры отсутствуют.
- Тесты и CI-конфигурация отсутствуют.
- Учтен предыдущий план из PR #1 (`https://github.com/vitalypanait/jiuji/pull/1`, ветка `agent/plan-1773060179`, коммит `a7b0b3d` от 2026-03-09):
  - уже зафиксирована базовая декомпозиция Telegram bot + Vision/LLM pipeline;
  - стек был сформулирован как "PHP" без обязательной привязки к Symfony;
  - security-контур webhook и явная защита image ingestion описаны частично и недостаточно конкретно.
- Связанные модули/интерфейсы в коде не найдены (проект не инициализирован), поэтому текущий план задает целевую структуру на Symfony и последовательность безопасного внедрения.

### Риски и зависимости
- `HIGH` Framework-shift риск: переход с абстрактного PHP-плана на Symfony меняет структуру каталогов, DI, lifecycle запроса и может сделать старые пути из PR #1 неактуальными.
- `HIGH` Подмена webhook-запросов: без валидации источника (секрет/подпись/IP) можно инициировать произвольные ответы бота и финансовые/репутационные потери.
- `HIGH` Replay атаки webhook: без окна времени, nonce/update-id дедупликации и idempotency store Telegram retries/злоумышленник вызовут дубликаты сообщений.
- `HIGH` Небезопасная обработка изображений: без MIME sniffing, лимитов размера/пикселей и безопасной перекодировки возможны DoS, image bombs, parser exploits.
- `MEDIUM` Ошибки интеграции Telegram/Vision API: таймауты и сетевые сбои блокируют синхронный webhook-path без очереди.
- `MEDIUM` Отсутствие бизнес-контекста и SLA: нет формальных NFR (latency, throughput, budget), что повышает риск недопроектирования reliability.
- `LOW` Неполная тестовая пирамида на старте: без контрактных/интеграционных тестов регрессии обнаружатся поздно.

Зависимости и блокеры:
- Symfony skeleton + базовая конфигурация блокируют все остальные подзадачи.
- Security middleware для webhook должен быть внедрен до сквозной orchestration и публичного endpoint.
- Image security pipeline должен быть внедрен до передачи данных в Vision adapter.
- Telegram adapter и Vision adapter можно реализовывать параллельно после фиксации контрактов и security-политик.
- Сквозной use-case, observability и финальная приемка зависят от готовности обоих адаптеров.

### Граф подзадач
```text
[1 Symfony foundation + ADR]
              |
              v
[2 Domain contracts + DTO/validators]
      |                     |
      v                     v
[3 Webhook security]   [4 Image security pipeline]
      |                     |
      v                     v
[5 Telegram Symfony adapter]   [6 Vision Symfony adapter]
              \               /
               \             /
                v           v
           [7 Orchestration + idempotency]
                        |
                        v
                [8 Observability + tests]
                        |
                        v
                 [9 Delivery/runbook]
```

### Подзадачи

#### Подзадача 1: Symfony foundation и архитектурный baseline
- **Описание**: Инициализировать проект на Symfony (PHP 8.3+), зафиксировать архитектурные решения (ADR), подготовить базовые `Kernel`, `config/packages`, маршрутизацию и контейнер зависимостей.
- **Файлы**: `composer.json`, `symfony.lock`, `bin/console`, `config/bundles.php`, `config/packages/*.yaml`, `config/routes.yaml`, `public/index.php`, `src/Kernel.php`, `docs/adr/adr-001-symfony-baseline.md`.
- **Запрещено трогать**: `docs/architecture-plan-1773060179.md`, `docs/architecture-plan-1773060930.md`.
- **Критерии приёмки**: `symfony server:start`/docker-эквивалент поднимает приложение; есть health endpoint; архитектурно зафиксировано, почему выбран Symfony и какие bundle/patterns обязательны.
- **Зависит от**: -

#### Подзадача 2: Контракты домена и формата ответа (3 фразы)
- **Описание**: Определить доменные DTO/Value Objects и контракты use-case для сценария "image -> 3 English phrases"; зафиксировать инварианты и валидацию через Symfony Validator.
- **Файлы**: `src/Domain/`, `src/Application/Contract/`, `src/Application/DTO/`, `config/packages/validator.yaml`, `tests/Unit/Domain/`, `docs/adr/adr-002-domain-contracts.md`.
- **Запрещено трогать**: `src/Infrastructure/Telegram/`, `src/Infrastructure/Vision/`.
- **Критерии приёмки**: unit-тесты подтверждают инвариант "ровно 3 фразы" и языковые/стилевые требования; контракты не привязаны к конкретному провайдеру AI.
- **Зависит от**: Подзадача 1

#### Подзадача 3: Security-контур webhook (anti-spoofing + anti-replay)
- **Описание**: Реализовать защиту webhook endpoint: проверка секрета (`X-Telegram-Bot-Api-Secret-Token`/route secret), опциональная IP allowlist Telegram, replay window, идемпотентность по `update_id`, отказ по fail-closed политике.
- **Файлы**: `src/Infrastructure/Security/Webhook/`, `src/Http/Middleware/` или `src/EventSubscriber/`, `config/packages/security.yaml`, `config/services.yaml`, `config/routes/telegram.yaml`, `tests/Integration/Security/WebhookSecurityTest.php`, `docs/security/webhook-threat-model.md`.
- **Запрещено трогать**: `src/Infrastructure/Vision/`, `docs/architecture-plan-1773060179.md`.
- **Критерии приёмки**: запросы без валидного секрета/подписи отвергаются; replay/duplicate update не вызывает повторной бизнес-обработки; в тестах есть позитивный и негативные security-сценарии.
- **Зависит от**: Подзадача 2

#### Подзадача 4: Безопасная загрузка и обработка изображений
- **Описание**: Ввести защищенный image ingestion pipeline: лимиты размера файла и пикселей, MIME sniffing через `finfo`, безопасная перекодировка (например, в JPEG/PNG), удаление метаданных, временное хранилище с TTL, проверка на decompression bomb/pixel flood.
- **Файлы**: `src/Infrastructure/Image/`, `src/Application/Service/ImageSanitizer.php`, `config/packages/framework.yaml`, `config/packages/flysystem.yaml` (если используется), `tests/Integration/Image/ImageSecurityPipelineTest.php`, `docs/security/image-handling-policy.md`.
- **Запрещено трогать**: `src/Infrastructure/Telegram/`, `src/Infrastructure/Vision/` (кроме интерфейсов приема sanitized image).
- **Критерии приёмки**: oversized/invalid image отклоняются контролируемо; downstream получает только sanitized binary; покрыты кейсы MIME spoof и oversized dimensions.
- **Зависит от**: Подзадача 2

#### Подзадача 5: Telegram adapter на Symfony HttpClient
- **Описание**: Реализовать входной/выходной Telegram adapter: прием update, выбор лучшего photo size, получение файла через Telegram API, отправка ответного сообщения; интеграция с security middleware из Подзадачи 3.
- **Файлы**: `src/Infrastructure/Telegram/`, `config/packages/telegram.yaml`, `config/routes/telegram.yaml`, `tests/Integration/Telegram/TelegramWebhookAdapterTest.php`.
- **Запрещено трогать**: `src/Infrastructure/Vision/`, `docs/security/image-handling-policy.md`.
- **Критерии приёмки**: обрабатываются только photo updates; ошибки Telegram API не приводят к 500 без логирования; интеграционные тесты используют моки HTTP-клиента.
- **Зависит от**: Подзадача 3, Подзадача 4

#### Подзадача 6: Vision/LLM adapter на Symfony HttpClient
- **Описание**: Реализовать адаптер к vision-модели (OpenAI или эквивалент) с нормализацией ответа в 3 фразы и graceful fallback; вход только sanitized image из Подзадачи 4.
- **Файлы**: `src/Infrastructure/Vision/`, `src/Application/Prompt/`, `config/packages/ai.yaml`, `tests/Integration/Vision/VisionAdapterTest.php`.
- **Запрещено трогать**: `src/Infrastructure/Telegram/`.
- **Критерии приёмки**: при валидном изображении возвращаются 3 фразы требуемых стилей; timeout/network/provider errors маппятся в контролируемые доменные ошибки.
- **Зависит от**: Подзадача 4

#### Подзадача 7: Сквозная orchestration и идемпотентность
- **Описание**: Собрать use-case и координатор потока `webhook -> security gate -> image sanitize -> vision -> reply`; добавить персистентную идемпотентность и стратегию ретраев (Messenger/queue или sync fallback).
- **Файлы**: `src/Application/UseCase/`, `src/Application/Service/`, `src/Infrastructure/Persistence/Idempotency/`, `config/packages/messenger.yaml`, `tests/Integration/Flow/EndToEndFlowTest.php`.
- **Запрещено трогать**: `docs/architecture-plan-1773060179.md`, `docs/architecture-plan-1773060930.md`.
- **Критерии приёмки**: повторный `update_id` не приводит к повторной отправке; E2E-тест подтверждает корректный happy path и ключевые error paths.
- **Зависит от**: Подзадача 5, Подзадача 6

#### Подзадача 8: Наблюдаемость и security/quality тесты
- **Описание**: Внедрить структурированные логи, correlation id, метрики, security regression-набор для webhook/image, а также контрактные тесты формата ответа.
- **Файлы**: `src/Infrastructure/Observability/`, `config/packages/monolog.yaml`, `tests/Integration/Security/`, `tests/Contract/`, `phpunit.xml.dist`, `docs/testing-strategy.md`.
- **Запрещено трогать**: доменные контракты в `src/Domain/` (кроме add-only расширений через ADR).
- **Критерии приёмки**: покрыты все риски уровня HIGH; есть автоматический прогон security и contract тестов в CI.
- **Зависит от**: Подзадача 7

#### Подзадача 9: Delivery, hardening и runbook
- **Описание**: Подготовить CI/CD, production-конфиг Symfony, чеклист секретов, ротацию токенов Telegram/AI, incident runbook по spoof/replay/image abuse.
- **Файлы**: `Dockerfile`, `docker-compose.yml`, `.github/workflows/ci.yml`, `.github/workflows/deploy.yml`, `docs/runbook.md`, `docs/security-checklist.md`, `docs/ops/webhook-incident-playbook.md`.
- **Запрещено трогать**: `src/Domain/`, `src/Application/Contract/`.
- **Критерии приёмки**: деплой воспроизводим; documented recovery на инциденты webhook spoof/replay и image abuse; секреты не логируются и проходят checklist.
- **Зависит от**: Подзадача 8
