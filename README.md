# Medical Records Management System (Performance Optimized)

REST API сервис с кешированием (Redis) и rate limiting (Token Bucket). Вариант 20, ДЗ №5.

## Функциональность

- Регистрация и аутентификация пользователей (JWT)
- Поиск пользователей по маске имени
- Регистрация пациентов
- Поиск пациентов по ФИО
- Создание медицинских записей
- Получение истории записей пациента
- Кеширование с Redis (Cache-Aside)
- Rate limiting с Token Bucket алгоритмом

## Технологии

- C++20 + **userver** (асинхронный фреймворк Яндекса)
- **MongoDB 7** — документная база данных
- **PostgreSQL 15** — реляционная база данных
- **Redis 7** — кеширование и rate limiting
- JWT аутентификация
- Docker (docker-compose)

## Оптимизации производительности

### Кеширование (Cache-Aside + Redis)

| Endpoint | TTL | Стратегия |
|----------|-----|-----------|
| `GET /api/patients/search?fullName=...` | 5 минут | Cache-Aside с инвалидацией по паттерну |
| `GET /api/medical-records/patient/{id}` | 1 минута | Cache-Aside с инвалидацией по ключу |

**Инвалидация:**
- При `POST /api/patients` — удаление `patients:search:*`
- При `POST /api/medical-records` — удаление `records:patient:{id}`

### Rate Limiting (Token Bucket)

| Endpoint | Лимит | Окно |
|----------|-------|------|
| `POST /api/auth/login` | 5 попыток | 1 минута |
| `POST /api/auth/register` | 3 попытки | 1 минута |

**Заголовки ответа:**
```
X-RateLimit-Limit: 5
X-RateLimit-Remaining: 4
X-RateLimit-Reset: 1621857600
Retry-After: 60
```

## Быстрый старт (Docker)

1. Установите Docker Desktop
2. Склонируйте репозиторий:
```
git clone https://github.com/Chigatu/medical-records-system-perf.git
cd medical-records-system-perf
```
3. Запустите сервисы (API + PostgreSQL + MongoDB + Redis):
```
docker-compose up --build
```
4. Проверьте:
```
curl http://localhost:8080/health
```

## Проверка кеширования

```bash
# Проверить ключи в Redis
docker exec -it medical-redis redis-cli

# Посмотреть статистику кеша
KEYS patients:*
KEYS records:*

# Hit rate
INFO stats | grep keyspace
```

## Проверка rate limiting

```bash
# Отправить 6 запросов логина подряд
for i in {1..6}; do
  curl -X POST http://localhost:8080/api/auth/login \
    -H "Content-Type: application/json" \
    -d '{"login":"test","password":"wrong"}' \
    -i 2>&1 | grep -E "HTTP|X-RateLimit"
done
# 6-й запрос вернет HTTP 429 Too Many Requests
```

## API Endpoints

| Метод | URL | Описание | Требуется JWT | Rate Limit |
|-------|-----|----------|---------------|------------|
| GET | /health | Проверка работоспособности | Нет | Нет |
| POST | /api/auth/register | Регистрация пользователя | Нет | 3/мин |
| POST | /api/auth/login | Вход в систему | Нет | 5/мин |
| POST | /api/patients | Создание пациента | Да | Нет |
| GET | /api/patients/search?fullName={name} | Поиск пациентов (кешируется) | Нет | Нет |
| POST | /api/medical-records | Создание мед.записи | Да | Нет |
| GET | /api/medical-records/patient/{id} | История записей (кешируется) | Да | Нет |

## Структура проекта

```
medical-records-system-perf/
├── src/
│   ├── handlers/               # HTTP обработчики с кешированием и rate limiting
│   ├── models/                 # Доменные модели
│   ├── repositories/           # Репозитории (InMemory, PostgreSQL, MongoDB)
│   ├── services/               # Бизнес-логика + CacheService + RateLimiterService
│   └── main.cpp                # Точка входа
├── configs/                    # Конфигурация userver
├── database/                   # MongoDB + PostgreSQL скрипты
├── performance_design.md       # Документация по оптимизации
├── Dockerfile
├── docker-compose.yaml         # API + PostgreSQL + MongoDB + Redis
└── README.md
```

## Тестирование

```
./test_api_userver.sh
```

## Производительность

| Операция | Без кеша | С кешем | Улучшение |
|----------|----------|---------|-----------|
| Поиск пациентов | 150ms | 15ms | 10x |
| История записей | 100ms | 10ms | 10x |
| Логин (rate limit) | 80ms | 80ms | Защита от брутфорса |
