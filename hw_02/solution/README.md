# ДЗ 2. Проектирование взаимодействия сервисов

**Система:** Сервис бронирования (аналог Booking.com)

---

## 1. Декомпозиция сервисов

| Сервис             | Ответственность                                                                                                                                                                                             |
|--------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **API Gateway**    | Единая точка входа. Роутинг запросов, балансировка нагрузки, rate limiting, терминация TLS, делегирование аутентификации в Auth.                                                                            |
| **Account**        | Данные пользователей-гостей: ФИО, email, профиль. Регистрация и управление аккаунтом гостя.                                                                                                                 |
| **Owner**          | Данные владельцев имущества, регистрация и верификация владельцев, управление их объектами.                                                                                                                 |
| **Inventory**      | Хранение и выдача информации о сдаваемом имуществе (апартаменты, квартиры, виллы, номера). Базовый набор параметров плюс специфичные атрибуты по типу объекта. Данные разнородные, поэтому NoSQL (MongoDB). |
| **Inventory Unit** | Бронируемая единица: `inventoryId` + конкретная опция (тип номера, конкретная квартира) с календарём доступности. Управление доступностью и холдами на конкретную единицу.                                  |
| **Search**         | Поиск инвентаря по параметрам (даты, локация, фильтры, цена). Оптимизирован под чтение, read-model на Elasticsearch.                                                                                        |
| **Booking**        | Оркестрация процесса бронирования: поиск, выбор варианта, ввод данных, summary, инициирование оплаты. Стейт-машина брони и история бронирований.                                                            |
| **Payment**        | Проведение оплаты через разных провайдеров, создание ордеров, отслеживание статуса платежа, возвраты.                                                                                                       |
| **Notification**   | Отправка уведомлений о статусе брони и прочей информации разными каналами (push, email, SMS). Слушает события ключевых сервисов.                                                                            |

---

## 2. Протоколы взаимодействия

| Взаимодействие                           | Протокол         | Обоснование                                                                                      |
|------------------------------------------|------------------|--------------------------------------------------------------------------------------------------|
| Client → Gateway                         | REST / WebSocket | REST для основных операций, WS для real-time статусов брони.                                     |
| Gateway → Account / Owner / Auth         | REST             | Простые CRUD-операции, синхронные, человекочитаемые контракты.                                   |
| Gateway → Search                         | GraphQL          | Поисковые запросы с большим числом опциональных фильтров; клиент запрашивает только нужные поля. |
| Booking → Inventory Unit / Payment       | gRPC             | Внутренние синхронные вызовы, критичные к латентности; строгие контракты, бинарный протокол.     |
| Search → Booking                         | gRPC             | Внутренний синхронный вызов.                                                                     |
| Между сервисами (изменения, нотификации) | Events (MQ)      | Асинхронная согласованность, слабая связанность, буферизация пиков.                              |
| Payment / Booking → External Providers   | REST (external)  | Интеграция с внешними платёжными провайдерами и поставщиками инвентаря.                          |

---

## 3. Схема взаимодействия

См. приложенный файл `booking-architecture.png` (схема с указанием протоколов на каждом ребре).

@startuml booking-architecture
title Сервис бронирования — взаимодействие сервисов

skinparam componentStyle rectangle
skinparam rectangle {
BackgroundColor White
BorderColor Black
}
skinparam ArrowColor Black
skinparam defaultTextAlignment center

actor "Client\n(Web/Mobile)" as Client

rectangle "API Gateway" as Gateway
rectangle "Auth" as Auth
rectangle "Account" as Account
rectangle "Owner" as Owner
rectangle "Inventory" as Inventory
rectangle "Inventory Unit" as Unit
rectangle "Search" as Search
rectangle "Booking" as Booking
rectangle "Payment" as Payment
rectangle "Notification" as Notification
queue "MQ" as MQ
cloud "External\nProviders" as Ext

' --- Client / Gateway ---
Client --> Gateway : REST / WS

' --- Gateway -> синхронные сервисы ---
Gateway --> Auth : REST
Gateway --> Account : REST
Gateway --> Owner : REST
Gateway --> Inventory : REST
Gateway --> Booking : REST
Gateway --> Search : GraphQL

' --- Синхронные внутренние (gRPC) ---
Search --> Booking : gRPC
Booking --> Payment : gRPC
Booking --> Unit : gRPC
Inventory --> Unit : gRPC

' --- События в брокер ---
Account ..> MQ : Events
Owner ..> MQ : Events
Inventory ..> MQ : Events
Booking ..> MQ : Events
Payment ..> MQ : Events

' --- Синхронизация read-model ---
MQ ..> Search : Events

' --- Нотификации ---
MQ ..> Notification : Events
Notification ..> Client : Push / Email / SMS

