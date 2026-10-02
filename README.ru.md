# Merezh Platform

Микросервисная платформа для управления пользователями, кошельками, заказами и платежами.

## 📋 Обзор

Merezh - это учебный микросервисный проект, реализованный на **Java 21 + Spring Boot 3**. Платформа состоит из **шести сервисов**, общающихся между собой по **HTTP** (в будущем планируется переход на **Kafka**). Проект демонстрирует:

- разделение на bounded context'ы,
- единую точку входа (Gateway) с JWT-аутентификацией,
- распределённую сагу `order → payment → wallet → payment → order`,
- идемпотентность, пессимистичные блокировки, health checks.

Проект создан как **портфолио** для демонстрации навыков проектирования микросервисов.

## 🏗️ Архитектура

```
                        ┌─────────────────────┐
                        │      Client         │
                        └──────────┬──────────┘
                                   │ HTTP
                                   ▼
                        ┌─────────────────────┐
                        │      Gateway        │  ← единственная точка входа
                        │      :8000          │  ← JWT, роли, маршрутизация
                        └──────────┬──────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
      ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
      │  User Service │    │  Auth Service │    │ Wallet Service│
      │    :8080      │◄───│    :8081      │    │    :8083      │
      └───────┬───────┘    └───────┬───────┘    └───────┬───────┘
              │                    │                    │
              ▼                    ▼                    ▼
      ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
      │   Postgres    │    │   Postgres    │    │   Postgres    │
      │  userservice  │    │  authservice  │    │  walletservice│
      └───────────────┘    └───────────────┘    └───────────────┘

              ┌────────────────────┐
              │                    │
              ▼                    ▼
      ┌───────────────┐    ┌───────────────┐
      │ Order Service │───▶│Payment Service│
      │    :8085      │    │    :8084      │
      └───────┬───────┘    └───────┬───────┘
              │                    │
              ▼                    ▼
      ┌───────────────┐    ┌───────────────┐
      │   Postgres    │    │   Postgres    │
      │  orderservice │    │paymentservice │
      └───────────────┘    └───────────────┘
```

**Все сервисы находятся в Docker-сети `dbnet`.** Наружу проброшен только порт **Gateway** (`127.0.0.1:8000`). Внутренние сервисы недоступны извне.

## 🚀 Технологический стек

**Backend**

- Java 21 - основной язык
- Spring Boot 3 - фреймворк приложения
- Spring Data JPA - доступ к БД и ORM
- Spring Security - аутентификация и авторизация (в Gateway)
- Spring Security Crypto - BCrypt
- JJWT - JWT-токены
- RestTemplate - синхронные HTTP-вызовы
- Jakarta Validation - валидация DTO
- Lombok - уменьшение boilerplate

**Базы данных**

- PostgreSQL - по одной БД на каждый сервис (database-per-service)

**DevOps**

- Docker - контейнеризация
- Docker Compose - оркестрация
- Spring Boot Actuator - health checks (liveness/readiness)

**Тестирование**

- JUnit 5
- Mockito

## 🧩 Сервисы

