# queue-flow: структура и задачи

## Структура репозитория (NestJS monorepo)

```
queue-flow/
├── apps/
│   ├── api/                      # HTTP + WebSocket
│   │   └── src/
│   │       ├── main.ts           # bootstrap, Swagger, ValidationPipe
│   │       ├── app.module.ts
│   │       ├── auth/             # JWT, guard, login оператора
│   │       ├── organizations/
│   │       ├── services/
│   │       ├── windows/          # окна, call-next
│   │       ├── tickets/          # контроллер, сервис, DTO, entity
│   │       ├── queue/            # логика очереди, позиция, кэш в Redis
│   │       ├── realtime/         # gateway, redis-adapter, rooms
│   │       └── health/
│   └── worker/                   # потребитель RabbitMQ
│       └── src/
│           ├── main.ts
│           ├── worker.module.ts
│           ├── notifications/    # обработчик "ticket.called" и др.
│           ├── timeouts/         # авто-пропуск талона
│           └── stats/
├── libs/
│   ├── database/                 # entities, миграции, data-source.ts
│   ├── messaging/                # топология RabbitMQ: exchange, queues, DLX, TTL
│   └── common/                   # константы, типы событий, фильтры
├── test/                         # e2e (REST + WebSocket)
├── public/board.html             # табло на socket.io-client
├── docker-compose.yml
├── Dockerfile
├── .env.example
├── .github/workflows/ci.yml
└── README.md
```

## Топология RabbitMQ

- exchange `queue.events` (topic)
- очередь `notifications` (ключ `ticket.called`, `ticket.skipped`) с retry
- очередь `ticket.timeout.wait` с TTL = TICKET_CALL_TIMEOUT_SEC и DLX -> `ticket.timeout.process`
- очередь `*.dlq` для сообщений, которые не обработались после N попыток
- идемпотентность: id события + ключ в Redis (SET NX с TTL) перед обработкой

## Задачи для GitHub Issues

### Milestone 1. Каркас
- [ ] Инициализировать NestJS monorepo (api, worker, libs)
- [ ] ESLint, Prettier, husky + lint-staged
- [ ] @nestjs/config с валидацией env (Joi или zod)
- [ ] docker-compose: postgres, redis, rabbitmq с healthcheck
- [ ] TypeORM: data-source, настройка миграций, скрипты migration:generate/run
- [ ] Сущности: Organization, Service, Window, Ticket, TicketEvent
- [ ] Первая миграция

### Milestone 2. REST API
- [ ] CRUD организаций и услуг (DTO + class-validator)
- [ ] CRUD окон, привязка к услугам
- [ ] POST /tickets: выдача талона, номер вида A-017, позиция в очереди
- [ ] GET /tickets/:id: статус и позиция
- [ ] POST /tickets/:id/cancel
- [ ] POST /windows/:id/call-next (транзакция, SELECT ... FOR UPDATE SKIP LOCKED)
- [ ] POST /tickets/:id/start и /done
- [ ] JWT-авторизация оператора, guard и роли
- [ ] Глобальный exception filter, единый формат ошибок
- [ ] Swagger: описания, примеры, bearer auth
- [ ] GET /health (проверка Postgres, Redis, RabbitMQ)

### Milestone 3. WebSocket
- [ ] Gateway на socket.io: события subscribe:service, subscribe:ticket
- [ ] Комнаты service:{id} и ticket:{id}
- [ ] Redis-адаптер (@socket.io/redis-adapter), проверка на 2 инстансах api
- [ ] При подключении отправлять актуальное состояние (reconnect)
- [ ] Страница public/board.html (табло)
- [ ] Аутентификация сокетов оператора (токен в handshake)

### Milestone 4. RabbitMQ и worker
- [ ] Библиотека messaging: объявление exchange, очередей, DLX
- [ ] Публикация событий ticket.created / called / skipped / done из api
- [ ] Worker: потребитель notifications (заглушка: лог или Telegram-бот)
- [ ] Retry с лимитом попыток и уходом в DLQ
- [ ] Идемпотентность через Redis (SET NX)
- [ ] Авто-пропуск талона по таймауту (TTL + DLX)
- [ ] Graceful shutdown воркера (ack/nack, закрытие соединений)

### Milestone 5. Redis
- [ ] Кэш состояния очереди и позиции, инвалидация при смене статуса
- [ ] Rate limit на POST /tickets (@nestjs/throttler + Redis storage)

### Milestone 6. Тесты
- [ ] Unit: логика очереди (порядок, пропуск, отмена, конкурентный call-next)
- [ ] Unit: идемпотентность и retry в воркере
- [ ] E2E REST (supertest + тестовая БД через testcontainers или compose)
- [ ] E2E WebSocket (socket.io-client): талон вызван -> событие пришло
- [ ] Покрытие и бейдж в README

### Milestone 7. Оформление
- [ ] GitHub Actions: lint, test, build, сервисы postgres/redis/rabbitmq
- [ ] README: описание, GIF, схема Mermaid, запуск одной командой
- [ ] Раздел "Архитектурные решения" в README
- [ ] Скрипт seed: демо-организация, услуги, окна, оператор
- [ ] (опц.) Деплой на VPS и ссылка на демо

## Порядок и хронология

Идите по milestones сверху вниз. Один issue = одна ветка = один небольшой PR
в собственный репозиторий: история выглядит живой и показывает рабочий процесс.

## Типовые ловушки

- Номер талона: считайте счётчик в БД (последовательность или строка с блокировкой), не в памяти.
- call-next: без SKIP LOCKED два оператора могут вызвать один и тот же талон.
- Publisher confirms в RabbitMQ: без них события могут теряться молча.
- Событие публиковать после коммита транзакции, иначе воркер увидит старое состояние.
- Socket.io за балансировщиком: нужен Redis-адаптер и sticky sessions либо transports: ['websocket'].