' --- Внешние интеграции ---
Payment ..> Ext : REST (external)
Booking ..> Ext : REST (external)

@enduml

---

## 4. API-контракты

### 4.1 Поиск инвентаря (GraphQL)


Спецификация openAPI `booking-api.openapi.yaml`

Овервью в свободной форме:
Выбран GraphQL: запросы длинные, с большим числом опциональных фильтров, клиенту нужен разный набор полей в выдаче.

```graphql
query SearchInventory {
  search(
    dates: { checkIn: "2026-08-01", checkOut: "2026-08-07" }
    price: { min: 50, max: 200, currency: "EUR" }
    city: "Kotor"
    country: "ME"
    filter: {
      type: APARTMENT
      persons: 2
      rooms: 1
      rating: 8.0
      services: [CONDITIONER, REFRIGERATOR, MINIBAR]
      distanceToCenter: 2000
    }
  ) {
    id
    name
    city
    country
    type
    description
    options { name price }
    facilities
    services
  }
}
```

### 4.2 Создание брони (REST)

```
POST /booking
```

Request:

```json
{
  "inventoryId": "inv_123",
  "subInventoryId": "unit_456",
  "accountId": "acc_789",
  "dates": {
    "checkIn": "2026-08-01",
    "checkOut": "2026-08-07"
  },
  "price": 840,
  "currency": "EUR",
  "guests": 2,
  "additionalInfo": {
    "lateCheckIn": true
  }
}
```

Response (требуется оплата):

```json
{
  "bookingId": "bkg_001",
  "status": "PENDING_PAYMENT",
  "paymentInfo": {
    "providerLink": "https://pay.provider/...",
    "metaInfo": {}
  }
}
```

Response (summary, если оплата позже или не нужна):

```json
{
  "bookingId": "bkg_001",
  "status": "CONFIRMED",
  "inventoryDetails": {
    "name": "...",
    "address": "..."
  },
  "userName": "Roman F.",
  "price": 840,
  "dates": {
    "checkIn": "2026-08-01",
    "checkOut": "2026-08-07"
  }
}
```

### 4.3 Добавление имущества (REST)

```
POST /inventory
```

Request:

```json
{
  "ownerId": "own_111",
  "type": "APARTMENT",
  "price": 120,
  "currency": "EUR",
  "availableDates": [
    {
      "from": "2026-08-01",
      "to": "2026-09-30"
    }
  ],
  "services": [
    "CONDITIONER",
    "WIFI",
    "KITCHEN"
  ],
  "facilities": [
    "PARKING",
    "POOL"
  ],
  "description": "..."
}
```

Response:

```json
{
  "inventoryId": "inv_123"
}
```

---

## 5. Асинхронность

Асинхронное взаимодействие через брокер сообщений применяется там, где не требуется немедленный синхронный ответ и важна слабая связанность:

1. **Индексирование в Elasticsearch.** При записи/изменении инвентаря событие уходит в MQ, Search-сервис консьюмит и обновляет read-model. Индексация
   идёт параллельно основной логике записи.
2. **Отправка нотификаций.** Notification слушает события ключевых сервисов и отправляет уведомления нужным каналом.
3. **Ответ владельца на бронь.** При подтверждении/отклонении владельцем гостю асинхронно уходит уведомление.
4. **Начисление бонусов.** После перехода брони в статус `DONE` событие триггерит расчёт и начисление бонусов.
5. **Рекомендации.** По завершении поездки формируются рекомендации мест для пользователя.

---

## 6. Применённые паттерны

**CQRS.** Основной паттерн. Запись (добавление и изменение имущества владельцами) и чтение (поиск подходящих вариантов) разделены. Запись идёт через
Inventory в MongoDB, чтение из Elasticsearch, который синхронизируется и индексируется параллельно записи через события. Модели записи и чтения
оптимизированы под свои задачи независимо.

**Saga.** Применяется на этапе оплаты брони для обеспечения транзакционности распределённого процесса (резерв единицы, списание оплаты, подтверждение
брони). Оркестрационный вариант: координатор(Booking service) управляет шагами и компенсациями при откате.

**API Gateway.** Реализован отдельным сервисом. Отвечает за роутинг, делегирование аутентификации, фильтрацию, rate limiting и прочую обработку
входящих запросов.

**Transactional Outbox.** Запись в таблицу outbox (событие для нотификации) и изменение статуса в таблице booking выполняются в одной локальной
транзакции. Это гарантирует согласованность между сменой статуса и фактом отправки уведомления.

**Distributed Lock / оптимистичная блокировка.** Для разрешения конкуренции, когда несколько пользователей хотят забронировать одну единицу. Варианты:
optimistic lock (версионирование в БД) или распределённый лок (Redis).

---