| Cервис             | Порт | Назначение                                          | README                               |
|---------------------|------|--------------------------------------------------|--------------------------------------|
| **Gateway**         | 8000 | Single entry point, JWT, routing                 | [README](https://github.com/CkutlsGit/merezh-gateway)  |
| **User Service**    | 8080 | User storage and management                      | [README](https://github.com/CkutlsGit/merezh-userservice)     |
| **Auth Service**    | 8081 | Authentication, JWT, refresh tokens              | [README](https://github.com/CkutlsGit/merezh-authservice)     |
| **Wallet Service**  | 8083 | Wallets and balances                             | [README](https://github.com/CkutlsGit/merezh-walletservice)   |
| **Order Service**   | 8085 | Orders and order items                           | [README](https://github.com/CkutlsGit/merezh-orderservice)    |
| **Payment Service** | 8084 | Payment processing                               | [README](https://github.com/CkutlsGit/merezh-paymentservice)  |

## 🛠️ Быстрый старт

### Требования

- Docker
- Docker Compose

### Запуск
Скопируйте репозитории всех сервисов и выполните команду `docker compose up --build` в директории шлюза (gateway).

## 📚 API

Единая точка входа - **Gateway** (`http://localhost:8000`). Все запросы проходят через него.

### Аутентификация

| Method | Endpoint                | Описание                          | Доступ        |
|--------|-------------------------|-----------------------------------|---------------|
| POST   | `/api/v1/auth/register` | Регистрация                       | Public        |
| POST   | `/api/v1/auth/login`    | Вход, получение пары токенов      | Public        |
| POST   | `/api/v1/auth/refresh`  | Обновление токенов по refresh     | Public        |
| POST   | `/api/v1/auth/logout`   | Выход                             | Authenticated |

### Пользователи

| Method | Endpoint              | Описание                    | Доступ |
|--------|-----------------------|-----------------------------|--------|
| GET    | `/api/v1/users`       | Получить всех пользователей | ADMIN  |
| GET    | `/api/v1/users/{id}`  | Получить пользователя по ID | ADMIN  |
| DELETE | `/api/v1/users/{id}`  | Удалить пользователя        | ADMIN  |

### Кошельки

| Method | Endpoint                   | Описание             | Доступ        |
|--------|----------------------------|----------------------|---------------|
| POST   | `/api/v1/wallets/create`   | Создать кошелёк      | Authenticated |
| GET    | `/api/v1/wallets/balance`  | Получить баланс      | Authenticated |
| POST   | `/api/v1/wallets/balance/sum` | Пополнить баланс  | Authenticated |
| POST   | `/api/v1/wallets/balance/sub` | Списать средства  | Authenticated |
| DELETE | `/api/v1/wallets/delete/{id}` | Удалить кошелёк   | ADMIN         |

### Заказы

| Method | Endpoint                 | Описание                       | Доступ        |
|--------|--------------------------|--------------------------------|---------------|
| GET    | `/api/v1/orders/get/{id}`| Получить заказ по ID           | Authenticated |
| GET    | `/api/v1/orders/get/user`| Получить заказы пользователя   | Authenticated |
| POST   | `/api/v1/orders/create`  | Создать заказ                  | Authenticated |

### Платежи

| Method | Endpoint                       | Описание                | Доступ        |
|--------|--------------------------------|-------------------------|---------------|
| GET    | `/api/v1/payments/get/{id}`    | Получить платёж по ID   | Authenticated |
| GET    | `/api/v1/payments/get/user`    | Получить платежи юзера  | Authenticated |
| POST   | `/api/v1/payments/pay/{orderId}` | Оплатить заказ        | Authenticated |

**Внутренние эндпоинты** (`/users/create`, `/users/validate`, `/orders/update`, `/payments/place`) **не публикуются** через Gateway - только для сервис-сервис взаимодействия.

## 🔄 Сага: жизненный цикл заказа

Платформа реализует **распределённую сагу** (хореография) при создании и оплате заказа:

```
┌──────────┐   1. POST /create        ┌──────────────┐
│  Client  │─────────────────────────▶│ Order Service│
└──────────┘                          └──────┬───────┘
                                             │
                                             │ 2. save Order(PAYMENT_WAITING)
                                             │
                                             │ 3. POST /place
                                             ▼
                                      ┌──────────────┐
                                      │Payment Service│
                                      └──────┬───────┘
                                             │
                                             │ 4. save Payment(WAITING)
                                             │
                                             │ 5. return WAITING
                                             ▼
                                      ┌──────────────┐
                                      │ Order Service│
                                      └──────────────┘

  === Позже пользователь оплачивает ===

┌──────────┐   6. POST /pay/{orderId}  ┌──────────────┐
│  Client  │─────────────────────────▶│Payment Service│
└──────────┘                          └──────┬───────┘
                                             │
                                             │ 7. POST /balance/sub
                                             ▼
                                      ┌──────────────┐
                                      │Wallet Service│
                                      └──────┬───────┘
                                             │
                                             │ 8. списание
                                             │
                                             │ 9. return balance
                                             ▼
                                      ┌──────────────┐
                                      │Payment Service│
                                      └──────┬───────┘
                                             │
                                             │ 10. Payment = SUCCESS
                                             │
                                             │ 11. POST /update (finally)
                                             ▼
                                      ┌──────────────┐
                                      │ Order Service│
                                      └──────┬───────┘
                                             │
                                             │ 12. Order = PAYMENT_SUCCESS
                                             ▼
```

**Особенности реализации:**

- `@Transactional(noRollbackFor = {HttpClientErrorException, HttpServerErrorException})` в Payment - статус сохраняется даже при сетевой ошибке.
- Callback в Order Service выполняется **всегда** через `finally`.
- При 4xx от Wallet - платёж помечается `FAILED`, при 5xx - `WAITING`.
- При повторном `/place` для существующего платежа вызывается `existsOrder` (повторная попытка списания, если `WAITING`).

## 🔒 Безопасность

- **Gateway - единственная точка входа.** Внутренние сервисы работают только в Docker-сети `dbnet`.
- **JWT-аутентификация.** Access (30 мин) + Refresh (7 дней). Подпись HS256.
- **Ротация refresh-токенов.** При каждом `/refresh` старый хэш заменяется новым.
- **Refresh-токены хранятся как SHA-256 хэши** - в случае утечки БД сырые токены не скомпрометированы.
- **Пароли хранятся как BCrypt-хэши.** Хэширование в Auth Service.
- **Подмена `X-User-Id` невозможна** - клиентский заголовок вырезается в Gateway.
- **Ролевая авторизация** - `hasRole("ADMIN")` для admin-эндпоинтов.
- **Внутренние эндпоинты не проксируются** - доступны только внутри `dbnet`.

## ⚙️ Конфигурация

Все секреты (пароли БД, JWT-секрет) вынесены в `docker-compose.yml` (для учебного проекта - в открытом виде).
